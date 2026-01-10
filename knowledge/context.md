---
title: Context
tags: []
---

# Context

## GitMem Project Overview

### Vision
GitMem is a semantic memory service for AI Agents, backed by Git. It treats memory as a "Living Codebase" and provides persistent, traceable long-term memory with an ingestion-digestion闭环 capability via the MCP (Model Context Protocol) protocol.

### Core Principles
1. **Ingestion Over Classification**: Raw memories are archived daily as "流水账" (logs/YYYY-MM-DD.md) to ensure context integrity.
2. **GitOps Driven**: Uses Git Commit as an event bus for stateless backend processing.
3. **Hybrid Architecture**: Local MCP Server (FastMCP) for fast read/write; remote Backend (FastAPI) for asynchronous LLM summarization.
4. **Dual Storage**: `logs/` stores raw data; `knowledge/` stores structured knowledge.

### Current Status
- [x] **Hybrid Architecture Implemented**:
  - `src/server.py`: Local client with Git cache read/write + HTTP Trigger.
  - `src/backend.py`: Remote server (FastAPI + Async Queue) supporting multi-tenant summarization.
- [x] **Storage Engine Refactored**:
  - Raw memories stored in `logs/` subdirectory.
  - Daily log headers automatically maintain cumulative tags.
- [x] **Smart Digest Service**:
  - Implemented incremental parsing based on Git Diff.
  - Implemented LLM-driven topic routing (`index.yaml`) and full rewrite summarization.
- [x] **Multi-Tenant Support**: Dynamic user repo pulling via Trigger; stateless service.

### Key Decisions
- **Communication Protocol**: RESTful (`POST /trigger-digest`) between local and backend (simplifies architecture, avoids SSE).
- **State Tracking**: Strictly uses `Ref-Commit` markers in Git Commit Messages; no local state files.
- **Summarization Strategy**: Full rewrite mode for knowledge files to ensure document structure consistency.

### TODOs
- [ ] **Semantic Retrieval Refactoring**:
  - **Smart Recall**: Prioritize structured data in `knowledge/`, supplement with latest increments in `logs/`.
  - **History**: Generate topic evolution graphs from `knowledge` file version history.
  - **Graph**: Generate knowledge heatmaps based on `index.yaml` and reference relationships.
- [ ] **Advanced Summarization Capabilities**:
  - Support conflict detection and manual intervention markers.
  - Regular full compaction (merge fragmented small topics).

## GitMem Architecture

GitMem uses a **Hybrid Architecture** to balance latency and intelligence:

1. **Local MCP Server (`src/server.py`)**:
   - Runs locally (stdio) with AI Agents (Claude, Windsurf, Gemini).
   - Provides instant `remember` and `recall` via local Git cache.
   - Triggers backend service on new writes.

2. **Backend Service (`src/backend.py`)**:
   - Runs permanently (e.g., on Render).
   - Receives `trigger-digest` signals.
   - Runs asynchronous **Digest Worker** using LLMs to summarize raw logs into structured knowledge.

3. **Storage (Git Repo)**:
   - `logs/`: Raw, append-only daily logs (high fidelity).
   - `knowledge/`: AI-curated structured markdown files (high utility).

## GitMem Setup & Usage

### Requirements
- Python 3.10+
- OpenAI API Key (for summarization)
- A Git Repository (GitHub/GitLab)

### Installation
```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### Running the Backend
Create `src/digest/.env` with LLM keys (see `.env.example`):
```bash
python src/backend.py
```

### Configuring Your Agent
Add the local server to your agent config (e.g., `claude_desktop_config.json`):
```json
"gitmem": {
  "command": "python",
  "args": ["/path/to/gitmem/src/server.py"],
  "env": {
    "GITMEM_URL": "https://github.com/your/memory-repo.git",
    "GITMEM_TOKEN": "your-git-token",
    "GITMEM_BACKEND_URL": "http://localhost:8000"
  }
}
```

### Usage Example
Tell your Agent:
> "Remember that the deployment port for QuantServer is 9090."

GitMem will:
1. Save it to `logs/YYYY-MM-DD.md`.
2. Trigger the backend.
3. Update `knowledge/quant_server.md` automatically.

## GitMem Technical Design

### Storage Schema

#### Directory Structure
**Dual Storage Architecture** distinguishes raw ingestion from processed knowledge:
```text
storage/
├── logs/                 # [User Input] Raw Memories
│   ├── YYYY-MM-DD.md     # Daily append-only logs
│   └── ...
├── knowledge/            # [System Output] Digested Knowledge
│   ├── [topic].md        # Structured topic-specific docs
│   └── ...
```

#### Raw Logs Markdown Format
Daily log files (`logs/YYYY-MM-DD.md`) use append mode with secondary headers for entries:
```markdown
---
date: 2026-01-07
tags: [cumulative, tags, list]
---


## Core Components

### MemoryStore (`src/store.py`)
- **Git Wrapper**: Handles `git init`, `add`, `commit`, `log` operations.
- **Metadata Manager**: Parses and merges Front Matter.
- **Daily Logger**: Appends `remember` requests to the day's log and updates cumulative tags in the header.

### Digest Worker (`src/digest/`)
Stateless background service for converting raw logs to structured knowledge:
- **Architecture**: "GitOps for Memory".
- **State Management**: Tracks progress via `Ref-Commit: <hash>` in Git Commit Messages.
- **Process**:
  1. Find last processed commit hash via `git log`.
  2. Identify new content in `logs/` since last processing via `git diff`.
  3. Call LLM for semantic summarization and merging.
  4. Commit updates to `knowledge/` with current HEAD hash in the message.
- **Details**: See `src/digest/README.md`.

### MCP Server (`src/server.py`)
Exposes tools via FastMCP:
- `remember(content, topic, tags)`: Appends to `logs/YYYY-MM-DD.md`.
- `recall(query, topic)`: Scans logs (short-term); prioritizes `knowledge/` (long-term).
- `history/graph`: Planned refactoring based on `knowledge` directory.

### Evolution Logic
- **Ingestion Strategy**: Minimal append (users don't need to manage filenames; context auto-saves to daily logs).
- **Digestion Strategy**: Asynchronous incremental (background worker organizes logs into structured knowledge).

## GitMem Project Dependencies

From `pyproject.toml`:
```toml
[project]
name = "gitmem"
version = "0.1.0"
description = "Git-backed, Metadata-enriched Memory Service for MCP"
requires-python = ">=3.10"
dependencies = [
    "mcp[cli]>=0.1.0",
    "gitpython>=3.1.0",
    "python-frontmatter>=1.0.0",
    "pyyaml>=6.0",
    "fastapi>=0.100.0",
    "uvicorn>=0.20.0",
    "httpx>=0.24.0",
    "openai>=1.0.0",
    "python-dotenv>=1.0.0",
    "pydantic>=2.0.0"
]

[project.scripts]
gitmem = "src.server:main"
```

## User-Specific Memories (Gemini)

### QuantServer Project Guidelines
- **Core Principles**:
  1. NO hidden defaults for quant variables (must be explicit/from config).
  2. Transparent error reporting (no masking errors with fake 100% lines).
  3. Strict data integrity validation before calculation.
  4. Absolute logic parity between backtest and production modules.
  
- **Mandatory UI/Frontend Workflow**:
  All UI changes in QuantServer must be verified locally before pushing to Render:
  1. Run local server via `run_server.py`.
  2. Visit `http://localhost:8000/view/QQQ/1y` using browser tools.
  3. Check console for syntax/type errors.
  4. Verify chart rendering and interactions.
  Do NOT push if local verification fails.