# Operations

## Docker compose (recommended for local)

```bash
docker compose up --build
```

Services:
- `postgres` (db)
- `redis` (queue)
- `api` (FastAPI)
- `worker` (RQ)

### Migrations
The backend Docker image runs migrations on startup by default:
- `RUN_MIGRATIONS=1` → runs `alembic upgrade head`
- `AUTO_CREATE_DB=0` → disables SQLAlchemy `create_all()`

In `docker-compose.yml` this is already configured:
- api: RUN_MIGRATIONS=1, AUTO_CREATE_DB=0
- worker: RUN_MIGRATIONS=0, AUTO_CREATE_DB=0

## Worker

The worker runs `python worker.py` and processes jobs from RQ queues.

Env:
- `REDIS_URL` (default: redis://localhost:6379/0)
- `RQ_QUEUES` (default: `default`)

## Manual migrations

```bash
alembic upgrade head
alembic revision --autogenerate -m "add_field"
```
