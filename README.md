# Open Science Artifact: Reproducible DNS Security Stack

This repository provides a fully reproducible, containerized security analytics stack used to evaluate LLM-assisted DNS investigation workflows. The artifact runs on **Windows, macOS, and Linux**, with optional **GPU acceleration (Linux/Windows + NVIDIA only)**.

The bring-up scripts (`up.ps1`, `up.sh`) automate environment preparation, credential validation, and service orchestration so the entire system can be launched with a single command.

---

# System Overview

The stack includes:

- **Elasticsearch + Kibana + Logstash (ELK)** for DNS telemetry storage and analysis  
- **n8n** for deterministic and agentic workflow orchestration  
- **Ollama** for local LLM inference (CPU or GPU)  
- **OpenCTI** for cyber threat intelligence ingestion and enrichment  
- **Model Context Protocol (MCP) Elasticsearch service** for governed LLM access to telemetry  

All services communicate over a dedicated Docker network (`ai-net`).

---

# Prerequisites

## Required (All Platforms)
- Docker Desktop (Docker Engine + Docker Compose v2)
- Git
- At least **16 GB RAM** (32 GB recommended)

## GPU Mode (Optional)
- Linux or Windows with NVIDIA GPU
- NVIDIA drivers installed
- NVIDIA Container Toolkit installed
- **Not supported on macOS**

---

# Starting the Stack

---

## Windows (PowerShell)

Open PowerShell in the project root.

### CPU Mode (Default)

```powershell
cd open-science
.\scripts\up.ps1
```

### GPU Mode (NVIDIA Only)

```powershell
cd open-science
$env:COMPOSE_PROFILES="gpu"
.\scripts\up.ps1
```

To switch back to CPU mode:

```powershell
Remove-Item Env:COMPOSE_PROFILES
```

---

## macOS

⚠️ GPU mode is **not supported** on macOS.

### CPU Mode

```bash
cd open-science
./scripts/up.sh
```

---

## Linux

###  CPU Mode

```bash
cd open-science
./scripts/up.sh
```

### GPU Mode (NVIDIA Only)

```bash
cd open-science
COMPOSE_PROFILES=gpu ./scripts/up.sh
```

---

# CPU vs GPU Execution Model (Ollama)

The stack supports either CPU or GPU Ollama — never both at the same time.

| Mode | Ollama Service Started | Port |
|------|------------------------|------|
| CPU (default) | `ollama` | 11434 |
| GPU (`COMPOSE_PROFILES=gpu`) | `ollama-gpu` | 11434 |

The bring-up scripts ensure:
- Only one Ollama service runs
- No port conflicts on `11434`
- No configuration changes are required when switching modes

---

## Ollama Base URL (Unchanged)

Inside Docker (n8n, agents, workflows):

```
http://ollama:11434
```

This remains stable across CPU and GPU modes.

---

# What the Bring-Up Scripts Do

Both `up.ps1` and `up.sh` perform the same logical steps:

### 1. Environment Preparation
- Validate Docker + Compose availability
- Create `ai-net` network if missing
- Create `.env` from `.env.example` if missing

### 2. Credential Safety
- Validate or generate:
  - `OPENCTI_ADMIN_EMAIL`
  - `OPENCTI_ADMIN_TOKEN` (UUID)
  - Kibana encryption keys:
    - `KIBANA_ENCRYPTION_KEY`
    - `KIBANA_REPORTING_KEY`
    - `KIBANA_SECURITY_KEY`

### 3. Logstash Guardrails
- Ensures `logstash/config/logstash.yml` exists
- Prevents file/directory mismatches

### 4. Image Pre-Pull
- Pre-pulls all Docker images with retries
- Avoids mid-start failures during evaluation

### 5. Ordered ELK Startup
- Start Elasticsearch first
- Wait for:
  - Cluster health
  - Security subsystem readiness
- Sync `kibana_system` password before starting Kibana

### 6. Main Stack Startup
- Kibana
- Logstash
- MCP Elasticsearch
- n8n (seed + runtime)
- Ollama (CPU or GPU)

### 7. OpenCTI Startup with Auto-Recovery
- Starts OpenCTI using layered compose files
- Detects RabbitMQ credential drift
- Performs one-time automatic recovery if needed

### 8. MCP Elasticsearch API Key Management
- Validates `ELK_ES_API_KEY`
- Generates a scoped API key if missing/invalid
- Recreates MCP container to apply updated credentials

### 9. Health Checks
- Verifies Kibana status endpoint
- Prints service URLs

---

# Service Endpoints

After successful startup:

| Service | URL |
|---------|-----|
| Elasticsearch | http://localhost:9200 |
| Kibana | http://localhost:5601 |
| n8n | http://localhost:5678 |
| Ollama | http://localhost:11434 |
| OpenCTI | http://localhost:8080 |

---

# Stopping the Stack

## Windows

```powershell
.\scripts\down.ps1
```

## macOS / Linux

```bash
./scripts/down.sh
```

By default, volumes are preserved (no data loss).

---

# Reproducibility Notes

- Credentials are deterministic (from `.env`) or auto-generated and persisted. Follow User Guide for additional credential instructions.
- No external APIs are required
- All LLM inference is local
- CPU and GPU modes behave identically from a workflow perspective
- The stack can be torn down and restarted without manual cleanup

---

# Troubleshooting

## Ports Already in Use

Remove any other conflicting ports prior to executing.

## GPU Mode Not Working on macOS

GPU passthrough is not supported on macOS Docker Desktop. Run without:

---

For artifact evaluation questions, refer to the accompanying paper and pdf setup guide.
