# AGENTS.md

## What this is

fotogal is a self-hosted, Immich-like photo gallery. Django REST Framework serves the API, a Flutter app is the client, and Docker Compose runs the infrastructure. Humans should read `README.md`.

## Current state

- `backend/` is a Django 6.1.2 + DRF skeleton using SQLite. No Celery, Postgres, or ML code exists yet.
- `app/` (Flutter) and `docker/` (Compose, Caddy) are empty placeholders.
- The architecture below is the target. Do not assume a component is implemented because it is listed.

## Repository map

- `backend/cfg/` — Django settings, URLs, WSGI/ASGI.
- `backend/foto/` — main Django app; put new models, serializers, views, and tasks here unless a new app is clearly justified.
- `app/` — Flutter mobile client.
- `docker/` — Docker Compose stack and Caddy config.
- `README.md` — human documentation. `LICENSE.md` — GPL-3.0.

## Fixed stack

Do not substitute these choices.

- Reverse proxy: Caddy (TLS, HTTP/3).
- API: Django REST Framework.
- Background jobs: Celery with RabbitMQ as the broker.
- Database: PostgreSQL with PostGIS and pgvector extensions. This is the only database.
- Cache: Valkey. Nothing else.
- Blob storage: SeaweedFS for originals and thumbnails.
- Embeddings: SigLIP via ONNX Runtime.
- Faces: InsightFace with the `buffalo_l` model pack.
- Captions: Moondream.
- Mobile app: Flutter.
- Orchestration: Docker Compose.

## Commands

Backend work happens in `backend/` with uv and Python 3.13:

```
uv sync
uv run python manage.py makemigrations
uv run python manage.py migrate
uv run python manage.py test
uv run python manage.py runserver
```

Compose and Flutter commands do not exist yet; add them here only once those directories contain real configuration.

## Rules

- Run all ML inference, thumbnail generation, and image processing in Celery workers, never in an HTTP request.
- Store image bytes only in SeaweedFS. PostgreSQL holds metadata, EXIF, and vectors, never blobs.
- Use Valkey only for cache and RabbitMQ only for task delivery.
- Never switch settings or tests back to SQLite for production paths; PostgreSQL is the source of truth.
- The embedding model and its output dimension are coupled. Changing the SigLIP model requires a re-embedding migration.
- Route configuration through environment variables. Never commit secrets, model weights, or media.
- Every model change includes a migration.
- Keep new dependencies in `backend/pyproject.toml` via `uv add`, not hand-edited lockfiles.
- Match existing Django layout: settings in `cfg/`, feature code in apps, DRF serializers and viewsets rather than ad-hoc JSON handling.

## Gotchas

- Creating the `postgis` and `vector` extensions requires database privileges; enable them in a migration or init script.
- pgvector needs an ANN index (HNSW or IVFFlat) and the distance operator must match the embedding normalization.
- Normalize SigLIP embeddings the same way at index time and query time.
- `buffalo_l` and Moondream are large downloads with their own licenses; never vendor them into the repo.
- Caddy HTTP/3 needs UDP 443 open. Do not assume QUIC works everywhere.
- Tailscale and public-domain deployments differ only in DNS and hostname configuration; avoid forking the stack per deployment.

## Pointers

- Feature and deployment documentation: `README.md`.
- License: GNU GPL v3.0 in `LICENSE.md`.
