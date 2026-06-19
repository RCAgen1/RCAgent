# RCAgent

Определяет, недоступен ли сервис из-за технической неисправности или из-за цензурной блокировки, используя распределённых наблюдателей и ИИ-агентов.

---

### Архитектура

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

### Как это работает

1. **Отправка** — пользователь вводит URL своего сервиса в веб-интерфейсе.
2. **Распределение** — бэкенд публикует задачу на проверку в Kafka; наблюдатели, подписанные на соответствующий канал, получают её.
3. **Проверка** — каждый наблюдатель (развёрнут в отдельном регионе у отдельного провайдера) пингует целевой сервис и собирает данные по HTTP и ICMP.
4. **Сбор** — результаты проб возвращаются на бэкенд.
5. **Решение** — бэкенд оценивает данные. Если сервис недоступен, он сначала пытается самостоятельно классифицировать причину по имеющимся данным. Если причина не очевидна — формирует инцидент (логи, метрики, результаты наблюдателей) и публикует его в RabbitMQ.
6. **Анализ** — Log Agent получает задачу, читает текущие логи и сопоставляет данные с накопленной базой знаний Knowledge Agent (новости, блок-листы, события из фидов). Возвращает диагноз.
7. **Отчёт** — итоговый вердикт и сопутствующие данные передаются пользователю через фронтенд.

Knowledge Agent работает в фоновом режиме постоянно: индексирует внешние источники через Scrapper и сохраняет структурированные знания в базу данных.

---

### Компоненты

| Компонент | Роль |
|---|---|
| **Frontend** | Веб-интерфейс для отправки URL и просмотра результатов проверок |
| **Backend** | Оркестрирует проверки, принимает первичное решение, маршрутизирует инциденты к агентам |
| **Kafka** | Распределяет задачи на проверку от бэкенда к наблюдателям |
| **RabbitMQ** | Шина сообщений между бэкендом и агентной системой |
| **Observers** | Лёгкие зонды в разных регионах и у разных провайдеров — выполняют HTTP и ICMP проверки целевого сервиса |
| **Log Agent** | LLM-агент, анализирующий данные инцидента (логи + метрики + база знаний) и определяющий первопричину |
| **Knowledge Agent** | Пассивно строит базу знаний из новостей, фидов и событий блокировок |
| **Scrapper** | Собирает и фильтрует контент из внешних источников для Knowledge Agent |
| **Database** | Хранит инциденты, базу знаний и конфигурацию бэкенда |
| **Monitoring** | Grafana-стек (Alloy + Mimir + Loki) для внутренней наблюдаемости: нагрузка на серверы, доступность наблюдателей и агентов, пропускная способность очередей, CPU/память |
