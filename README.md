# RCAgent

Determines whether a service is unreachable due to a technical outage or censorship-based blocking, using distributed observers and AI agents.

> **Track:** Industrial · **Team 62**, Innopolis University · **Live demo:** [http://161.104.53.210:3000](http://161.104.53.210:3000)

---

### Overview

Conventional uptime monitors (e.g. Uptime Kuma) can only tell you *that* a service is down — a single global up/down flag per monitor. They cannot tell you **why**, and they cannot distinguish a real outage from region- or ISP-specific blocking.

RCAgent closes that gap. Distributed observers probe each target from multiple Russian regions and ISPs, so one service can be reported `down` from Rostelecom yet `up` from MTS at the same time, tracked as two separate per-vantage incidents. When a target is unreachable, the backend first classifies the cause itself using a closed-vocabulary rule engine (block page, DNS tampering, SNI/DPI, TCP reset, TLS failure, HTTP error, throttling…). Definitive causes are self-resolved with full confidence; ambiguous cases are escalated to AI agents that cross-reference real-time logs with a continuously indexed knowledge base of provider news and block-list events, returning a structured root cause with a confidence score and supporting evidence.

### Features

- **Root-cause classification, not just up/down** — a 12-rule closed-vocabulary classifier derives the specific cause (DNS-block, IP-block, SNI/DPI, TLS/cert, HTTP 4xx/5xx, throttling, and more) with a confidence score.
- **Self-classify vs. AI escalation** — the backend resolves definitive causes on its own and escalates only ambiguous/`unknown` cases to the AI agents, keeping cost and noise low.
- **Per-vantage incidents** — outages are tracked per `(target, region, provider)`, so regional blocking is visible instead of being averaged away.
- **AI verdicts with evidence** — an LLM-based Log Agent retrieves semantically similar provider-news events and returns a diagnosis with confidence and citations.
- **Continuous knowledge base** — a Knowledge Agent + Scrapper index external feeds/news in the background for correlation with incidents.
- **Multi-region distributed observers** — lightweight probes deployed across regions and ISPs, dispatched via Kafka.
- **Auth for humans and agents** — JWT register/login for users; secret + public-key registration/refresh for observer agents.
- **Alerting & dashboards** — Telegram/Slack alerts plus a Grafana (Alloy + Mimir + Loki) monitoring stack.

---

### Architecture

```mermaid
flowchart TB
    C([User])
    W[Frontend]
    B[Backend]
    K[(Kafka)]
    RMQ[(RabbitMQ)]
    WO[External Sources]
    DEV([Developer])

    subgraph OBS[Observers]
        R1[Region 1]
        R2[Region 2]
    end

    subgraph AGN[Agents]
        LA[Log Agent]
        KA[Knowledge Agent]
        S[Scrapper]
    end

    subgraph DB[Database]
        KD[(Knowledge)]
        ID[(Incidents)]
        BD[(Backend DB)]
    end

    subgraph MON[Monitoring]
        AL[Alloy]
        MI[(Mimir)]
        LO[(Loki)]
        GR[Grafana]
        AL -->|metrics| MI
        AL -->|logs| LO
        MI --> GR
        LO --> GR
    end

    C -->|Submit URL| W
    W -->|Request| B
    B -->|Publish task| K
    K -->|Task| R1
    K -->|Task| R2
    R1 -->|Probe results| B
    R2 -->|Probe results| B
    B -->|Incident| RMQ
    RMQ -->|Task| LA
    LA -->|Analysis| RMQ
    RMQ -->|Result| B
    B -->|Report| W
    W -->|Status| C

    WO -->|Feeds| S
    S -->|Knowledge| KA
    KA -->|Store| KD
    LA -.->|Read| KD

    B -->|R/W| BD
    B -->|R/W| ID

    B -->|Telemetry| AL
    R1 -->|Metrics| AL
    R2 -->|Metrics| AL
    DEV -->|View| GR

    classDef client     fill:#6a1b9a,stroke:#4a148c,color:#fff
    classDef frontend   fill:#00695c,stroke:#004d40,color:#fff
    classDef backend    fill:#1a237e,stroke:#0d1642,color:#fff
    classDef broker     fill:#e65100,stroke:#bf360c,color:#fff
    classDef monitoring fill:#f57f17,stroke:#e65100,color:#000
    classDef storage    fill:#1565c0,stroke:#0d47a1,color:#fff
    classDef observer   fill:#2e7d32,stroke:#1b5e20,color:#fff
    classDef agent      fill:#4a148c,stroke:#311b92,color:#fff
    classDef external   fill:#37474f,stroke:#263238,color:#fff
    classDef dev        fill:#880e4f,stroke:#560027,color:#fff

    class C client
    class W frontend
    class B backend
    class K,RMQ broker
    class AL,GR monitoring
    class MI,LO,KD,ID,BD storage
    class R1,R2 observer
    class S,LA,KA agent
    class WO external
    class DEV dev
```

---

### How It Works

1. **Submit** — the user enters a service URL in the web interface.
2. **Distribute** — the backend publishes a check task to Kafka; observers subscribed to the relevant channel receive it.
3. **Probe** — each observer (deployed in a different region and on a different ISP) pings the target service and collects HTTP/ICMP data.
4. **Collect** — probe results are returned to the backend.
5. **Decide** — the backend evaluates the results. If the service is inaccessible, it first attempts to classify the cause from available data alone. If the cause is inconclusive, it packages logs and metrics into an incident and publishes it to RabbitMQ.
6. **Analyze** — the Log Agent receives the incident task, reads real-time logs, and cross-references the Knowledge Agent's accumulated data (news, block lists, feed events). It returns a diagnosis.
7. **Report** — the final verdict and supporting data are delivered to the user via the frontend.

The Knowledge Agent runs continuously in the background, indexing external sources through the Scrapper and persisting structured knowledge to the database.

---

### Components

| Component | Role |
|---|---|
| **Frontend** | Web UI for submitting URLs and viewing check results and reports |
| **Backend** | Orchestrates checks, makes first-pass decisions, routes incidents to agents |
| **Kafka** | Distributes check tasks from backend to observers |
| **RabbitMQ** | Message bus between backend and the agent system |
| **Observers** | Lightweight probes in multiple regions and ISPs — perform HTTP and ICMP checks against the target |
| **Log Agent** | LLM-based agent that analyzes incident data (logs + metrics + knowledge) and determines root cause |
| **Knowledge Agent** | Passively builds a knowledge base from news, feeds, and block-list events |
| **Scrapper** | Fetches and filters content from external sources for the Knowledge Agent |
| **Database** | Stores incidents, knowledge base, and backend configuration |
| **Monitoring** | Grafana stack (Alloy + Mimir + Loki) for internal observability: server load, observer and agent availability, queue throughput, CPU/memory |

---

### Tech Stack

| Layer | Technologies |
|---|---|
| **Backend** | Go (`net/http` + `gorilla/mux`), PostgreSQL (`pgx`), Kafka (`segmentio/kafka-go`), RabbitMQ (`amqp091-go`) |
| **ML / AI Agents** | Python 3.11+, sentence-transformers (E5 — `intfloat/e5-large-v2`, 1024-dim), FAISS, GLiNER, PyTorch, numpy, SQLAlchemy, `asyncpg` + `pgvector`, `feedparser`, `httpx`, `pika` (RabbitMQ), OpenAI-compatible LLM API (OpenAI / Deepseek / Ollama via `BaseLLMClient`) |
| **Observers** | Static Linux binary (HTTP + ICMP probes) |
| **Frontend** | React 19 + Vite, Tailwind CSS 3, React Router DOM 7, React-Leaflet + Leaflet (Esri tiles), Axios |
| **Infrastructure** | Docker Compose, GitHub Actions (CI), Grafana + Alloy + Mimir + Loki |

---

### Setup

RCAgent is not a single-command demo — it's a distributed system: shared
infrastructure (Postgres ×2, Kafka, RabbitMQ), the backend/frontend/agents
application layer, and one or more observer nodes on **separate** machines
with no shared Docker network. Cloning this repo alone is not enough; the
`backend`, `frontend`, `agents` and `linux_binary` directories are separate
Git repositories checked out as submodules:

```bash
git clone --recurse-submodules https://github.com/RCAgen1/RCAgent.git
cd RCAgent
```

**→ See [`docs/deployment.md`](docs/deployment.md) for the full deployment
guide** — the required Docker networks, every service's `.env`/compose
commands in order, and how the observer connects from a different machine.
[`docker/README.md`](docker/README.md) covers just the shared infrastructure
layer (`docker/services/*`).

---

### Deployment

The feature-complete build is deployed on the project VM (see
[`docs/deployment.md`](docs/deployment.md) for how):

- **Application (frontend):** [http://161.104.53.210:3000](http://161.104.53.210:3000)

---

### Results (current snapshot)

The pipeline was run against real configured sources — 17 enabled sources producing 4723 raw news items, 4330 structured events (0 failed), and 1003 incidents. The metrics below come from **small curated verification sets** used as smoke checks and for threshold selection; they are not held-out production benchmarks.

| Stage | Metric | Value | Scope / caveat |
|---|---|---|---|
| Extraction | event-type / provider accuracy, region recall | 1.000 | Curated verification set of **10 cases** |
| Retrieval | recall@1 / @3 / @5 | 1.000 | **11 queries** — relevant item always in top-5 |
| Retrieval | precision@5 (default) | 0.200 | ~1 relevant per 5 retrieved before thresholding |
| Retrieval | precision / recall / f1 at `score_threshold = 0.75` | 1.000 / 1.000 / 1.000 | Best f1 in the threshold sweep, **tuned on the same 11-query set** (in-sample) |
| Reasoning | cause accuracy | 1.000 | **4 deterministic mock-LLM smoke cases**; production-LLM quality **not yet measured** |

The `0.75` threshold was selected from a sweep (precision rises from 0.611 at 0.30 to 1.000 at 0.75 while recall stays 1.000, then recall drops at 0.80+). Full threshold sweep, event-type distribution, and evaluation details are in [`agents/docs`](https://github.com/RCAgen1/agents/tree/main/docs) (`ml1.md`, `ml2.md`, `ml3.md`).

---

### Documentation

- **Deployment guide:** [`docs/deployment.md`](docs/deployment.md)
- **API contract:** [`backend/docs/contracts/api.md`](https://github.com/RCAgen1/backend/blob/main/docs/contracts/api.md)
- **Backend docs:** [`backend/docs`](https://github.com/RCAgen1/backend/tree/main/docs)
- **ML docs:** [`agents/docs`](https://github.com/RCAgen1/agents/tree/main/docs) — `ml1.md` (extraction & validation), `ml2.md` (retrieval & thresholds), `ml3.md` (LLM evaluation)
- **Blocking classification reference:** [`docs/blocking.ru.md`](docs/blocking.ru.md)
- **Project plan:** [`docs/rcagent-plan.md`](docs/rcagent-plan.md)

A full index of every document in the project is at the
[bottom of this README](#documentation-index).

---

### Team & Contributions

**Team 62 — Industrial track**, Innopolis University.

| Track | Member | Role & Contribution |
|---|---|---|
| **Frontend / Design** | Ilmira Usmanova | Team Leader, Frontend, Designer — UI, Figma design system, logo, reports |
| **Backend / DevOps** | Egor Kazakov | Backend & DevOps — deployment of backend/frontend/observer to the VM, bug fixes across all three |
| **Backend** | Grigorii Zhivolup | Full-stack integration — incident engine, per-vantage model, and AI-escalation flow wired end-to-end |
| **Backend** | Alexei Bondarchuk | Testing & CI — unit tests (mockery / pgxmock / testify) and integration tests (testcontainers: Postgres + Kafka) |
| **ML** | Georgii Pyanov | ML-1 — event extraction on real sources, reproducible verification script, pattern/alias fixes (power/provider outage, event-type priority, weather, RSS region fallback), `score_threshold = 0.75` selection, `docs/ml1.md` |
| **ML** | Anna Tikhonova | ML-2 — end-to-end orchestration service, retrieval & pre-deployment layer, `docs/ml2.md` |
| **ML** | Maria Karpova | ML-3 — LLM-judge evaluation pipeline, failure-case analysis, `docs/ml3.md` |

---

### Documentation Index

Every document in the project, by repository. `backend`, `frontend`, `agents`
and `linux_binary` are separate Git repositories (checked out here as
submodules), so their docs are linked on GitHub rather than as local paths.

#### This repo (`RCAgent`)

- [`docs/deployment.md`](docs/deployment.md) — full deployment guide
- [`docs/rcagent-plan.md`](docs/rcagent-plan.md) — project plan
- [`docs/blocking.ru.md`](docs/blocking.ru.md) — blocking-classification reference (RU)
- [`docs/monitoring.ru.md`](docs/monitoring.ru.md) — monitoring stack notes (RU)
- [`docs/README.ru.md`](docs/README.ru.md) — repo overview (RU)
- [`docker/README.md`](docker/README.md) — shared infrastructure services

#### [`backend`](https://github.com/RCAgen1/backend)

- [`README.md`](https://github.com/RCAgen1/backend/blob/main/README.md)
- [`docs/README.md`](https://github.com/RCAgen1/backend/blob/main/docs/README.md) — docs index
- **Architecture:** [`components.md`](https://github.com/RCAgen1/backend/blob/main/docs/architecture/components.md) · [`go-service-layout.md`](https://github.com/RCAgen1/backend/blob/main/docs/architecture/go-service-layout.md) · [`incident-engine.md`](https://github.com/RCAgen1/backend/blob/main/docs/architecture/incident-engine.md)
- **Contracts:** [`api.md`](https://github.com/RCAgen1/backend/blob/main/docs/contracts/api.md) · [`db-schema.md`](https://github.com/RCAgen1/backend/blob/main/docs/contracts/db-schema.md) · [`error-classes.md`](https://github.com/RCAgen1/backend/blob/main/docs/contracts/error-classes.md) · [`kafka-messages.md`](https://github.com/RCAgen1/backend/blob/main/docs/contracts/kafka-messages.md) · [`rabbitmq-messages.md`](https://github.com/RCAgen1/backend/blob/main/docs/contracts/rabbitmq-messages.md)
- **Decisions (ADRs):** [`0001-kafka`](https://github.com/RCAgen1/backend/blob/main/docs/decisions/adr-0001-kafka.md) · [`0002-postgres`](https://github.com/RCAgen1/backend/blob/main/docs/decisions/adr-0002-postgres.md) · [`0003-region-as-config`](https://github.com/RCAgen1/backend/blob/main/docs/decisions/adr-0003-region-as-config.md) · [`0004-server-side-classification`](https://github.com/RCAgen1/backend/blob/main/docs/decisions/adr-0004-server-side-classification.md) · [`0005-auth-jwt`](https://github.com/RCAgen1/backend/blob/main/docs/decisions/adr-0005-auth-jwt.md) · [`0006-no-checks-table-in-mvp`](https://github.com/RCAgen1/backend/blob/main/docs/decisions/adr-0006-no-checks-table-in-mvp.md)
- **Guides:** [`configuration.md`](https://github.com/RCAgen1/backend/blob/main/docs/guides/configuration.md)
- **Observability:** [`logging.md`](https://github.com/RCAgen1/backend/blob/main/docs/observability/logging.md) · [`metrics.md`](https://github.com/RCAgen1/backend/blob/main/docs/observability/metrics.md)
- [`test/integration/README.md`](https://github.com/RCAgen1/backend/blob/main/test/integration/README.md) — integration test setup

#### [`agents`](https://github.com/RCAgen1/agents)

- [`README.md`](https://github.com/RCAgen1/agents/blob/main/README.md)
- [`docs/setup.md`](https://github.com/RCAgen1/agents/blob/main/docs/setup.md)
- [`docs/architecture.md`](https://github.com/RCAgen1/agents/blob/main/docs/architecture.md)
- [`docs/config.md`](https://github.com/RCAgen1/agents/blob/main/docs/config.md)
- [`docs/broker.md`](https://github.com/RCAgen1/agents/blob/main/docs/broker.md) — RabbitMQ contract
- [`docs/contracts.md`](https://github.com/RCAgen1/agents/blob/main/docs/contracts.md)
- [`docs/database.md`](https://github.com/RCAgen1/agents/blob/main/docs/database.md)
- **ML pipeline stages:** [`ml1.md`](https://github.com/RCAgen1/agents/blob/main/docs/ml1.md) (extraction) · [`ml2.md`](https://github.com/RCAgen1/agents/blob/main/docs/ml2.md) (retrieval) · [`ml3.md`](https://github.com/RCAgen1/agents/blob/main/docs/ml3.md) (reasoning/LLM)
- [`docs/ml_event_extraction_part.md`](https://github.com/RCAgen1/agents/blob/main/docs/ml_event_extraction_part.md) · [`docs/ml_retrieval_part.md`](https://github.com/RCAgen1/agents/blob/main/docs/ml_retrieval_part.md) · [`docs/ml_reasoning_part.md`](https://github.com/RCAgen1/agents/blob/main/docs/ml_reasoning_part.md)

#### [`frontend`](https://github.com/RCAgen1/frontend)

- [`README.md`](https://github.com/RCAgen1/frontend/blob/main/README.md)

#### [`linux_binary`](https://github.com/RCAgen1/linux_binary) (observer)

- [`README.md`](https://github.com/RCAgen1/linux_binary/blob/main/README.md) — build, configuration, deployment, probe/field reference
