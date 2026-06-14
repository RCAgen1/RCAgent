# Docker

Все команды выполняются из **корня репозитория** (`RCAgent/`).

```
docker/
├── Dockerfile.frontend   — Vite + React → nginx (порт 80)
├── Dockerfile.backend    — Python FastAPI (порт 8000)
├── Dockerfile.agents     — Python ML-агенты (порт 8001)
├── docker-compose.yml    — поднимает все три сервиса
└── README.md
```

---

## Запуск всех сервисов сразу

```bash
# Собрать образы и запустить в фоне
docker compose -f docker/docker-compose.yml up --build -d

# Посмотреть логи всех сервисов
docker compose -f docker/docker-compose.yml logs -f

# Остановить и удалить контейнеры
docker compose -f docker/docker-compose.yml down
```

После запуска:
| Сервис   | URL                   |
|----------|-----------------------|
| Frontend | http://localhost:3000 |
| Backend  | http://localhost:8000 |
| Agents   | http://localhost:8001 |

---

## Запуск отдельного сервиса

### Frontend

```bash
docker build -f docker/Dockerfile.frontend -t rcagent-frontend ./frontend
docker run -p 3000:80 rcagent-frontend
```

### Backend

```bash
docker build -f docker/Dockerfile.backend -t rcagent-backend ./backend
docker run -p 8000:8000 rcagent-backend
```

### Agents

```bash
# Создать volume для кеша моделей (один раз)
docker volume create models_cache

docker build -f docker/Dockerfile.agents -t rcagent-agents ./agents
docker run -p 8001:8001 -v models_cache:/root/.cache/huggingface rcagent-agents
```

> Volume `models_cache` нужен чтобы sentence-transformers не скачивал модели (~400 MB)
> заново при каждом `docker run`.

---

## Полезные команды

```bash
# Пересобрать конкретный сервис без кеша
docker compose -f docker/docker-compose.yml build --no-cache agents

# Зайти в запущенный контейнер
docker compose -f docker/docker-compose.yml exec frontend sh

# Посмотреть логи только одного сервиса
docker compose -f docker/docker-compose.yml logs -f backend

# Удалить все остановленные контейнеры и неиспользуемые образы
docker system prune
```
