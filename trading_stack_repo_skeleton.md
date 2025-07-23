# 🏦 Local Hedge‑Fund Trading Stack — Repository Skeleton

A concise, fully‑wired starting point for running a complete quantitative‑trading lab on your RTX 4070 workstation (or any modern machine).  The layout delivers:

- **Infrastructure** — QuestDB 9, Apache Kafka 3, NautilusTrader 1.219, Prometheus, Grafana & Loki, all orchestrated with Docker Compose.
- **CI** — GitHub Actions workflow builds container images and runs smoke tests on every push.
- **CD** — A second workflow redeploys the stack automatically on pushes to `main` (via a self‑hosted runner).

Clone the file structure below, commit, and push—your Actions tab will light up immediately.

---

## 1 ▪ `README.md`

```markdown
# Local Hedge‑Fund Trading Stack

Run a multi‑container, research‑grade trading system locally in minutes.

**Components**
- **Data Ingestion **: Apache Kafka 3 · QuestDB 9
- **Backtesting  **: NautilusTrader (Rust + Python) · ABIDES
- **ML / RL**   : PyTorch 2.2 · FinRL‑DeepSeek
- **Observability**: Prometheus · Grafana · Loki

## Quick Start
```bash
# Build and launch everything
docker compose up -d --build

# Tear down
docker compose down
```
```

---

## 2 ▪ `docker-compose.yml`
```yaml
version: "3.9"

services:
  questdb:
    image: questdb/questdb:9.0
    restart: unless-stopped
    environment:
      - QDB_PG_PORT=8812
    ports:
      - "8812:8812"   # Postgres wire
      - "9000:9000"   # Web console

  zookeeper:
    image: confluentinc/cp-zookeeper:7.6.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    restart: unless-stopped
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    depends_on: [zookeeper]
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    restart: unless-stopped
    ports:
      - "9092:9092"

  nautilus:
    image: ghcr.io/nautechsystems/nautilus_trader:1.219.0-python
    depends_on: [kafka, questdb]
    environment:
      NT__CONFIG__REDIS__HOST: redis
    volumes:
      - ./config/nautilus:/nautilus/config
    restart: unless-stopped

  redis:
    image: redis:7.2-alpine
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:v2.52.0
    volumes:
      - ./config/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana-oss:11.0.0
    depends_on: [prometheus]
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
  grafana-data:
```

---

## 3 ▪ `.github/workflows/ci.yml`

```yaml
name: CI
on:
  push:
    branches: ["**"]
  pull_request:

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [questdb, kafka, nautilus]
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build ${{ matrix.service }}
        run: docker compose build ${{ matrix.service }}

      - name: Smoke tests (Nautilus only)
        if: matrix.service == 'nautilus'
        run: docker compose run --rm nautilus pytest -q
```

---

## 4 ▪ `.github/workflows/deploy.yml`

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: self-hosted   # Runner lives on your workstation
    steps:
      - uses: actions/checkout@v4

      - name: Pull latest images
        run: docker compose pull

      - name: Launch stack
        run: docker compose up -d --remove-orphans

      - name: Comment on commit
        uses: peter-evans/commit-comment@v3
        with:
          body: "🚀 Trading stack deployed on ${{ runner.name }} at $(date)"
```

> ### Required secrets
>
> - `GHCR_PAT` — token for GitHub Container Registry (if you push custom images).
> - any additional cloud keys your Terraform/SSH deploy might need.

---

## 5 ▪ `Makefile` (optional helper)
```makefile
up:
	docker compose up -d --build

down:
	docker compose down

logs:
	docker compose logs -f --tail=100
```

---

## 6 ▪ Next Steps

1. **Add a self‑hosted runner** (Settings ▸ Actions ▸ Runners ▸ *Add runner*).  Label it `self-hosted,workstation`.
2. **Create empty `config/` folders** for NautilusTrader, Prometheus, etc.
3. Drop a placeholder strategy file, e.g. `src/strategies/hello_strategy.py`, so tests pass.
4. Evolve `docker-compose.yml` as you add ABIDES, RedisAI, or other services.

Once these files are in place, every push builds all service images, and any push to **main** redeploys your local stack automatically.
