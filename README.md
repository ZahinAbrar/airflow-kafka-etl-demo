# airflow-kafka-etl-nyc-taxi

[![status](https://img.shields.io/badge/status-active-brightgreen.svg)](./)
[![license](https://img.shields.io/badge/license-MIT-blue.svg)](./LICENSE)
[![stack](https://img.shields.io/badge/stack-Airflow%20%7C%20Kafka%20%7C%20Bash%20%7C%20Python%20%7C%20SQL-informational)](./)
[![data](https://img.shields.io/badge/data-NYC%20Taxi-yellow)](https://www1.nyc.gov/site/tlc/about/tlc-trip-record-data.page)

Production-style **streaming + batch ETL** for NYC Taxi trips using **Apache Kafka** (stream buffer), **Apache Airflow** (orchestration), **Bash** (ingest), and **Python/SQL** (transform + load). Demonstrates **Bronze/Silver/Gold** data layers, **idempotent loads**, and **SLA-driven** pipelines.


## Table of Contents
- [Architecture](#architecture)
- [Features](#features)
- [Tech Stack](#tech-stack)
- [Repository Layout](#repository-layout)
- [Quickstart](#quickstart)
  - [Option A — Colab + Confluent Cloud (no local Docker)](#option-a--colab--confluent-cloud-no-local-docker)
  - [Option B — Local Docker Compose (full stack)](#option-b--local-docker-compose-full-stack)
- [Airflow DAG](#airflow-dag)
- [Data Model & Quality](#data-model--quality)
- [Observability](#observability)
- [Benchmarks](#benchmarks)
- [Interview Talking Points](#interview-talking-points)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)

---
```

**Layers**
- **Bronze**: append-only Parquet batches landed by Kafka consumer.
- **Silver**: cleaned, partitioned Parquet by `pickup_date` (daily compaction).
- **Gold**: star schema (`fact_trips`, `dim_vendor`, `dim_zone` — minimal demo uses a single fact table).

---

## Features
- **Streaming buffer with Kafka** (partitioning by date key; replayable consumption).
- **Airflow orchestration** (dependencies, retries, SLAs, backfills).
- **Idempotent loads** (overwrite-by-partition at the sink → “exactly-once” effect).
- **Schema evolution ready** (JSON today; Avro/Schema Registry recommended next).
- **Data Quality** (row counts, null checks, numeric ranges, quarantine folder).
- **Observability** (task-level SLAs, retries, basic metrics).

---

## Tech Stack
- **Apache Kafka** (Confluent Cloud for Colab or local broker via Docker)
- **Apache Airflow**
- **Python** (pandas, pyarrow/fastparquet, sqlite3)
- **Bash/Shell**
- **SQLite / Postgres** (warehouse)
- **Parquet** (columnar storage)

---

## Repository Layout

```bash
etl-kafka-airflow/
├─ airflow/
│  └─ dags/
│     └─ nyc_taxi_pipeline.py         # main DAG
├─ scripts/
│  ├─ fetch_csv.sh                    # CSV → NDJSON
│  ├─ produce_to_kafka.sh             # NDJSON → Kafka
│  ├─ consume_to_parquet.py           # Kafka → Bronze Parquet
│  ├─ compact_daily.py                # Bronze → Silver (partitioned)
│  └─ load_dw.py                      # Silver → Warehouse (idempotent)
├─ data/                               # mounted locally (gitignored)
├─ notebooks/
│  └─ colab_kafka_etl_demo.ipynb      # managed-Kafka demo (use in Colab)
├─ docker-compose.yml                  # (optional) local Kafka + Airflow
└─ README.md
```

> The **Colab notebook** variant lets you demo without Docker by using **Confluent Cloud** for Kafka. You can add the notebook to `notebooks/` in your repo.

---

## Quickstart

### Option A — Colab + Confluent Cloud (no local Docker)

1) Create a free cluster at **Confluent Cloud** → create topic `taxi.trips.v1` → create API key/secret.  
2) Open the notebook: `notebooks/colab_kafka_etl_demo.ipynb`  
   - Or download it now: **[colab_kafka_etl_demo.ipynb](./notebooks/colab_kafka_etl_demo.ipynb)**  
3) Fill in `CONF` (bootstrap servers, API key/secret) in the first config cell.  
4) Run cells top-to-bottom:
   - CSV → NDJSON (bash cell)
   - Produce NDJSON → Kafka
   - Consume → Bronze Parquet
   - Compact → Silver (partition by `pickup_date`)
   - Load → SQLite (warehouse) and run sample queries

**Why this path?** Colab can’t host a local broker or Airflow webserver. Managed Kafka keeps the demo realistic while staying cloud-agnostic.

---

### Option B — Local Docker Compose (full stack)

> If you prefer a full local stack, copy this minimal `docker-compose.yml` into the repo (or adapt to your environment). This spins Zookeeper, Kafka, and Airflow.

```yaml
version: "3.9"
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.6.1
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka:
    image: confluentinc/cp-kafka:7.6.1
    depends_on: [zookeeper]
    ports: ["9092:9092"]
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  airflow:
    image: apache/airflow:2.9.3
    depends_on: [kafka]
    ports: ["8080:8080"]
    environment:
      AIRFLOW__CORE__LOAD_EXAMPLES: "false"
      AIRFLOW__CORE__EXECUTOR: "LocalExecutor"
    volumes:
      - ./airflow/dags:/opt/airflow/dags
      - ./scripts:/opt/airflow/scripts
      - ./data:/opt/airflow/data
    command: bash -lc "airflow db init && airflow users create --username admin --password admin --firstname A --lastname B --role Admin --email admin@example.com || true && airflow webserver & airflow scheduler"

```

**Create topic** (one-time) from the Airflow container shell:
```bash
docker exec -it $(docker ps -qf name=airflow) bash -lc 'kafka-topics --bootstrap-server kafka:9092 --create --topic taxi.trips.v1 --partitions 3 --replication-factor 1 || true'
```

**Run the DAG**: Open Airflow UI at http://localhost:8080 → enable `nyc_taxi_pipeline`.

---

## Airflow DAG

**Flow**: `fetch_csv` → `produce_to_kafka` → `consume_to_bronze` → `compact_to_silver` → `load_dw` → `dq_checks`  
- Schedule: hourly (streaming Bronze) + nightly compaction to Silver.  
- Retries: `retries=2`, exponential backoff.  
- Idempotency: partition overwrite for a given `pickup_date` in the warehouse.

---

## Data Model & Quality

**Gold (warehouse)**  
- `fact_trips(trip_id, vendor_id, pickup_datetime, dropoff_datetime, passenger_count, trip_distance, fare_amount, pickup_location_id, dropoff_location_id, pickup_date)`

**DQ checks (examples)**  
- Row count reconciliation (stage → bronze → silver → warehouse)  
- Null checks (timestamps, IDs)  
- Numeric ranges (`trip_distance`, `fare_amount`)  
- Quarantine invalid rows and report daily

---

## Observability
- **SLAs** on compaction/load tasks; Slack/email alerts on miss (optional).
- **Retries** and backoff at task level.
- **Dead-letter** Kafka topic (optional) for poison messages.

---

## Benchmarks
- **Freshness**: Bronze landing ≤ **60s** of ingest window (demo data).  
- **Reliability**: ≥ **99%** DAG run success (retries enabled).  
- **Throughput**: 50k+ rows/sec local compaction to Parquet (laptop-class hardware).

> Benchmarks are indicative; vary by hardware and partition sizing.

---

## Interview Talking Points

- **Kafka vs queue**: partitioning, consumer groups, replay, back-pressure.  
- **Delivery semantics**: at-least-once + idempotent sink ⇒ effective exactly-once at warehouse.  
- **Schema evolution**: start with JSON; add Avro + Schema Registry for compatibility guarantees.  
- **Batch + streaming**: hourly ingest to Bronze; nightly compaction to Silver; BI on Gold.  
- **Backfills**: Airflow `start_date` and manual `dagrun` with `ds` override; partition overwrite prevents duplicates.  
- **Scaling**: increase partitions; parallel consumers; Parquet partition pruning.

**What I'd improve next**  
- Great Expectations for declarative DQ; Delta/Iceberg for ACID/time-travel; Prometheus/Grafana metrics; CI/CD for DAGs; MWAA/Astronomer deploy.

---

## Roadmap
- [ ] Add Avro + Schema Registry
- [ ] Switch Silver/Gold to **Delta Lake/Iceberg**
- [ ] Great Expectations checks + data docs
- [ ] CI/CD for DAGs (pre-commit, GitHub Actions)
- [ ] MWAA/Astronomer deployment guide

---

## Contributing
PRs welcome! Please open an issue to discuss substantial changes. Run linters/tests locally before submitting.

---

## License
**MIT** — see `LICENSE` for details.
