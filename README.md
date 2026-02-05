# 🏯 Sovereign RAG Stack
## Enterprise-Grade, Self-Hosted AI Infrastructure

<p align="center">
  <strong>The Complete AI Platform for Air-Gapped Deployments</strong><br>
  <em>Vector Search + Local LLM + Workflow Automation in One Stack</em>
</p>

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [The Stack](#the-stack)
4. [Quick Start](#quick-start)
5. [Service Deep Dive](#service-deep-dive)
6. [Configuration](#configuration)
7. [Production Deployment](#production-deployment)
8. [Troubleshooting](#troubleshooting)

---

## Overview

This is **not** a simple chatbot script. This is a **full-topology microservices architecture** designed to run an entire AI department on-premise or in an air-gapped environment.

### Why "Sovereign"?

| Principle | Guarantee |
|-----------|-----------|
| 🔒 **Zero Data Leakage** | Your documents never leave this cluster |
| 🔄 **Zero Vendor Lock-in** | Swap Llama 3 → Deepseek R1 in one config line |
| 📜 **Full Auditability** | Every interaction logged in your local Postgres |
| 💰 **Zero API Costs** | Run inference on your own GPU |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SOVEREIGN RAG ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│   ┌─────────────┐                              ┌──────────────────┐  │
│   │   Client    │                              │     Qdrant       │  │
│   │ (Dashboard) │────────────────────────────►│  (Vector Brain)  │  │
│   └──────┬──────┘                              │   768 dims       │  │
│          │                                     └────────┬─────────┘  │
│          │ [Chat]                                       │            │
│          ▼                                              │ [Search]   │
│   ┌─────────────┐              ┌─────────────┐         │            │
│   │    n8n      │◄─────────────│  Ingestion  │─────────┘            │
│   │  Workflow   │              │    API      │                       │
│   │   Engine    │              │  (FastAPI)  │                       │
│   └──────┬──────┘              └─────────────┘                       │
│          │                                                           │
│          │ [Inference]                                               │
│          ▼                                                           │
│   ┌─────────────┐              ┌─────────────┐                       │
│   │   Ollama    │              │  Postgres   │                       │
│   │  (Local LLM)│              │   (Logs)    │                       │
│   │ Llama/Mistral│              │             │                       │
│   └─────────────┘              └─────────────┘                       │
│                                                                       │
│   Network: sovereign-net (bridge)                                    │
│   GPU: NVIDIA (optional)                                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Data Flow

```
   [Document Upload]                    [User Question]
          │                                    │
          ▼                                    ▼
   ┌─────────────┐                    ┌─────────────┐
   │ Ingestion   │                    │    n8n      │
   │    API      │                    │  Workflow   │
   └──────┬──────┘                    └──────┬──────┘
          │                                  │
          │ [Chunk + Embed]                  │ [Embed Query]
          ▼                                  ▼
   ┌─────────────────────────────────────────┐
   │              QDRANT                      │
   │         Vector Database                  │
   └─────────────────┬───────────────────────┘
                     │
                     │ [Top 5 Similar Chunks]
                     ▼
              ┌─────────────┐
              │   Ollama    │
              │    LLM      │
              └──────┬──────┘
                     │
                     │ [Generated Answer]
                     ▼
              ┌─────────────┐
              │   Client    │
              └─────────────┘
```

---

## The Stack

| Service | Technology | Role | Port |
|---------|------------|------|------|
| 🧠 **Brain** | Qdrant | High-performance Vector Store (Rust) | 6333 |
| ⚙️ **Logic** | n8n | Low-code workflow orchestration | 5678 |
| 🤖 **Engine** | Ollama | Local LLM inference (Llama 3, Mistral) | 11434 |
| 📥 **Ingest** | FastAPI (Python) | Document parsing & chunking | 8000 |
| 🖥️ **UI** | Next.js | Web dashboard | 3000 |
| 🗄️ **Database** | PostgreSQL 16 | System state & audit logs | 5432 |

---

## Quick Start

### Prerequisites

- Docker & Docker Compose v2+
- NVIDIA GPU (optional, for acceleration)
- 16GB+ RAM recommended

### One-Command Deployment

```bash
# Clone the repository
git clone https://github.com/tahaex/sovereign-rag-stack.git
cd sovereign-rag-stack

# Start all services
docker-compose up -d

# Check status
docker-compose ps
```

### Verify Services

```bash
# Qdrant health
curl http://localhost:6333/readyz

# n8n UI
open http://localhost:5678

# Pull a model for Ollama
docker exec -it netics-ollama ollama pull llama3
```

---

## Service Deep Dive

### 1. Qdrant (Vector Database)

```yaml
qdrant:
  image: qdrant/qdrant:latest
  container_name: netics-qdrant
  ports:
    - "6333:6333"
  volumes:
    - qdrant_data:/qdrant/storage
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:6333/readyz"]
    interval: 30s
    retries: 3
```

**Key Features:**
- Written in Rust for maximum performance
- Supports filtering during vector search
- Built-in snapshots for backup

**Create a Collection:**

```bash
curl -X PUT "http://localhost:6333/collections/documents" \
  -H "Content-Type: application/json" \
  -d '{
    "vectors": {
      "size": 768,
      "distance": "Cosine"
    }
  }'
```

### 2. Ollama (Local LLM)

```yaml
ollama:
  image: ollama/ollama:latest
  container_name: netics-ollama
  volumes:
    - ollama_models:/root/.ollama
  deploy:
    resources:
      reservations:
        devices:
          - driver: nvidia
            count: 1
            capabilities: [gpu]
```

**GPU Passthrough:** Enabled via NVIDIA Container Toolkit.

**Available Models:**

| Model | Size | Use Case |
|-------|------|----------|
| `llama3` | 8B | General chat |
| `mistral` | 7B | Fast inference |
| `deepseek-coder` | 7B | Code generation |
| `nomic-embed-text` | 137M | Embeddings |

**Pull a Model:**

```bash
docker exec -it netics-ollama ollama pull llama3
```

### 3. n8n (Workflow Engine)

```yaml
n8n:
  image: n8nio/n8n:latest
  container_name: netics-n8n
  environment:
    - DB_TYPE=postgresdb
    - DB_POSTGRESDB_HOST=postgres
    - DB_POSTGRESDB_PORT=5432
    - WEBHOOK_URL=https://api.yourdomain.com/
  ports:
    - "5678:5678"
  depends_on:
    postgres:
      condition: service_healthy
```

**Why n8n?**
- Visual workflow builder
- Persistent execution history
- 400+ integrations

### 4. Ingestion API (FastAPI)

```yaml
ingestion-api:
  build: ./backend-api
  container_name: netics-ingestion-api
  environment:
    - QDRANT_HOST=qdrant
    - QDRANT_PORT=6333
  depends_on:
    qdrant:
      condition: service_healthy
```

**Endpoints:**

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/ingest` | Upload and process document |
| GET | `/health` | Service health check |

### 5. PostgreSQL (System Database)

```yaml
postgres:
  image: postgres:16-alpine
  container_name: netics-pg
  environment:
    - POSTGRES_USER=n8n
    - POSTGRES_PASSWORD=secure_password_change_me
  volumes:
    - pg_data:/var/lib/postgresql/data
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U n8n"]
    interval: 10s
```

---

## Configuration

### Environment Variables

| Variable | Service | Description |
|----------|---------|-------------|
| `POSTGRES_PASSWORD` | postgres | **⚠️ Change in production!** |
| `WEBHOOK_URL` | n8n | Public URL for webhooks |
| `QDRANT_HOST` | ingestion-api | Vector DB connection |

### Volumes

| Volume | Purpose | Backup? |
|--------|---------|---------|
| `qdrant_data` | Vector embeddings | ✅ Yes |
| `ollama_models` | Downloaded LLM weights | ⚠️ Optional |
| `pg_data` | n8n workflows & history | ✅ Yes |

---

## Production Deployment

### 1. Security Checklist

- [ ] Change `POSTGRES_PASSWORD`
- [ ] Enable Qdrant API key authentication
- [ ] Put services behind reverse proxy (Traefik/Caddy)
- [ ] Enable TLS/HTTPS

### 2. Reverse Proxy Example (Traefik)

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.n8n.rule=Host(`n8n.yourdomain.com`)"
  - "traefik.http.routers.n8n.tls.certresolver=letsencrypt"
```

### 3. Backup Strategy

```bash
# Qdrant snapshot
curl -X POST "http://localhost:6333/collections/documents/snapshots"

# PostgreSQL dump
docker exec netics-pg pg_dump -U n8n > backup.sql
```

---

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Ollama CUDA error | Missing NVIDIA driver | Install `nvidia-container-toolkit` |
| n8n won't start | Postgres not ready | Wait for healthcheck or restart |
| Qdrant connection refused | Wrong network | Ensure all services use `sovereign-net` |
| Slow inference | No GPU | Use smaller model or add GPU |

### Logs

```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f ollama
```

### Reset Everything

```bash
docker-compose down -v  # Warning: Deletes all data!
docker-compose up -d
```

---

## 📜 License

MIT License - Free for personal and commercial use.

---

<p align="center">
  <strong>Built by Taha E. for Netics Agency</strong><br>
  <a href="https://netics.fr">netics.fr</a>
</p>
