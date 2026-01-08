---
title: context_project_readme
tags: []
---
# GitMem

GitMem is a semantic memory service for AI Agents, backed by Git. It treats memory as a "Living Codebase".

## Architecture

GitMem employs a **Hybrid Architecture** to balance latency and intelligence:

1.  **Local MCP Server (`src/server.py`)**: 
    - Runs locally (Stdio) with your Agent (Claude, Windsurf, Gemini).
    - Provides instant `remember` and `recall` capabilities using a local Git cache.
    - Triggers the backend service upon new writes.

2.  **Backend Service (`src/backend.py`)**:
    - Runs permanently (e.g., on Render).
    - Receives `trigger-digest` signals.
    - Runs an asynchronous **Digest Worker** that uses LLMs to summarize raw logs into structured knowledge.

3.  **Storage (`Git Repo`)**:
    - `logs/`: Raw, append-only daily logs (High Fidelity).
    - `knowledge/`: AI-curated, structured markdown files (High Utility).

## Setup

### 1. Requirements
- Python 3.10+
- OpenAI API Key (for digestion)
- A Git Repository (GitHub/GitLab)

### 2. Installation
```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 3. Running the Backend
Create `src/digest/.env` with your LLM keys (see `.env.example`).
```bash
python src/backend.py
```

### 4. Configuring Your Agent
Add the local server to your `claude_desktop_config.json` or equivalent:
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

## Usage
Simply tell your Agent:
> "Remember that the deployment port for QuantServer is 9090."

GitMem will:
1.  Save it to `logs/YYYY-MM-DD.md`.
2.  Trigger the backend.
3.  Update `knowledge/quant_server.md` automatically.