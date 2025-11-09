# 🧠 AI-Models-Cluster — Architecture

## 1. Overview

AI-Models-Cluster is a Kubernetes-based platform for running multiple local AI models
behind a unified API, with built-in monitoring and logging.

The design focuses on:
- 🔁 Orchestration of multiple model services
- 📈 Observability by default (Prometheus + Grafana)
- 📦 Containerized, reproducible deployments
- 📐 Clean separation of concerns (gateway, models, infra, observability)

---

## 2. High-Level Architecture



---

## 3. Namespaces

- `ai-gateway`: API Gateway / Inference Router.
- `ai-models`: One Deployment per AI model.
- `ai-observability`: Prometheus, Grafana (and optionally Loki).
- `ai-infra`: Ingress Controller, cert-manager, shared infra.

This separation keeps the platform readable and modular.

---

## 4. Core Components

### 4.1 API Gateway

Responsibilities:
- Single entrypoint for all inference requests.
- Route traffic to the correct model service.
- Basic auth / API key validation.
- Simple rate limiting / request validation.
- Expose `/healthz` and `/metrics`.

### 4.2 Model Services

Each model runs as an independent microservice:

- One Deployment per model (e.g. `llm-gptq-service`, `embedding-service`).
- Container image includes:
  - Runtime (Python/C++/whatever)
  - Model files (mounted or baked in)
- Common HTTP contract:
  - `POST /infer`
  - `GET /healthz`
  - `GET /metrics`
- Scalable via HPA based on CPU or custom metrics.

### 4.3 Observability

- **Prometheus** scrapes:
  - API Gateway
  - Model services
  - Kubernetes components
- **Grafana**:
  - Dashboards for:
    - Latency per model
    - Request throughput
    - Error rates
    - Resource usage per pod/model

*(Details in `observability.md`.)*

### 4.4 Ingress & Networking

- Ingress Controller (NGINX/Traefik).
- TLS termination at Ingress (self-signed or real cert).
- Routes:
  - `/api` → API Gateway
  - `/grafana` → Grafana (optional, protected)
  - `/prometheus` → Prometheus (optional, protected)

---

## 5. Request Flow

1. Client sends `POST /api/infer` with:
   - `model`: model identifier
   - `payload`: input data
2. Ingress routes to `api-gateway`.
3. API Gateway:
   - Validates API key (if configured)
   - Looks up target service in model registry (ConfigMap)
   - Forwards request to selected model service
4. Model service performs inference and returns response.
5. Gateway returns unified JSON response to client.

Metrics:
- Each step exposes latency / status as Prometheus metrics.

---

## 6. Configuration & Model Registry

Models are registered via ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: model-registry
  namespace: ai-gateway
data:
  models.yaml: |
    - name: llm-gptq
      service: llm-gptq-service.ai-models.svc.cluster.local
      path: /infer
    - name: embeddings
      service: embedding-service.ai-models.svc.cluster.local
      path: /infer
```

API Gateway reads this at startup to know where to route requests.

---

## 7. Security (Basic)

- TLS at Ingress.
- API key/JWT validation in gateway.
- Namespaced resources + RBAC for Prometheus only.
- Optional:
  - NetworkPolicies to restrict access to model services.

---

## 8. Tech Stack & Design Choices

- **Kubernetes**: scheduling, scaling, service discovery.
- **Docker**: containerized models + gateway.
- **Prometheus + Grafana**: native metrics stack.
- **Helm/Kustomize**: repeatable deployments.
- **Stateless services**: each model service is replaceable & horizontally scalable.

---

## 9. Future Enhancements

- Add GPU-aware scheduling (nodeSelector / tolerations).
- Canary releases for new model versions.
- Async queue for long-running inference requests.
- Multi-tenant auth & per-tenant metrics.
