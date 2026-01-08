---
title: pyproject_toml
tags: []
---
# pyproject_toml
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