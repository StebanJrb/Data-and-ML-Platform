# 🚀 Data ML Platform

A modular **Data & MLOps platform** designed to orchestrate data pipelines and track machine learning experiments using containerized services.

This project integrates:

* **Apache Airflow** for orchestration
* **MLflow** for experiment tracking
* **Docker & Docker Compose** for reproducible environments

---

## 🧠 Architecture Overview

This platform follows a **decoupled architecture**:

* Each project (ETL / ML) lives in its own repository
* CI/CD pipelines build **versioned Docker images**
* Airflow orchestrates execution of those images
* MLflow tracks experiments and results

```
GitHub Repo (projects)
        ↓
CI/CD (GitHub Actions)
        ↓
Docker Image Registry
        ↓
Airflow (orchestration)
        ↓
MLflow (experiment tracking)
```

---

## 📦 Project Structure

```
data-ml-platform/
│
├── docker-compose.yml        # Main orchestration file
├── .env                      # Environment variables
│
├── airflow/
│   └── dags/                # Airflow DAGs
│
├── mlflow/                  # MLflow configuration & data
│
└── README.md
```

---

## ⚙️ Services

### 🔄 Airflow

* Orchestrates pipelines using DAGs
* Executes containerized jobs
* Handles scheduling and dependencies

### 🧪 MLflow

* Tracks experiments
* Stores metrics, parameters, and artifacts
* Provides a UI for model comparison

---

## 🚀 Getting Started

### 1. Clone the repository

```bash
git clone https://github.com/your-username/data-ml-platform.git
cd data-ml-platform
```

---

### 2. Start the platform

```bash
docker compose up --build
```

---

### 3. Access services

* Airflow UI → http://localhost:8080
* MLflow UI → http://localhost:5000

---

## 🔗 How Projects Integrate

External projects are **not executed directly from GitHub**.

Instead:

1. Each project builds a Docker image via CI/CD
2. The image is pushed to a registry (e.g., GHCR)
3. Airflow executes the image using a DAG

Example:

```python
DockerOperator(
    task_id="train_model",
    image="ghcr.io/your-user/ml-project:1.0.0",
    command="python train.py"
)
```

---

## 📊 MLflow Integration

Inside your ML projects:

```python
import mlflow

mlflow.set_tracking_uri("http://mlflow:5000")

with mlflow.start_run():
    mlflow.log_param("model", "example")
```

---

## 🧩 Design Principles

* **Decoupled services** (orchestration ≠ execution)
* **Reproducibility via Docker images**
* **Versioned pipelines**
* **Scalable architecture (Kubernetes-ready)**

---

## 📌 Roadmap

* [ ] Add example DAGs
* [ ] Integrate DockerOperator pipelines
* [ ] Add sample ML project
* [ ] CI/CD for infrastructure
* [ ] Kubernetes deployment

---

## 🤝 Contributing

This is a personal portfolio project, but suggestions and ideas are welcome.
