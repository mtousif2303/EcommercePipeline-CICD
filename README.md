# EcommercePipeline-CICD

A containerised **Streamlit** data-visualisation app that fetches live e-commerce product and seller data, packaged with Docker and deployed to a local Kubernetes cluster via Minikube.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [CI/CD Flow](#cicd-flow)
- [Data Flow](#data-flow)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Kubernetes Deployment](#kubernetes-deployment)
- [Configuration Reference](#configuration-reference)

---

## Overview

| Layer | Technology |
|---|---|
| Application | Python 3.9 · Streamlit · Pandas · Requests |
| Containerisation | Docker (`python:3.9` base image) |
| Registry | Docker Hub (`mtousif2303/ecommce-pipeline`) |
| Orchestration | Kubernetes (Minikube) |
| External data | Fake Store API · Random User API |

---

## Architecture


<img width="1558" height="1112" alt="image" src="https://github.com/user-attachments/assets/cb88623c-c75f-463a-a468-ba1622b0c380" />


<img width="1560" height="780" alt="image" src="https://github.com/user-attachments/assets/58d563f3-6019-4b53-ab2e-2e1c4f62ff57" />



```
┌─────────────────────────────────────────────────────────────────┐
│                        External APIs                            │
│  fakestoreapi.com/products    randomuser.me/api?results=20      │
└────────────────────┬────────────────────────────────────────────┘
                     │ HTTP GET
                     ▼
          ┌──────────────────────┐
          │    Streamlit app     │  (app.py · port 8501)
          │  fetch → transform   │
          │  merge → visualise   │
          └──────────┬───────────┘
                     │ packaged by
                     ▼
          ┌──────────────────────┐        ┌──────────────────┐
          │   Docker Container   │ push → │    Docker Hub    │
          │  python:3.9 · 2.3GB  │        │  mtousif2303/    │
          └──────────────────────┘        │  ecommce-pipeline│
                                          └────────┬─────────┘
                                                   │ pull (imagePullPolicy)
                     ┌─────────────────────────────▼──────────────────────────┐
                     │              Kubernetes Cluster (Minikube)              │
                     │                                                          │
                     │  ┌──────────────────────┐   ┌──────────────────────┐   │
                     │  │ Deployment: ecomm     │   │  Service: myapp      │   │
                     │  │  replicas: 2          │   │  type: LoadBalancer  │   │
                     │  │  ┌────────┐ ┌──────┐  │   │  port: 8501          │   │
                     │  │  │ Pod 1  │ │Pod 2 │  │──▶│  nodePort: 30678     │   │
                     │  │  │ :8501  │ │:8501 │  │   └──────────────────────┘   │
                     │  │  └────────┘ └──────┘  │                              │
                     │  └──────────────────────┘                               │
                     └────────────────────────────────────┬─────────────────────┘
                                                          │ minikube tunnel
                                                          ▼
                                               http://127.0.0.1:<port>
                                                    (Browser)
```

---

## CI/CD Flow

```
 ┌──────────┐    ┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐    ┌────────────────┐
 │  1. Code │    │   2. Build      │    │  3. Tag & Push   │    │  4. Deploy      │    │  5. Expose     │
 │          │───▶│                 │───▶│                  │───▶│                 │───▶│                │
 │  app.py  │    │  docker build   │    │  docker tag      │    │  minikube start │    │  minikube      │
 │Dockerfile│    │  -t ecommerce-  │    │  docker push     │    │  kubectl apply  │    │  service myapp │
 │          │    │  pipeline-image │    │  → Docker Hub    │    │  -f *.yaml      │    │  → localhost   │
 └──────────┘    └─────────────────┘    └──────────────────┘    └─────────────────┘    └────────────────┘
```

---

## Data Flow

```
fakestoreapi.com  ──┐
                    ├──▶  transform_products()  ──▶  assign_sellers()  ──▶  Streamlit UI  ──▶  Browser
randomuser.me     ──┘     transform_users()
```

1. `fetch_products()` — calls Fake Store API, returns 20 products.
2. `fetch_users()` — calls Random User API, returns 20 user profiles.
3. `transform_products()` — selects `id, title, price, category, description, image`.
4. `transform_users()` — flattens nested JSON into `user_id, name, email, city, country, picture`.
5. `assign_sellers()` — randomly assigns a seller to each product via `merge()`.
6. Streamlit renders: full data table · summary stats · bar chart of prices · optional product images · optional seller cards.

---

## Project Structure

```
EcommercePipeline-CICD/
├── app.py               # Streamlit application
├── Dockerfile           # Container build instructions
├── deployment.yaml      # Kubernetes Deployment (2 replicas)
├── service.yaml         # Kubernetes Service (LoadBalancer · port 8501)
└── README.md
```

---

## Prerequisites

| Tool | Version used | Install |
|---|---|---|
| Python | 3.9+ | https://python.org |
| Docker Desktop | latest | https://docker.com |
| Minikube | v1.38.1 | https://minikube.sigs.k8s.io |
| kubectl | bundled with Minikube | — |

---

## Quick Start

### 1 — Run locally (no Docker)

```bash
pip install streamlit pandas requests
streamlit run app.py
# App available at http://localhost:8501
```

### 2 — Run with Docker

```bash
# Build the image
docker build -t ecommerce-pipeline-image .

# Run the container
docker run -p 8501:8501 ecommerce-pipeline-image

# App available at http://localhost:8501
```

---

## Kubernetes Deployment

### Step 1 — Tag and push to Docker Hub

```bash
docker tag ecommerce-pipeline-image:latest mtousif2303/ecommce-pipeline:latest
docker push mtousif2303/ecommce-pipeline:latest
```

### Step 2 — Start Minikube

```bash
minikube start
```

### Step 3 — Apply manifests

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

### Step 4 — Verify the deployment

```bash
# Check pods are running
kubectl get pods

# Check the deployment
kubectl get deployment ecomm

# Check the service
kubectl get service myapp
```

### Step 5 — Open the app in your browser

```bash
minikube service myapp
# Minikube opens a tunnel and launches the app in your default browser
```

### Useful kubectl commands

```bash
# Scale replicas up or down
kubectl scale deployment ecomm --replicas=3

# View pod logs
kubectl logs -l app=myapp

# Describe a pod for debugging
kubectl describe pod <pod-name>

# Delete everything
kubectl delete -f deployment.yaml
kubectl delete -f service.yaml
```

---

## Configuration Reference

### Dockerfile

```dockerfile
FROM python:3.9
WORKDIR /app
COPY . /app
RUN pip install streamlit pandas requests
EXPOSE 8501
CMD ["streamlit", "run", "app.py", "--server.port=8501", "--server.address=0.0.0.0"]
```

### deployment.yaml

| Field | Value |
|---|---|
| Name | `ecomm` |
| Image | `mtousif2303/ecommce-pipeline:latest` |
| Replicas | `2` |
| Container port | `8501` |
| Memory limit | `128Mi` |
| CPU limit | `500m` |

### service.yaml

| Field | Value |
|---|---|
| Name | `myapp` |
| Type | `LoadBalancer` |
| Port | `8501` |
| Target port | `8501` |
| NodePort | auto-assigned (e.g. `30678`) |

---

## Docker Hub

Image: [`mtousif2303/ecommce-pipeline`](https://hub.docker.com/r/mtousif2303/ecommce-pipeline)

```bash
docker pull mtousif2303/ecommce-pipeline:latest
```
