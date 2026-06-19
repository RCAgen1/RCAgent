# RCAgent

Determines whether a service is unreachable due to a technical outage or censorship-based blocking, using distributed observers and AI agents.

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
