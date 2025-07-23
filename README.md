# Local Hedge‑Fund Trading Stack

Run a multi‑container, research‑grade trading system locally in minutes.

**Components**
- **Data Ingestion**: Apache Kafka 3 · QuestDB 9
- **Backtesting**: NautilusTrader (Rust + Python) · ABIDES
- **ML / RL**: PyTorch 2.2 · FinRL‑DeepSeek
- **Observability**: Prometheus · Grafana · Loki

## Quick Start
```bash
docker compose up -d --build   # Build and launch everything
docker compose down            # Tear down
```
