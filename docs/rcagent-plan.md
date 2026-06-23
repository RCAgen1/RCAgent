# RCAgent — Project Plan (Weeks 1–8)

**Project:** RCAgent — AI-powered root cause analysis for service unavailability  
**Goal:** Build a system that monitors service availability from multiple Russian regions and uses two AI agents to diagnose whether an outage is a technical failure or external interference (blocking, DDoS, censorship).

---

## Team Tracks

| Track | Stack |
|---|---|
| **Backend** | Go (net/http + mux), PostgreSQL, Kafka, Prometheus |
| **ML / AI Agents** | Python 3.11+, FastAPI, sentence-transformers (E5), numpy, FAISS, GLiNER, LLM API, feedparser, beautifulsoup4, SQLAlchemy, Alembic, PostgreSQL |
| **Probe Agent** | Static Linux binary (→ Android APK optional) |
| **Infrastructure** | Docker Compose, Uptime Kuma, Loki, GitHub Actions |
| **UX / Frontend** | Figma, React 19 + Vite 8, Tailwind CSS 3, React Router DOM 7, React-Leaflet + Leaflet (Esri tile layer for map), Axios (planned for API integration) |

---

## Week 1 — Idea, Planning & Team Setup

**All**
- Idea proposition, team gathering, writing a project plan and roadmap.

---

## Week 2 — Foundation & Schema Alignment

**Backend**
- Repo setup, Docker Compose (Go + PostgreSQL + Kafka), base HTTP server with health checks, minimal end-to-end HTTP probe with persisted result.

**ML / AI Agents**
- Align with backend on incident log format and response schema; design DB schema (`news`, `events`, `incidents`, `candidate_logs`), write migrations and indexes; implement RSS ingestion for target cities and store raw news in DB; set up E5 embedding model, implement text-to-vector conversion and embedding persistence; define I/O schemas (Incident, Event, CandidateEvent, ReasoningResult) and create reasoning service skeleton.

**UX / Frontend**
- Create low-fidelity wireframes in Figma for all 4 core screens: Overview, Incident Detail, Probe Agents, News Feed.
- Set up React + Vite project with Tailwind CSS and React Router; implement Sidebar, TopBar, and skeleton pages for all screens.

---

## Week 3 — Core API, Ingestion & Entity Extraction

**Backend**
- User authentication (registration, login, JWT); CRUD for monitoring targets and in-process scheduler for HTTP probes; prototype Kafka producer–consumer message flow.

**ML / AI Agents**
- News deduplication (URL + text signatures), cleaning and normalization; GLiNER for entity extraction (provider, region, event type) with LLM fallback; full News→Event pipeline with structured events saved to DB; extend pipeline with embedding generation and storage; basic nearest-neighbor retrieval (numpy, temporary); context builder combining incident with candidate events; initial FastAPI skeleton with mock POST /analyze_incident connected to backend — first end-to-end call.

**UX / Frontend**
- High-fidelity Figma designs for all 4 screens (colors, typography, component styles).
- Full React implementation of all 4 screens with mock data: Overview, Incident Detail, Probe Agents, News Feed.

---

## Week 4 — MVP + Integration

**Backend**
- Incident lifecycle (`operational → down → recovered`) with deduplication; Kafka topics and message schema; API producer + consumer worker; region-aware worker service.

**ML / AI Agents**

*ML-1:* Improve deduplication and cleaning based on collected data analysis. Connect to the shared PostgreSQL database.

*ML-2:* Implement FAISS-based retrieval service (replace numpy). Connect to the shared PostgreSQL database (replace SQLite). Add logging of all retrieved candidate events. Return fully populated CandidateEvent objects (event_id, provider, region, summary, timestamp, score).

*ML-3:* Implement rule-based scoring: embedding similarity, temporal distance, region match, provider match, event type match. Implement candidate filtering based on minimum score threshold. Replace mock with real /analyze_incident logic. Implement broker skeleton. Design BaseLLMClient abstraction with implementations for OpenAI-compatible providers (OpenAI, Deepseek, Ollama). Add LLM provider configuration via .env.

*All:* Merge develop → main. Full end-to-end test on a real incident with FAISS and shared database. Connect mock response via broker to the backend — first real integration. Document all bugs found.

**Checkpoint:** backend sends an incident via broker, ML service returns a mock response. FAISS and shared database are operational.

**UX / Frontend**
- Begin connecting frontend pages to live backend API as endpoints become available; replace mock data progressively.

---

## Week 5 — LLM + Real Response to Backend

**Backend**
- Region-aware message routing and per-region result aggregation; error classification (timeout, DNS, TLS, HTTP 5xx); message reliability (at-least-once delivery, idempotent handling, dead-letter topic).

**ML / AI Agents**

*ML-1:* Final stabilization of the ingestion pipeline. Improve structured event extraction quality.

*ML-2:* Tune FAISS retrieval thresholds. Measure Recall and Precision. Build feature dataset from retrieval logs. Stretch goal: start training LightGBM ranking model on accumulated logs.

*ML-3:* Design the LLM prompt based on incidents and ranked candidate events. Implement LLM API integration and response parsing via BaseLLMClient. Implement confidence estimation. Replace mock response with real LLM response — backend starts receiving real results. Create evaluation dataset with synthetic incidents agreed with backend.

*All:* Verify that real LLM responses are correctly delivered to the backend via broker.

**Checkpoint:** full stack operational with real LLM response delivered via broker to the backend.

**UX / Frontend**
- Continue API integration; fix any issues found during backend–frontend wiring.

---

## Week 6 — Testing + Final Integration

**Backend**
- Expose Prometheus metrics (check volume, latency, incidents, consumer lag); full end-to-end integration and feature freeze.

**ML / AI Agents**

*ML-1:* Test and improve entity extraction quality on real data.

*ML-2:* Re-evaluate retrieval quality: Recall, Precision, tuning thresholds. Stretch goal: finalize LightGBM, compare against rule-based baseline.

*ML-3:* Implement evaluator.py and test_cases.json. Run automated evaluation on the prepared dataset. Fix issues found during testing.

*All:* Final integration check with the backend — all contracts, formats, edge cases, stress test. Analyze end-to-end results on real data, prioritize bugs.

**Checkpoint:** backend integration fully verified, all critical bugs identified and fixed.

**UX / Frontend**
- Full integration pass: all 4 pages pull live data from the backend API.
- Set up Uptime Kuma dashboards: probe health, incident rate, Kafka consumer lag, LLM call latency.

---

## Week 7 — Final Touches + Demo

**Backend**
- Unit and integration tests (containerized PostgreSQL + Kafka); bug fixing and reliability improvements; README and architecture diagram.

**ML / AI Agents**

*ML-1:* Fix ingestion pipeline bugs found during integration.

*ML-2:* Final retrieval tuning based on Week 5 results. Fix retrieval-related issues.

*ML-3:* Prepare demo-ready JSON responses. Validate the complete reasoning workflow end-to-end.

*All:* Support backend during final assembly. Prepare for demo.

**Final deliverable:** complete reasoning pipeline via message broker, evaluated on real data, demo-ready.

**UX / Frontend**
- Bug fixes and UI polish.

**All**
- Prepare and rehearse the live demonstration scenario: probe detects outage → agents diagnose → AI verdict shown on dashboard. Prepare 15-min video.

---

## Week 8 — Presentation

**PRESENTATION ON JULY 20**

---

## Key Milestones

| Week | Milestone |
|---|---|
| End of W2 | Single probe check works end-to-end; schemas agreed; Figma wireframes done; React skeleton running |
| End of W3 | News ingestion with embeddings live; first end-to-end ML call; all 4 frontend screens implemented with mock data |
| End of W4 | Full incident lifecycle working; FAISS retrieval live; frontend API wiring started |
| End of W5 | Main Agent produces root-cause verdicts; frontend fully connected to available endpoints |
| End of W6 | Full system integrated end-to-end; Uptime Kuma dashboards live; all pages on live data |
| End of W7 | Tested, documented, and demo-ready, video is made |
| W8 | Presentation on July 20 |

---

## Inter-Team Dependencies

- **W2:** ML and Backend must agree on data schemas before any agent development begins.
- **W3 (Frontend → Backend):** Frontend can replace mock data with live data only after Backend exposes REST API endpoints for services, incidents, and probe results.
- **W3 (ML → Backend):** FastAPI mock endpoint (POST /analyze_incident) requires a stable incident schema from Backend before the first end-to-end call.
- **W4 (ML → Backend):** Main Agent integration depends on Log Analyst and News Scraper being stable and delivering structured output.
- **W5 (Frontend → Backend):** Full frontend integration requires the Backend API to be feature-frozen.
