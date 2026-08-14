

<p align="center">
  <img src="https://img.shields.io/pypi/v/sollol?style=for-the-badge&color=blue" alt="PyPI">
  <img src="https://img.shields.io/github/stars/B-A-M-N/SOLLOL?style=for-the-badge&color=gold" alt="Stars">
  <img src="https://img.shields.io/github/forks/B-A-M-N/SOLLOL?style=for-the-badge&color=lightblue" alt="Forks">
  <img src="https://img.shields.io/github/license/B-A-M-N/SOLLOL?style=for-the-badge&color=green" alt="License">
  <img src="https://img.shields.io/badge/python-3.8+-blue?style=for-the-badge&logo=python" alt="Python">
  <img src="https://img.shields.io/badge/Ollama-Compatible-green?style=for-the-badge" alt="Ollama">
</p>

<h1 align="center">SOLLOL</h1>
<p align="center">
  <b>Super Ollama Load balancer & Orchestration Layer</b><br>
  <i>Intelligent routing, observability, and distributed inference for Ollama clusters.</i>
</p>

---

## What It Is

SOLLOL sits between your applications and a collection of Ollama nodes. It discovers them, monitors their health, scores them by GPU/CPU capacity and current load, then routes each request to the best available node. If a node dies, it fails over automatically.

Think of it as a drop-in replacement for talking to Ollama directly — same API, but with intelligent routing, observability, and cluster management layered on top.

## Quick Start

```bash
pip install sollol
```

```python
from sollol import OllamaPool

# Auto-discover all Ollama nodes on the network
pool = OllamaPool.auto_configure()

# Make a request — SOLLOL routes it to the best node
response = pool.chat(
    model="llama3.2",
    messages=[{"role": "user", "content": "Hello!"}]
)
print(response['message']['content'])
```

That's it. No async/await, no config files. It finds nodes, picks the best one, routes the request.

### CLI (Gateway Mode)

Run SOLLOL as a gateway that replaces Ollama on port 11434:

```bash
sollol up
```

Applications talking to `localhost:11434` now go through SOLLOL's routing engine instead of hitting a single node directly.

## Core Features

### Intelligent Routing

SOLLOL scores every node on multiple factors before routing:

| Factor | What It Checks |
|--------|---------------|
| **Success rate** | Historical reliability of each node |
| **Latency** | Response time from recent requests |
| **GPU availability** | Whether the node has a GPU (via gpustat + Redis) |
| **Current load** | Queue depth and active requests |
| **Task type** | Generation, embedding, or classification workloads |
| **Priority** | Request priority (CRITICAL → BATCH) |
| **Specialization** | Nodes that historically perform well for specific models |

Scoring formula:

```
Score = 100 (baseline)
      × success_rate
      ÷ (1 + latency_penalty)
      × gpu_bonus (1.5x if GPU available & needed)
      ÷ (1 + load_penalty)
      × priority_alignment
      × task_specialization
```

### Auto-Discovery

Scans the local network for Ollama instances. No need to configure node addresses manually.

### GPU-Aware Routing

Install the GPU reporter on each node:

```bash
sollol install-gpu-reporter --redis-host 192.168.1.10
```

Publishes real-time VRAM stats to Redis every 5 seconds. SOLLOL uses this to avoid routing heavy models to nodes that don't have the memory.

### Dashboard

```bash
python3 -m sollol.dashboard_service &
```

Web UI at `http://localhost:8080` showing:
- Node health and status
- P50/P95/P99 latency metrics
- Active applications using the cluster
- GPU memory usage (if reporter installed)
- Live request/activity logs

### Distributed Execution

**Ray** — Parallel request execution via Ray actors for multi-agent workloads.

**Dask** — Batch processing for embeddings and bulk inference with work stealing.

### Sync API

No async/await needed:

```python
from sollol import OllamaPool
from sollol.priority_helpers import Priority

pool = OllamaPool.auto_configure()

response = pool.chat(
    model="llama3.2",
    messages=[{"role": "user", "content": "Hello!"}],
    priority=Priority.HIGH,
    timeout=60
)
```

### Hedging

Duplicate slow requests to a second node and take the first response back. Configurable threshold.

### Circuit Breakers

Nodes that start failing get temporarily removed from the pool until they recover.

## Architecture

```
┌─────────────────────┐
│   Your Application   │
└──────────┬──────────┘
           │
           ▼
┌──────────────────────────────────┐
│      SOLLOL Gateway (:11434)     │
│  ┌────────────────────────────┐  │
│  │  Intelligent Routing Engine │  │
│  │  Scores all nodes, picks    │  │
│  │  the best, routes request   │  │
│  └────────────┬───────────────┘  │
│  ┌────────────┴───────────────┐  │
│  │  Priority Queue + Hedging  │  │
│  │  + Circuit Breakers        │  │
│  └────────────┬───────────────┘  │
└───────────────┼──────────────────┘
                │
    ┌───────────┼───────────┐
    ▼           ▼           ▼
┌───────┐  ┌───────┐  ┌───────┐
│ Node 1 │  │ Node 2 │  │ Node 3 │
│ GPU 24 │  │ GPU 16 │  │ CPU    │
└───────┘  └───────┘  └───────┘
```

## Installation

```bash
# From PyPI
pip install sollol

# From source
git clone https://github.com/B-A-M-N/SOLLOL.git
cd SOLLOL
pip install -e .
```

## Commands

| Command | Description |
|---------|-------------|
| `sollol up` | Start the SOLLOL gateway on port 11434 |
| `sollol install-gpu-reporter --redis-host <ip>` | Set up GPU monitoring on a node |

## Configuration

All settings work via environment variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `SOLLOL_PORT` | 11434 | Gateway port |
| `SOLLOL_RAY_WORKERS` | 4 | Ray actor count |
| `SOLLOL_DASK_WORKERS` | 2 | Dask worker count |
| `OLLAMA_NODES` | auto-discover | Comma-separated node addresses |
| `RPC_BACKENDS` | none | Comma-separated llama.cpp RPC backends |
| `SOLLOL_BATCH_PROCESSING` | true | Enable Dask batch mode |

## Design Principle

> *Route to the right node, fail over fast, and never lose a request to a dead endpoint.*

## Acknowledgments

- **[Dallan Loomis](https://github.com/DallanL)** — for the interactions and guidance that kept this project on track
- **My parents** — for the support that made all of this possible
- **My son** — the reason I build

## License

[MIT](LICENSE)
