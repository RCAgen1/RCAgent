# docker/

Shared infrastructure services — each is its own Compose project, on its
own `external` Docker network, independent of `backend`/`frontend`/`agents`.
See **[docs/deployment.md](../docs/deployment.md)** for the full deployment
order, required networks, and how these connect to the rest of the project.

```
docker/
└── services/
    ├── postgres/     — Postgres for the backend (`rcagent` database)
    ├── postgres-ml/  — Postgres/pgvector for agents (`incident_ml` database)
    ├── kafka/        — Redpanda (job/result topics) + web console
    ├── rabbitmq/     — RabbitMQ (AI-agent escalation queue)
    └── monitoring/   — Grafana + Mimir + Loki behind Traefik (optional)
```

There is no single `docker-compose.yml` that starts everything — each
service is brought up independently:

```bash
cd docker/services/<name>
cp .env.example .env   # fill in real credentials
docker compose up -d
```

`backend`, `frontend` and `agents` each have their own `docker-compose.yaml`/
`.yml` in their own repo, and join whichever of these services' networks
they need (see [docs/deployment.md](../docs/deployment.md#1-create-the-shared-docker-networks)
for the exact network names and why there are several instead of one).
