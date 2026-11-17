# CampusHub – 9-service Docker stack

The application code in `app/` is untouched. All Docker work lives in:

| File | Purpose |
|---|---|
| `app/frontend/Dockerfile` | multi-stage: Node builds React → unprivileged Nginx serves `dist/` |
| `app/api/Dockerfile` | one image used by both **api** and **worker** |
| `*/.dockerignore`, `.gitignore` | keep `node_modules`, `.env`, `.git` out of images / git |
| `docker-compose.yml` | the 9-service stack for one machine |
| `docker-stack.yml` | Swarm-style stack file (registry images, replicas, configs) |
| `.env.example` | placeholder variables (copy to `.env`) |

## How to run

```bash
cp .env.example .env          # then edit the passwords
docker compose up -d --build --wait
docker compose ps             # all 9 should be running / healthy
```

Open **http://localhost:8080** (app), `http://localhost:8080/api/health` (health).
Admin UIs (this machine only): MailHog `http://127.0.0.1:8025`,
RabbitMQ `http://127.0.0.1:15672`, MinIO console `http://127.0.0.1:9001`.

Run the checks: `DC="docker compose" ENV_FILE=.env ./grade.sh`
Stop: `docker compose down` (data kept) or `docker compose down -v` (data deleted).

## Architecture

```
 browser ──▶ gateway (nginx :8080)  ← only published port
              ├─ /          ▶ frontend (static React)
              ├─ /api/      ▶ api ──▶ db (Postgres), redis (cache), rabbitmq (queue), minio (files)
              └─ /storage/  ▶ minio:9000/campushub/
 worker ◀── rabbitmq ;  worker ──▶ mailhog (SMTP)

 networks:  frontend [gateway, frontend]
            backend  [gateway, api, worker, minio, mailhog]
            data     [api, worker, db, redis, rabbitmq]
```

## Three design decisions

1. **Three networks (frontend / backend / data).** The database, cache and queue sit on
   `data`, which only the api and worker join, so a compromised gateway or frontend
   cannot talk to them directly. Least-privilege networking.
2. **Health checks + `depends_on: condition: service_healthy`.** The API runs a DB
   migration at start and needs Postgres, Redis, RabbitMQ and MinIO ready. Waiting for
   *healthy* (not just *started*) removes random start-up crashes.
3. **One image for api and worker.** Same code, different command. One build, one
   vulnerability scan, no version drift. The worker disables the HTTP health check
   because it has no web server.

Also: named volumes (`pgdata`, `miniodata`) keep data across `down`/`up`; every service
has `restart: unless-stopped` and CPU/memory limits; only the gateway is public, the
admin UIs are bound to `127.0.0.1`; secrets only come from `.env` (never committed).

## Pushing images to a registry

```bash
docker login
docker tag campushub-api:1.0      <user>/campushub-api:1.0
docker tag campushub-frontend:1.0 <user>/campushub-frontend:1.0
docker push <user>/campushub-api:1.0
docker push <user>/campushub-frontend:1.0
```
Anyone can then run `docker pull <user>/campushub-api:1.0`.
