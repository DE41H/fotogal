# fotogal

fotogal is a self-hosted photo gallery you run on your own hardware. Point it at a Tailscale tailnet or your own domain, then browse, search, and upload photos from the Flutter mobile app.

## Features

- Self-hosted: the whole stack runs with one Docker Compose file.
- Photo upload and browsing from the Flutter app.
- EXIF metadata and location data stored with PostGIS.
- Semantic search over image content using int8 SigLIP 2 embeddings stored in pgvector.
- Face detection and grouping using InsightFace `buffalo_l`, with int8 models.
- Caddy reverse proxy with TLS and HTTP/3 (QUIC).
- Reachable over Tailscale or a public domain.

## Architecture

| Service | Role |
| --- | --- |
| Caddy | Reverse proxy, TLS, HTTP/3 |
| Django REST Framework | HTTP API |
| Celery | Background workers |
| PostgreSQL + PostGIS + pgvector | Metadata, EXIF and geo data, embeddings |
| RabbitMQ | Task broker |
| Valkey | Cache |
| SeaweedFS | Blob storage for original images and thumbnails |
| ONNX Runtime | SigLIP 2 embedding inference (int8) |
| InsightFace | Face detection, embeddings, and grouping |
| Flutter | Mobile client |

When a photo is uploaded:

1. The API stores the original in SeaweedFS and a row in PostgreSQL.
2. A Celery worker extracts EXIF data and generates thumbnails.
3. The worker produces a SigLIP 2 embedding, stores it in pgvector, and extracts faces.
4. Faces are clustered into people, and search combines vector similarity, face matches, EXIF fields, and tags.

## Tech stack

| Layer | Choice | Why |
| --- | --- | --- |
| Reverse proxy | Caddy | Automatic TLS, simple config, QUIC support |
| Backend | Django REST Framework | Fast prototyping of a typed API |
| Task queue | Celery + RabbitMQ | Reliable background processing for imports and ML |
| Database | PostgreSQL + PostGIS + pgvector | One database for metadata, geo queries, and vectors |
| Cache | Valkey | Redis-compatible, fast, open source |
| Object storage | SeaweedFS | Self-hosted blob store that scales with the library |
| Embeddings | ONNX Runtime + SigLIP 2 base (int8) | Strong multilingual search at a CPU-friendly size |
| Faces | InsightFace `buffalo_l` (int8) | Good accuracy without a hosted service |
| Mobile app | Flutter | Native rendering performance on iOS and Android |
| Deployment | Docker Compose | Single-file, reproducible self-hosting |

## Repository layout

- `backend/` — Django project (`cfg/`) and the `foto` app.
- `app/` — Flutter mobile app (planned).
- `docker/` — Docker Compose stack and service configuration (planned).
- `onnx/` — local model cache mounted read-only by the API and workers (weights are gitignored).
- `README.md` — this file.
- `AGENTS.md` — working notes for AI coding agents.
- `LICENSE.md` — GNU GPL v3.0.

## Getting started

### Backend (available today)

Requires Python 3.13 and [uv](https://docs.astral.sh/uv/).

```
cd backend
uv sync
uv run python manage.py migrate
uv run python manage.py runserver
```

The API is served at `http://127.0.0.1:8000/`. Create an admin user with `uv run python manage.py createsuperuser` and sign in at `/admin/`.

The backend currently uses SQLite for local development. PostgreSQL with PostGIS and pgvector will replace it as part of the Compose stack.

### Full stack (planned)

The Compose stack is not implemented yet. Once `docker/` is populated, the intended flow will be:

```
cp .env.example .env
docker compose -f docker/docker-compose.yml up -d
```

Do not expect these commands to work against the current repository.

## Configuration

Configuration will be supplied through a single `.env` file consumed by Compose. Planned variables:

| Variable | Purpose |
| --- | --- |
| `FOTOGAL_DOMAIN` | Public domain or Tailscale hostname |
| `SECRET_KEY` | Django secret key |
| `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD` | Database connection |
| `RABBITMQ_URL` | Task broker URL |
| `VALKEY_URL` | Cache URL |
| `SEAWEEDFS_ENDPOINT` | Blob storage endpoint |
| `SIGLIP_MODEL_DIR` | SigLIP 2 ONNX model directory |
| `SIGLIP_REVISION` | Pinned SigLIP 2 model revision |
| `SIGLIP_PRECISION` | Model precision, always `int8` |
| `INSIGHTFACE_MODEL_DIR` | InsightFace model directory |
| `MAX_UPLOAD_SIZE` | Upload size limit |

Never commit real secrets, model weights, or uploaded media.

### Models (planned)

Models live in `onnx/` on the host and are bind-mounted read-only into the API and workers. Weights are gitignored; only the fetch script and manifest are tracked.

- SigLIP 2 base patch16-384, int8: `vision_model_int8.onnx` and `text_model_int8.onnx`, producing 768-dimensional embeddings.
- InsightFace `buffalo_l`, int8: detection, landmarks, and 512-dimensional face embeddings.

The vision and text towers must use the same precision. Changing the model, revision, or precision requires re-embedding every asset.

## Reverse proxy

Caddy terminates TLS and proxies the API and app traffic to the backend.

- Domain mode: point an A/AAAA record at the host and let Caddy obtain certificates automatically.
- Tailscale mode: use the machine's tailnet hostname and keep the gallery private to the tailnet.
- HTTP/3 requires UDP port 443 in addition to TCP 443. Caddy falls back to HTTP/2 when QUIC is blocked.

## Project status

Early scaffolding. The Django backend skeleton exists; the Flutter app, Compose stack, and ML pipeline are still to be built.

## License

fotogal is licensed under the GNU General Public License v3.0. See [LICENSE.md](LICENSE.md).
