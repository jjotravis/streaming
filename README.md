# streaming

# PyFlink Streaming Workshop (Deployment Guide)

This document is a deployment‑oriented companion to the full workshop material found in `README (2).md`.  It distills the exercises into a set of repeatable steps and operational notes so you can run the pipeline in a development cluster and understand the patterns involved in moving to production.

> **Goal:** build and deploy a real‑time streaming pipeline that ingests NYC taxi trips from a producer, processes them with Flink, and sinks results to PostgreSQL.

```
Producer (Python) -> Kafka (Redpanda) -> Flink -> PostgreSQL
```

---

## 1. Prerequisites

- Docker & Docker Compose (or Kubernetes for production)
- `uv` for Python environment management (https://docs.astral.sh/uv/)
- A SQL client such as `pgcli`, DBeaver, DataGrip, etc.

> In production you replace `uv` with your normal Python packaging/CI workflow and run Flink/Redpanda/Postgres on dedicated hosts or managed services.

---

## 2. Broker: Redpanda (Kafka compatible)

Create `docker-compose.yml` with a single‑node Redpanda service:

```yaml
services:
  redpanda:
    image: redpandadata/redpanda:v25.3.9
    command: ... # see workshop for full args
    ports:
      - 9092:9092      # external
      - 29092:29092    # internal
      - 8082:8082      # proxy
```

The command uses separate listeners for internal (container) and external clients; adjust addresses when deploying to a cluster or cloud.  In a managed Kafka offering you can skip the Redpanda container entirely.

Start the broker:

```bash
docker compose up -d redpanda
```

Verify with `docker compose ps`.

---

## 3. Python producer & consumer

### Project setup

```bash
uv init -p 3.12
uv add kafka-python pandas pyarrow psycopg2-binary
mkdir -p src/{producers,consumers,job}
```

Shared model (`src/models.py`):

```python
from dataclasses import dataclass
import json

@dataclass
class Ride:
    PULocationID: int
    DOLocationID: int
    trip_distance: float
    total_amount: float
    tpep_pickup_datetime: int  # epoch ms


def ride_from_row(row):
    ...

def ride_deserializer(data):
    ...
```

Producer scripts send JSON‑serialized `Ride` objects to topic `rides`.  The workshop includes both a batch producer (`producer.py`) and a realtime generator (`producer_realtime.py`) that simulates late events.

Consumer examples show how to read from Kafka either to console or directly into PostgreSQL.  These are useful for debugging and for comparison with the Flink job, but they are not used in a deployed pipeline.

---

## 4. PostgreSQL for sinks

Add to `docker-compose.yml`:

```yaml
  postgres:
    image: postgres:18
    environment:
      POSTGRES_DB: postgres
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
```

Start it with:

```bash
docker compose up -d postgres
```

Create tables for raw and aggregated events (examples in workshop).  In production you will manage schemas with migrations.

---

## 5. Flink cluster and custom image

### Build image

Flink requires a custom image with Python, PyFlink, and connector JARs.  The included `Dockerfile.flink` does this:

- base: `flink:2.2.0-scala_2.12-java17`
- install Python 3.12 and `uv` packages
- download Kafka & JDBC connector JARs
- apply JVM config tweaks

Build the image and add services:

```yaml
  jobmanager:
    build:
      context: .
      dockerfile: ./Dockerfile.flink
    image: pyflink-workshop
    ports:
      - "8081:8081"  # web UI
    volumes:
      - ./:/opt/flink/usrlib
      - ./src/:/opt/src
    command: jobmanager
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager
        jobmanager.memory.process.size: 1600m

  taskmanager:
    image: pyflink-workshop
    depends_on:
      - jobmanager
    volumes:
      - ./:/opt/flink/usrlib
      - ./src/:/opt/src
    command: taskmanager --taskmanager.registration.timeout 5 min
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager
        taskmanager.memory.process.size: 1728m
        taskmanager.numberOfTaskSlots: 15
        parallelism.default: 3
```

> **Deployment note:** In production you'll typically build the image once in CI and push to a registry.  Job artifacts may be baked into the image or mounted from object storage.

Start the cluster:

```bash
docker compose up --build -d
```

Verify via `docker compose ps` and visit [http://localhost:8081](http://localhost:8081).

---

## 6. Flink jobs

Jobs are plain Python scripts submitted to the cluster.

### Pass‑through job

Reads from Kafka (`latest-offset`) and writes raw events to PostgreSQL.

`src/job/pass_through_job.py`:

```python
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.table import EnvironmentSettings, StreamTableEnvironment

# helper functions create_events_source_kafka and create_processed_events_sink_postgres

if __name__ == '__main__':
    log_processing()
```

Submit with:

```bash
docker compose exec jobmanager ./bin/flink run \
    -py /opt/src/job/pass_through_job.py \
    --pyFiles /opt/src -d
```

### Aggregation job

Performs 1‑hour tumbling windows, watermarking, and upserts into an aggregated table with a primary key.  See `src/job/aggregation_job.py`.

Submit the same way as above.

### Key configuration points

| Setting | Purpose |
|---------|---------|
| `scan.startup.mode` | `latest-offset` vs `earliest-offset` for initial startup/backfill |
| `enable_checkpointing(10_000)` | checkpoint interval in milliseconds |
| `WATERMARK ... - INTERVAL '5' SECOND` | 5‑second lateness tolerance |
| `primary key` on sink | enables upsert for late/out‑of‑order events |
| `parallelism.default` / `env.set_parallelism(3)` | control task parallelism |

> **Production tip:** store Flink configuration in `flink-conf.yaml` or via environment variables; use a state backend (RocksDB, filesystem) to persist checkpoints.

---

## 7. Operational considerations

- **Checkpointing & recovery:** Flink keeps offsets and state. Restarting the same job instance resumes from the latest checkpoint; canceling and resubmitting starts from the offset setting again.
- **Watermarks & late data:** tune the watermark interval based on your data's out‑of‑order characteristics. Use `allowed-lateness` if necessary and rely on sink upserts.
- **Scaling:** add more task managers or increase slots to support higher throughput. Monitor via the web UI.
- **Deploying jobs:** in CI you can trigger `flink run` on a cluster or build a Docker image containing the job. Managed services often expose APIs/CLIs for job submission.
- **Schema evolution:** use DDL `CREATE TABLE` with versions or migration scripts; avoid breaking changes in running jobs.
- **Monitoring:** integrate with Prometheus/Grafana, set alerts for job failures, high latencies, or checkpoint failures.

---

## 8. Cleanup

```bash
docker compose down   # stop containers
docker compose down -v  # remove volumes if needed
```

---

## Further reading

- Flink documentation: https://nightlies.apache.org/flink
- Redpanda vs Kafka: https://redpanda.com/docs
- DataTalksClub workshop repository (full examples and live code)

---

This guide is meant to get you from zero to a runnable streaming pipeline and highlight the pieces you'll need to think about when deploying Flink jobs in production.  Happy streaming! 
