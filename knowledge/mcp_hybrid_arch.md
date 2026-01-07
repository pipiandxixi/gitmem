---
title: mcp_hybrid_arch
tags: ['architecture', 'hybrid-model', 'refactoring']
---
# mcp_hybrid_arch
Hybrid MCP Architecture Design for GitMem:
- **Local Component**: `src/server.py` running in Stdio mode. Handles fast, synchronous writes to local `logs/YYYY-MM-DD.md`.
- **Remote Component**: `src/backend.py` running as a persistent service. Exposes `/trigger-digest` and runs async digestion workers.
- **Workflow**: Local write -> HTTP Trigger -> Remote Worker pulls repo -> LLM Digests content -> Push back to `knowledge/`.
- **Storage Layout**: Raw logs in `logs/`, processed knowledge in `knowledge/`.