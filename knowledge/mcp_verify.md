---
title: mcp_verify
tags: ['verification', 'test']
---

# mcp_verify

## GitMem Hybrid Architecture Verification Steps
1. Local Write: Content is saved to the `logs/` directory.
2. Async Trigger: Backend receives a signal for the written entry.
3. Digestion: Worker processes the entry and generates `knowledge/mcp_verify.md`.
4. Result: Structured knowledge is automatically created.