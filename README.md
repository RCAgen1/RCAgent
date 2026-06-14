# RCAgent
Differentiating technical outages from censorship-induced blocks using two specialized AI agents.

### Architecture
```mermaid
flowchart TB
    C([Client])

    W[Web Page]

    B[Backend]
    K[(Kafka)]

    W -->|Source Link| B


    subgraph MNT["Monitoring"]
        A[Alloy]
        M[(Mimir)]
        L[(Loki)]
        G[Grafana]
    
        A -->|Metrics| M
        A -->|Logs| L
        M -->|Metrics| G
        L -->|Logs| G
    end
    B -->|Data| A
    B -->|Events| K
    C -->|Source Link| W

    subgraph OBS["Observers"]
        subgraph R1["Region 1"]
            A1[APK]
            B1[BIN]
        end
        subgraph R2["Region 2"]
            A2[APK]
            B2[BIN]
        end

    end

    subgraph AGN["Agents"]
        S[Scrapper]
        LA[Log Agent]
        KA[Knowledge Agent]

        LA -->|Incident Response| B
        B -->| Call on Incident| LA
    end

    WO[World(Subscriptions,Sites...)]
    WO -->|Knowledge| S
    S -->|Filtered&Mapped Knowledge| KA

    subgraph DB["Database"]
        KD[(Knowledge)]
        ID[("Incidents")]
        BD[(Backend)]

        B <-->|Auth,Templates,Settings| BD
        B <-->|Incidents| ID
        KA -->|Knowledge| KD
    end

    A1 & B1 & A2 & B2 -->|Metrics| K

    classDef monitoring fill:#ff9800,stroke:#e65100,color:#000
    classDef storage  fill:#1565c0,stroke:#0d47a1,color:#fff
    classDef observer fill:#2e7d32,stroke:#1b5e20,color:#fff
    classDef client   fill:#6a1b9a,stroke:#4a148c,color:#fff
    classDef frontend fill:#00695c,stroke:#004d40,color:#fff

    class A,G monitoring
    class M,L,K,KD,ID,BD storage
    class A1,B1,A2,B2 observer
    class C client
    class W frontend
```

