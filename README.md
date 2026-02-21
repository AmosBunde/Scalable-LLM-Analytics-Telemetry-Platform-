# Scalable LLM Analytics & Telemetry Platform

Build a mini analytics infrastructure that **collects usage/telemetry from an AI app**, processes high‑volume events, and serves **near real‑time metrics** for model performance, feature adoption, and reliability.

This repo is designed to be:
- **Runnable locally** with Docker Compose (fast iteration).
- **Deployable to cloud** (VM-first, optional Kubernetes).
- A strong portfolio showcase for **analytics infra**, **streaming**, **observability**, and **data quality**.

---

## What this system does

### Telemetry you can simulate/ingest
- **Latency** (p50/p95/p99, end-to-end and model-only)
- **Token usage** (input/output/total)
- **Errors** (timeouts, rate limits, 5xx, model errors)
- **Retention / sessions** (user/session ids, churn)
- **Experiment/model versions** (A/B, canary, model_id, prompt_version)

### Processing flow (high level)
1. **Collectors** receive telemetry (HTTP/gRPC) and publish to **Kafka**.
2. **Stream processor** (Flink or Spark Structured Streaming) enriches, validates, and writes:
   - **Raw events** → object storage (Parquet) for replay/backfill
   - **Aggregations** → ClickHouse for fast metrics queries
3. **API service** exposes metrics endpoints and powers a small dashboard.
4. **Observability** via Prometheus + Grafana (pipeline health, lag, QPS, error budgets).

---

## Tech stack

- **Collectors:** Go (recommended) or C++ (high-performance option)
- **Streaming:** Flink (recommended) or Spark Structured Streaming
- **Event bus:** Kafka
- **Warehouse/OLAP:** ClickHouse (metrics) + Parquet lake (raw)
- **Orchestration (optional):** Airflow for batch backfills / daily rollups
- **API:** FastAPI
- **UI:** React (minimal)
- **Observability:** Prometheus + Grafana
- **Infra:** Docker Compose locally; VM-first in cloud; Kubernetes optional

---

## Architecture diagram (text)

```
AI App / Simulator
      |
      v
Collectors (Go/C++)  ---> Kafka (events topics) ---> Stream Processor (Flink/Spark)
      |                                      \            |
      |                                       \           v
      |                                        ---> Raw Lake (Parquet on S3/MinIO)
      |
      v
API (FastAPI) <--- ClickHouse (aggregates) <--- stream outputs / batch jobs
      |
      v
React Dashboard

Prometheus scrapes metrics from: collectors, kafka, processor, api, clickhouse
Grafana visualizes: latency, tokens, errors, retention, lag, SLIs/SLOs
```

---

## Repository structure (suggested)

> If you already have files, align them to this layout. The local/cloud instructions assume this structure.

```text
llm-telemetry-analytics/
├─ README.md
├─ .gitignore
├─ .env.example
├─ docker-compose.yml
├─ Makefile
├─ docs/
│  ├─ architecture.md
│  ├─ event-schema.md
│  ├─ dashboards/
│  │  ├─ grafana/             # exported dashboards JSON
│  │  └─ prometheus/          # alert rules
│  └─ runbooks/
│     ├─ kafka.md
│     ├─ clickhouse.md
│     └─ troubleshooting.md
├─ infra/
│  ├─ terraform/
│  │  ├─ aws/                 # optional
│  │  └─ hetzner/             # optional
│  ├─ k8s/                     # optional k8s manifests/helm
│  └─ ansible/                 # optional VM provisioning
├─ services/
│  ├─ collector-go/
│  │  ├─ cmd/collector/
│  │  ├─ internal/
│  │  ├─ Dockerfile
│  │  └─ go.mod
│  ├─ processor-flink/
│  │  ├─ src/
│  │  ├─ Dockerfile
│  │  └─ README.md
│  ├─ api/
│  │  ├─ app/
│  │  ├─ tests/
│  │  ├─ Dockerfile
│  │  └─ requirements.txt
│  ├─ dashboard/
│  │  ├─ src/
│  │  ├─ Dockerfile
│  │  └─ package.json
│  └─ simulator/
│     ├─ src/
│     ├─ Dockerfile
│     └─ README.md
├─ sql/
│  ├─ clickhouse/
│  │  ├─ 001_create_tables.sql
│  │  ├─ 002_materialized_views.sql
│  │  └─ 003_seed.sql
│  └─ quality/
│     ├─ expectations.yml
│     └─ checks.sql
└─ scripts/
   ├─ dev_bootstrap.sh
   ├─ load_test.sh
   ├─ kafka_topics.sh
   └─ sample_events.jsonl
```

---

## Event schema (example)

A single telemetry event (JSON) published to Kafka topic `telemetry.events.v1`:

```json
{
  "event_id": "uuid",
  "ts": "2026-02-21T12:34:56.789Z",
  "user_id": "u_123",
  "session_id": "s_456",
  "request_id": "r_789",
  "model_id": "gpt-oss-1",
  "model_version": "2026-02-21",
  "prompt_version": "pv_12",
  "experiment": { "name": "reranker_v2", "variant": "B" },

  "latency_ms": 812,
  "tokens_in": 421,
  "tokens_out": 233,
  "status": "ok",
  "error_type": null,

  "region": "eu-central",
  "client": { "app": "web", "os": "macos" }
}
```

---

## Local development (Docker Compose)

### Prerequisites
- Docker Desktop / Docker Engine
- `make` (optional but convenient)
- Python 3.10+ (for local API dev outside containers, optional)
- Node 18+ (for local dashboard dev outside containers, optional)

### 1) Clone & configure

```bash
git clone <your-github-repo-url>
cd llm-telemetry-analytics

cp .env.example .env
# Edit .env as needed
```

**.env example**
```bash
# Kafka
KAFKA_BROKERS=kafka:9092
KAFKA_TOPIC=telemetry.events.v1

# ClickHouse
CLICKHOUSE_HOST=clickhouse
CLICKHOUSE_PORT=8123
CLICKHOUSE_USER=default
CLICKHOUSE_PASSWORD=
CLICKHOUSE_DB=telemetry

# API
API_PORT=8000

# Simulator
SIM_QPS=10
SIM_USERS=100
```

### 2) Start everything

```bash
docker compose up -d --build
```

Services you should see:
- Kafka + ZooKeeper (or KRaft, depending on compose)
- ClickHouse
- Prometheus
- Grafana
- Collector
- Stream processor
- API
- Dashboard
- Simulator (optional)

### 3) Initialize Kafka topics and ClickHouse tables

If you have scripts:

```bash
./scripts/kafka_topics.sh
cat ./sql/clickhouse/001_create_tables.sql | docker exec -i clickhouse clickhouse-client
cat ./sql/clickhouse/002_materialized_views.sql | docker exec -i clickhouse clickhouse-client
```

If you don’t have those scripts yet, create:
- Kafka topic: `telemetry.events.v1`
- ClickHouse database: `telemetry`

### 4) Open the UI
- API: `http://localhost:8000/docs`
- Grafana: `http://localhost:3000` (default user/pass often `admin/admin` unless set)
- Prometheus: `http://localhost:9090`
- Dashboard: `http://localhost:5173` (if using Vite) or `http://localhost:8080` (if served)

### 5) Generate traffic
Run simulator container (if not always-on):

```bash
docker compose up -d simulator
```

or run locally:

```bash
python -m services.simulator --qps 10 --users 100
```

### 6) Verify data is flowing

**Kafka**
```bash
docker exec -it kafka kafka-console-consumer.sh   --bootstrap-server kafka:9092   --topic telemetry.events.v1   --from-beginning --max-messages 5
```

**ClickHouse**
```bash
docker exec -it clickhouse clickhouse-client
SHOW DATABASES;
USE telemetry;
SHOW TABLES;
SELECT count() FROM events_raw;
SELECT toStartOfMinute(ts) AS m, quantile(0.95)(latency_ms) AS p95
FROM events_raw
GROUP BY m
ORDER BY m DESC
LIMIT 10;
```

---

## Suggested ClickHouse tables (minimal)

### Raw events table
Use MergeTree partitioning by date, order by timestamp + request_id.

```sql
CREATE DATABASE IF NOT EXISTS telemetry;

CREATE TABLE IF NOT EXISTS telemetry.events_raw
(
  event_id UUID,
  ts DateTime64(3, 'UTC'),
  user_id String,
  session_id String,
  request_id String,
  model_id String,
  model_version String,
  prompt_version String,
  experiment_name String,
  experiment_variant String,

  latency_ms UInt32,
  tokens_in UInt32,
  tokens_out UInt32,
  status LowCardinality(String),
  error_type LowCardinality(Nullable(String)),

  region LowCardinality(String),
  client_app LowCardinality(String),
  client_os LowCardinality(String)
)
ENGINE = MergeTree
PARTITION BY toDate(ts)
ORDER BY (toDate(ts), ts, request_id)
SETTINGS index_granularity = 8192;
```

### Aggregations (materialized view)
Example: per-minute metrics by model/version.

```sql
CREATE TABLE IF NOT EXISTS telemetry.metrics_minute
(
  minute DateTime('UTC'),
  model_id String,
  model_version String,

  requests UInt64,
  errors UInt64,
  p50_latency_ms Float64,
  p95_latency_ms Float64,
  tokens_in_sum UInt64,
  tokens_out_sum UInt64
)
ENGINE = SummingMergeTree
PARTITION BY toDate(minute)
ORDER BY (minute, model_id, model_version);

CREATE MATERIALIZED VIEW IF NOT EXISTS telemetry.mv_metrics_minute
TO telemetry.metrics_minute
AS
SELECT
  toStartOfMinute(ts) AS minute,
  model_id,
  model_version,
  count() AS requests,
  sum(status != 'ok') AS errors,
  quantile(0.50)(latency_ms) AS p50_latency_ms,
  quantile(0.95)(latency_ms) AS p95_latency_ms,
  sum(tokens_in) AS tokens_in_sum,
  sum(tokens_out) AS tokens_out_sum
FROM telemetry.events_raw
GROUP BY minute, model_id, model_version;
```

---

## Data quality & validation

You want to fail fast on bad telemetry, but still keep raw data for forensic debugging.

### Recommended checks
- Required fields present: `ts`, `request_id`, `model_id`, `latency_ms`
- Valid ranges:
  - latency_ms >= 0 and <= 120000
  - tokens_in/out <= 200000 (or your model max)
- Cardinality controls:
  - sanitize high-cardinality labels (avoid putting prompts into tags/labels)
- Schema evolution:
  - version your Kafka topic (`telemetry.events.v1`, `v2`…)
  - keep backward compatible changes whenever possible

### Where to enforce
- Collector: reject malformed payloads (HTTP 400) and emit counter metrics.
- Processor: route invalid events to a **dead-letter topic** `telemetry.dlq.v1`.
- Warehouse: constraints as “soft checks” + anomaly dashboards/alerts.

---

## API service (FastAPI) endpoints (example)

- `GET /health`
- `GET /metrics` (Prometheus scrape)
- `GET /v1/metrics/minute?model_id=...&from=...&to=...`
- `GET /v1/errors/top?from=...&to=...`
- `GET /v1/retention/d1?cohort_date=...`

---

## Observability (Prometheus + Grafana)

### What to instrument
- Collector:
  - request QPS, 4xx/5xx, payload bytes, validation failures
- Kafka:
  - broker health, partitions under-replicated
- Processor:
  - consumer lag, checkpoint duration (Flink), job restarts
- ClickHouse:
  - inserts/sec, query latency, disk usage, parts
- API:
  - request latency, error rate, cache hit rate

### Recommended alerts (starter)
- Kafka consumer lag > threshold for 5m
- Error rate > X% for 10m
- p95 latency > target for 10m
- ClickHouse disk > 80%
- Processor restart loop

---

## Cloud deployment

You have two practical options. Start with **VM-first** unless you have a strong need for Kubernetes.

### Option A (recommended): VM-first on a single host + optional worker
**Use when:** you want to ship fast, keep costs reasonable, and still be production-like.

**Minimum**
- 1 VM: Kafka + ClickHouse + API + Collector + Prometheus/Grafana
- Optional 2nd VM: Stream processor worker (or separate storage)

**Pros**
- Simple ops, easy debugging
- Lower cost and fewer moving parts
- Still demonstrates real infra patterns

**Cons**
- Less “cloud native”
- You need to be careful with disk/IO sizing

#### Suggested sizing (starter)
- 4–8 vCPU, 16–32GB RAM, NVMe disk 200GB+
- If you expect millions of events/day, ClickHouse likes disk and memory.

### Option B: Kubernetes (optional)
**Use when:** you want autoscaling, multi-node resilience, or you’re already comfortable operating k8s.

This repo supports k8s via `infra/k8s/` (optional). If you don’t need it, ignore it.

---

## Cloud setup: VM-first (step-by-step)

The steps below assume a Linux VM (Ubuntu 22.04 or similar). Works similarly on Hetzner, AWS EC2, GCP, etc.

### 1) Provision a VM
- Ubuntu 22.04 LTS
- Open ports (or put behind a reverse proxy):
  - 22 (SSH)
  - 80/443 (optional reverse proxy)
  - 3000 (Grafana), 9090 (Prometheus), 8123 (ClickHouse HTTP), 8000 (API)
- Attach storage if needed (ClickHouse likes fast disks)

### 2) Install Docker
```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo   "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu   $(. /etc/os-release && echo $VERSION_CODENAME) stable" |   sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker
```

### 3) Deploy
```bash
git clone <your-github-repo-url>
cd llm-telemetry-analytics
cp .env.example .env
# Set public hostnames, passwords, and any cloud storage credentials

docker compose up -d --build
```

### 4) Put behind a reverse proxy (recommended)
Use Caddy or Nginx to terminate TLS and route:
- `/api` → FastAPI
- `/grafana` → Grafana (optional)
- `/` → dashboard

If you want a simple approach, add a `caddy` service and a `Caddyfile` in the repo.

### 5) Persistent volumes
On a VM, use bind mounts:
- ClickHouse: `/var/lib/clickhouse`
- Grafana: `/var/lib/grafana`
- Prometheus: `/prometheus`

Make sure these directories are on your fast disk.

---

## Optional: object storage for raw events

### Local: MinIO
- Add MinIO to compose
- Processor writes Parquet to `s3://raw-events/...`

### Cloud: S3 / compatible
- Configure processor with `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and bucket name
- Partition by date and model_id for efficient scans

Example layout:
```
s3://llm-telemetry-raw/year=2026/month=02/day=21/model_id=gpt-oss-1/part-0000.parquet
```

---

## Scaling notes (practical)

If you’re targeting ~**1000 queries/day** (and 1 event per query), this system is comfortably within a single VM.

If you’re targeting **millions of events/day**:
- Use multiple Kafka partitions (e.g., 12–48 depending on throughput)
- Move ClickHouse to its own VM with larger disk
- Run Flink/Spark on dedicated workers
- Consider batching inserts into ClickHouse (avoid row-by-row inserts)
- Store raw events to S3 for cheap retention, keep aggregates in ClickHouse

---

## GitHub setup

### 1) Create the repo
- Create a new repo on GitHub (public or private)
- Push code:

```bash
git init
git add .
git commit -m "Initial commit: telemetry analytics platform"
git branch -M main
git remote add origin <your-github-repo-url>
git push -u origin main
```

### 2) Recommended GitHub Actions (CI)
Add `.github/workflows/ci.yml` to run:
- lint/format (Go + Python + TS)
- unit tests (API + processor logic)
- build Docker images

### 3) Release checklist
- Ensure `.env` is never committed
- Add `CHANGELOG.md` if you want versioning
- Add architecture diagrams and dashboard screenshots under `docs/`

---

## Troubleshooting (common)

### Kafka topic exists but no data in ClickHouse
- Confirm collector publishes to the right brokers/topic
- Confirm processor consumer group is running and has no auth/network errors
- Check processor logs for schema parsing failures
- Query ClickHouse raw table count

### ClickHouse inserts are slow
- Batch inserts (e.g., 5k–50k rows per insert)
- Avoid too many small parts (tune insert settings)
- Ensure disk isn’t saturated

### Grafana shows “no data”
- Confirm Prometheus targets are UP
- Confirm scrape endpoints are reachable from Prometheus container
- Confirm dashboards use correct datasource name

---

## Make targets (optional)

If you keep a Makefile, these targets are handy:

```bash
make up           # docker compose up -d --build
make down         # docker compose down -v
make logs         # docker compose logs -f --tail=200
make init         # create topics + clickhouse tables
make seed         # seed demo data
make test         # run tests
```

---

## Roadmap ideas (portfolio upgrades)
- Add a **DLQ topic** + replay tool
- Add **schema registry** (Avro/Protobuf) with versioning
- Add **SLO dashboards** (p95 latency, error budget burn)
- Add **retention cohorts** and “feature adoption funnel”
- Add **experiment analysis** endpoint (variant comparison)
- Add **backfill job** from Parquet → ClickHouse aggregates

---

## License
MIT (or your preference).

---

## Contact / Notes
This project is intended as a portfolio-quality reference implementation. If you adapt it for production, review security, auth, rate limiting, and data retention policies.
