# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Stack

- **Apache Airflow 3.2.0** (CeleryExecutor) — pipeline orchestration
- **MLflow 2.19.0** — experiment tracking and artifact storage
- **PostgreSQL 16** — two separate databases: one for Airflow, one for MLflow
- **Redis 7.2** — Celery broker for Airflow

## Key Commands

```bash
# Start the full platform
docker compose up -d

# Start with Flower (Celery monitor) UI on :5555
docker compose --profile flower up -d

# Run Airflow CLI commands (starts a one-off bash container)
docker compose --profile debug run --rm airflow-cli airflow dags list

# View logs for a specific service
docker compose logs -f mlflow-server
docker compose logs -f airflow-apiserver

# Stop everything
docker compose down

# Stop and remove volumes (full reset)
docker compose down -v

# Rebuild after changes (e.g. after editing Dockerfile.mlflow)
docker compose up -d --build
```

## Environment Setup

Copy `.env` and fill in the required secrets before starting:

```bash
# Generate FERNET_KEY
python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"

# Generate JWT secret
openssl rand -hex 32
```

Required `.env` variables that must not be blank: `FERNET_KEY`, `AIRFLOW__API_AUTH__JWT_SECRET`.

All other variables have working defaults in `.env` (DB credentials, admin user `airflow/airflow`, etc.) and can be left as-is for local development.

To install extra Python packages into Airflow containers without rebuilding, set `_PIP_ADDITIONAL_REQUIREMENTS` in `.env`:

```
_PIP_ADDITIONAL_REQUIREMENTS=apache-airflow-providers-docker==3.9.0 scikit-learn
```

## Architecture

The platform is **infrastructure-only** — it does not contain the actual ML or ETL code. External projects are:
1. Built into Docker images via their own CI/CD (GitHub Actions)
2. Pushed to an image registry (e.g., GHCR)
3. Executed by Airflow via `DockerOperator`

```
External GitHub repos → CI/CD → Docker Registry → Airflow (DockerOperator) → MLflow (tracking)
```

Typical DAG pattern for running an external project:

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

## Service Ports

| Service       | Port  |
|---------------|-------|
| Airflow UI    | 8080  |
| MLflow UI     | 5000  |
| Flower (opt)  | 5555  |

## Airflow v3 Notes

This uses Airflow 3.x, which differs from 2.x:
- `api-server` replaces the old `webserver` command
- `dag-processor` is a separate service (new in v3)
- Health endpoint is `/api/v2/monitor/health` (not `/health`)
- Execution API URL is configured via `AIRFLOW__CORE__EXECUTION_API_SERVER_URL`

**Startup order:** `airflow-init` (one-shot migration + user creation) must complete before all other Airflow services start. `airflow-worker` additionally waits for `airflow-apiserver` to be healthy.

## Directory Layout

- `dags/` — Airflow DAG files (mounted into all Airflow containers)
- `config/airflow.cfg` — Airflow configuration (mounted at `/opt/airflow/config/airflow.cfg`)
- `plugins/` — Airflow plugins
- `logs/` — Airflow task logs (gitignored)
- `mlartifacts/` — MLflow artifact storage (local filesystem, mounted into `mlflow-server`)
- `Dockerfile.mlflow` — extends `ghcr.io/mlflow/mlflow:v2.19.0` to add `psycopg2-binary` for PostgreSQL backend

## MLflow Integration from DAGs

```python
import mlflow

mlflow.set_tracking_uri("http://mlflow-server:5000")  # use service name inside Docker network

with mlflow.start_run():
    mlflow.log_param("key", "value")
```

The `MLFLOW_TRACKING_URI=http://mlflow-server:5000` env var is injected into all Airflow containers automatically.
