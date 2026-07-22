# Deployment Guide

How to deploy RCAgent for real use, end to end. This is not the one-shot
local demo in the [main README](../README.md#setup) — it is the actual
topology the project runs in: one server hosting the backend, frontend,
agents and shared infrastructure, plus one or more **separate** machines
each running an observer.

## Topology

```
Server (e.g. 203.0.113.10)                  Observer host(s)
┌──────────────────────────────────┐        ┌─────────────────────┐
│ docker/services/                 │        │ linux_binary         │
│  ├── postgres      (rcagent db)  │        │  (observer binary)   │
│  ├── postgres-ml   (incident_ml) │        └──────────┬───────────┘
│  ├── kafka         (jobs/results)│                   │
│  ├── rabbitmq      (AI escalation)│                  │ HTTPS (register/refresh)
│  └── monitoring    (Grafana stack, optional)          │ Kafka (external listener, 19092)
│                                   │◄──────────────────┘
│ backend/  (app)                  │
│ frontend/ (nginx + SPA)          │
│ agents/   (news-worker, ml3-consumer, e2e)            │
└──────────────────────────────────┘
```

The observer only ever talks to the server over its **public** endpoints
(backend HTTPS, Kafka's external listener). It never shares a Docker
network with anything on the server — the two machines are never assumed
to be on the same LAN.

Everything under `docker/services/*` runs on the **server**, alongside
`backend`, `frontend` and `agents`. Nothing here needs to be duplicated on
an observer host.

## Prerequisites

- Docker with the Compose plugin, on the server and on every observer host.
- Go 1.23+ only if you build `linux_binary` natively instead of with Docker.
- A `.env` next to every `docker-compose.yml`/`.yaml` listed below. Each one
  has an `.env.example` beside it — copy it and fill in real values:
  ```bash
  cp .env.example .env
  ```

## 1. Create the shared Docker networks

Every `docker/services/*` project, plus `backend`, `frontend` and `agents`,
join one or more `external: true` networks so they can reach each other
without any project depending on another project's compose file. Create
them once, before bringing anything up:

```bash
docker network create backend-db-net       # backend <-> docker/services/postgres
docker network create backend-kafka-net    # backend  <-> docker/services/kafka
docker network create backend-rabbitmq-net # backend + agents <-> docker/services/rabbitmq
docker network create agents-db-net        # backend + agents <-> docker/services/postgres-ml
docker network create rcagent-net          # frontend <-> backend (same-origin API proxy)
```

`docker/services/monitoring` is self-contained (Traefik + Mimir + Loki +
Grafana) and creates its own bridge networks — no manual network needed for
it.

> **Why so many networks instead of one shared one?** Each infra service is
> deployable/restartable independently, and no service's compose file
> references another project's services directly (Compose can't
> `depends_on` a service in a different project). A container that needs
> several of these joins all of them — see `backend/docker-compose.yaml`.
>
> **Naming gotcha to avoid:** don't let two services on networks the same
> container joins register the same Docker Compose service name. `app`
> joins both `backend-db-net` (where the service is named `db`, the
> backend's own Postgres) and `agents-db-net`. If the Postgres in
> `agents-db-net` were *also* named `db`, `app`'s DNS lookup for `db` would
> be ambiguous between two different databases. That's why
> `docker/services/postgres-ml`'s service is named `postgres-ml`, not `db`.

## 2. Infrastructure services (`docker/services/*`)

Bring these up first — `backend` and `agents` depend on them being healthy.

| Service | Path | Provides | Network |
|---|---|---|---|
| Postgres (backend) | [`docker/services/postgres`](../docker/services/postgres) | `rcagent` database | `backend-db-net` |
| Postgres (agents, pgvector) | [`docker/services/postgres-ml`](../docker/services/postgres-ml) | `incident_ml` database | `agents-db-net` |
| Kafka (Redpanda) | [`docker/services/kafka`](../docker/services/kafka) | job/result topics + web console | `backend-kafka-net` |
| RabbitMQ | [`docker/services/rabbitmq`](../docker/services/rabbitmq) | AI-agent escalation queue | `backend-rabbitmq-net` |
| Monitoring (optional) | [`docker/services/monitoring`](../docker/services/monitoring) | Grafana + Mimir + Loki behind Traefik | its own bridge networks |

For each of the four required services:

```bash
cd docker/services/<name>
cp .env.example .env   # fill in real credentials
docker compose up -d
```

Kafka publishes two listeners — `internal://kafka:9092` for containers on
`backend-kafka-net`, and `external://<KAFKA_ADVERTISED_HOST>:19092` for
observers on other machines. Set `KAFKA_ADVERTISED_HOST` in
`docker/services/kafka/.env` to this server's public IP/domain, and make
sure port `19092` is open in the firewall/security group.

The Redpanda web console sits behind an nginx Basic Auth proxy on `:8081`
(`KAFKA_CONSOLE_USER`/`PASSWORD` in the same `.env`) — the open-source
console has no login of its own.

Monitoring is optional and needs its own DNS records (`GRAFANA_DOMAIN`,
`MIMIR_DOMAIN`, `LOKI_DOMAIN` in its `.env`) plus a Traefik dynamic config
on the host at `/stack/monitoring/traefik` (not tracked in this repo — see
the comment in [`docker/services/monitoring/.env.example`](../docker/services/monitoring/.env.example)).
Skip it for a first deployment; nothing else depends on it.

**Postgres passwords only apply on first `initdb`.** If you change
`POSTGRES_PASSWORD` in an `.env` after the volume already exists, the
running database keeps the old password — Postgres does not re-read it on
every start. Either set the password correctly *before* the first
`docker compose up`, or fix a mismatch in place:
```bash
docker exec -it <container> psql -U postgres -c "ALTER USER rcagent WITH PASSWORD 'new-password';"
```

## 3. Application (`backend`, `frontend`, `agents`)

All three read their `.env` via `env_file:` and expect the networks from
step 1 to already exist and be healthy.

### Backend

```bash
cd backend
cp .env.example .env   # DATABASE_URL / NEWS_DATABASE_URL / RABBITMQ_URL / KAFKA_BROKERS
                        # must match the real credentials from step 2
docker compose up -d --build app
```

Database migrations (`migrations/*.sql`, applied via `goose`) run through the
`migrate` service in [`backend/docker-compose.yaml`](../backend/docker-compose.yaml).
It ships commented out in this repo — uncomment it (or run the equivalent
`goose` command directly) before the first `app` start against a fresh
database.

Exposes `:8080` (HTTP API + `/swagger/`). `NEWS_DATABASE_URL` is a
**read-only** second connection to `docker/services/postgres-ml` — the
backend never writes to the agents' database.

### Frontend

```bash
cd frontend
cp .env.example .env   # FRONTEND_PORT
docker compose up -d --build
```

The frontend does not need a backend URL at build or run time: its nginx
(see [`frontend/nginx.conf`](../frontend/nginx.conf)) proxies `/v1`, `/api`,
`/healthz` and `/swagger/` to `app:8080` over `rcagent-net`, so the browser
only ever talks to the frontend's own origin.

### Agents

```bash
cd agents
cp .env.example .env   # DATABASE_URL (-> postgres-ml) / RABBITMQ_URL / LLM_*
docker compose up -d --build news-worker ml3-consumer
```

Two long-running services:
- **`news-worker`** — continuously ingests configured RSS/HTML sources and
  extracts structured events (`--interval-seconds 900` by default).
- **`ml3-consumer`** — consumes `incident.analysis.requests` from RabbitMQ,
  runs retrieval + LLM reasoning, publishes back to
  `incident.analysis.results`. Builds from
  [`Dockerfile.ml3`](../agents/Dockerfile.ml3) (heavy — torch,
  sentence-transformers); expect the first build to take a while.

`e2e` is a one-shot smoke test, not a persistent service — run it manually
(`docker compose run --rm e2e`) when you want to verify the pipeline, not as
part of a routine deploy.

If `LLM_API_KEY` is empty or `LLM_PROVIDER=mock`, ML-3 falls back to a
mock LLM client instead of calling a real model — see
`ml3_reasoning/messaging/message_handler.py`.

## 4. Observer (separate machine)

Run on any machine with outbound network access to the server — it needs
**no** inbound ports and joins none of the server's Docker networks.

```bash
cd linux_binary
cp .env.example .env   # SERVER_IP or BACKEND_URL, KAFKA_BROKERS (the *external*
                        # listener, host:19092), TWIP_TOKEN, INSTALL_SECRET
SECRET=$(make gen-secret)   # or reuse the INSTALL_SECRET already in the backend's .env
make docker INSTALL_SECRET=$SECRET
docker compose up -d
```

`INSTALL_SECRET` must be the exact same value the backend was built/configured
with (`backend/.env`'s `INSTALL_SECRET`) — it is how the backend authorizes a
new observer registration. `KAFKA_BROKERS` here is the server's public
address on port `19092` (Kafka's external listener from step 2), not the
internal `kafka:9092` used inside the server's own Docker networks.

See [`linux_binary/README.md`](../linux_binary/README.md) for the full
configuration reference, systemd unit, and probe/field documentation.

## Verifying the deployment

```bash
# Infra containers healthy?
docker compose -f docker/services/postgres/docker-compose.yml ps
docker compose -f docker/services/postgres-ml/docker-compose.yml ps
docker compose -f docker/services/kafka/docker-compose.yml ps
docker compose -f docker/services/rabbitmq/docker-compose.yml ps

# Backend reachable and healthy?
curl -f http://<server>:8080/healthz

# Frontend serving and proxying?
curl -f http://<server>:<FRONTEND_PORT>/healthz

# Observer registered? Watch its logs for "connection established"
# followed (with KAFKA_BROKERS set) by "daemon mode started".
docker compose logs -f observer
```

A registered observer only starts receiving jobs once its derived (or
explicit `NODE_REGION`/`NODE_PROVIDER`) region/provider pair exists as an
**enabled** row in the backend's `region_providers` catalog — check
`GET /v1/region-providers`.

## Updating a running deployment

Code changes require an image rebuild, not just a restart — `docker compose
up -d` alone reuses the existing image if nothing about the compose file
changed:

```bash
docker compose build <service> && docker compose up -d <service>
```

Config-only changes (`.env`, compose file edits with no code change) need
the container **recreated**, not just restarted, since `env_file` is read
at container creation:

```bash
docker compose up -d <service>   # not `docker compose restart`
```

`docker/services/*` are not tracked the same way as the four application
repos — some paths are committed in the top-level `RCAgent` repo, but
`.env` files everywhere are gitignored on purpose (they hold real
credentials). Keep them in sync across machines by hand.
