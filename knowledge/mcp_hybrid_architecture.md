---
title: mcp_hybrid_architecture
tags: ['architecture', 'hybrid-mcp', 'gitops']
---
# mcp_hybrid_architecture

Hybrid MCP Architecture Design for GitMem:
- Local Component (Stdio): Provides low-latency writes to local daily logs and fast reads from a cached repository.
- Remote Component (SSE): Acts as a centralized 'Brain' for heavy-duty, asynchronous LLM digestion.
- Trigger Mechanism: After a local write, the Local MCP Server sends a POST request to the remote '/trigger-digest' webhook.
- Data Flow: Raw memories are appended to 'logs/YYYY-MM-DD.md'. The remote worker then performs a full-rewrite merge of new facts into structured 'knowledge/{topic}.md' files.
- State Tracking: Uses 'Ref-Commit' markers in Git logs to manage the processing pipeline statelessly.