# Data & ML Platform

Personal infrastructure platform for orchestrating data pipelines and tracking ML experiments. All actual ETL/ML code lives in separate repositories and is executed here as versioned Docker images.

---

## Stack

| Service | Version | Role |
|---|---|---|
| Apache Airflow | 3.2.0 | Pipeline orchestration (CeleryExecutor) |
| MLflow | 2.19.0 | Experiment tracking & artifact storage |
| PostgreSQL | 16 | Two isolated DBs: one for Airflow, one for MLflow |
| Redis | 7.2 | Celery broker |

---

## Architecture

This platform is **infrastructure-only**. External projects are built into Docker images via their own CI/CD and executed here by Airflow.

```
External GitHub repos
        ↓
CI/CD (GitHub Actions) → Docker image registry (GHCR)
        ↓
Airflow — DockerOperator pulls & runs the image
        ↓
MLflow — receives experiment logs from inside the container
```

---

## Services

```
airflow-postgres     PostgreSQL 16 — Airflow metadata DB
redis                Redis 7.2     — Celery broker
airflow-init         One-shot container: DB migrations + admin user creation
airflow-apiserver    Airflow API server + UI  →  :8080
airflow-scheduler    DAG scheduling
airflow-dag-processor DAG file parsing (new in Airflow v3)
airflow-worker       Celery worker — runs tasks
airflow-triggerer    Async task triggering

mlflow-postgres      PostgreSQL 16 — MLflow backend store
mlflow-server        MLflow tracking server  →  :5000

flower (--profile flower)   Celery monitor UI  →  :5555
airflow-cli (--profile debug) One-off CLI container
```

---

## Directory Layout

```
data-ml-platform/
├── docker-compose.yaml       # Full stack definition
├── Dockerfile.mlflow         # Extends mlflow image with psycopg2-binary
├── .env                      # Secrets & config (copy from .env, fill secrets)
├── dags/                     # Airflow DAG files (mounted into all AF containers)
├── config/
│   └── airflow.cfg           # Airflow config (mounted into containers)
├── plugins/                  # Airflow plugins
├── logs/                     # Airflow task logs (gitignored)
└── mlartifacts/              # MLflow artifact storage (gitignored)
```

---

## Getting Started

### 1. Clone

```bash
git clone https://github.com/StebanJrb/data-ml-platform.git
cd data-ml-platform
```

### 2. Configure secrets

Copy `.env` and fill in the two required secrets:

```bash
# FERNET_KEY — encrypts Airflow connections
python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"

# AIRFLOW__API_AUTH__JWT_SECRET — signs API tokens
openssl rand -hex 32
```

All other variables in `.env` have working defaults for local development.

### 3. Start the platform

```bash
docker compose up -d
```

First start takes longer — `airflow-init` runs DB migrations before the other services come up.

### 4. Access

| UI | URL | Default credentials |
|---|---|---|
| Airflow | http://localhost:8080 | `airflow` / `airflow` |
| MLflow | http://localhost:5000 | — |
| Flower (optional) | http://localhost:5555 | — |

---

## Common Commands

```bash
# Start with Flower (Celery monitor)
docker compose --profile flower up -d

# Run Airflow CLI commands
docker compose --profile debug run --rm airflow-cli airflow dags list

# Follow logs for a specific service
docker compose logs -f mlflow-server
docker compose logs -f airflow-apiserver

# Stop everything
docker compose down

# Full reset (removes volumes/data)
docker compose down -v

# Rebuild after Dockerfile changes
docker compose up -d --build
```

---

## Adding Extra Python Packages (without rebuilding)

Set `_PIP_ADDITIONAL_REQUIREMENTS` in `.env` before starting:

```
_PIP_ADDITIONAL_REQUIREMENTS=apache-airflow-providers-docker==3.9.0 scikit-learn
```

---

## Integrating External Projects

### DAG pattern

```python
from airflow.providers.docker.operators.docker import DockerOperator

DockerOperator(
    task_id="train_model",
    image="ghcr.io/your-user/ml-project:1.0.0",
    command="python train.py",
    environment={"MLFLOW_TRACKING_URI": "http://mlflow-server:5000"},
    network_mode="data-ml-platform_default",  # join the compose network
)
```

The `MLFLOW_TRACKING_URI=http://mlflow-server:5000` env var is already injected into all Airflow containers, so workers can reach MLflow without extra config.

### Logging to MLflow from inside a container

```python
import mlflow

mlflow.set_tracking_uri("http://mlflow-server:5000")  # service name, not localhost

with mlflow.start_run():
    mlflow.log_param("learning_rate", 0.01)
    mlflow.log_metric("accuracy", 0.95)
```

---

## Airflow v3 Notes

Airflow 3.x introduces breaking changes vs 2.x:

- `api-server` replaces the old `webserver`
- `dag-processor` is a new dedicated service for parsing DAG files
- Health endpoint is `/api/v2/monitor/health` (not `/health`)
- Workers need `AIRFLOW__CORE__EXECUTION_API_SERVER_URL` pointing to the API server

---

## Roadmap

- [ ] Example DAG using DockerOperator
- [ ] Sample ML project (external repo) integrated end-to-end
- [ ] Remote artifact storage (S3/GCS) for MLflow
- [ ] CI/CD for this infrastructure repo
