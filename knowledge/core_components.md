---
title: core_components
tags: []
---

# core_components

## 2. 核心组件 (Core Components)

### 2.1 MemoryStore (`src/store.py`)
- **Git Wrapper**: 负责 `git init`, `add`, `commit`, `log` 等操作。
- **Metadata Manager**: 负责 Front Matter 的解析与合并。
- **Daily Logger**: 负责将所有 `remember` 请求追加到当日的日志文件中，并更新文件头的累积标签。

### 2.2 Digest Worker (`src/digest/`) - *Planned*
一个独立的后台服务，负责将 `logs/` 中的原始数据归纳为 `knowledge/` 中的结构化知识。
- **架构**: "GitOps for Memory"。
- **状态管理**: 无状态设计。通过 Git Commit Message 中的 `Ref-Commit: <hash>` 标记来追踪处理进度。
- **流程**: 
    1. `git log` 查找上次机器处理的 Commit Hash。
    2. `git diff` 找出自上次处理以来 `logs/` 目录的新增内容。
    3. 调用 LLM 进行语义归纳与合并。
    4. 提交更新到 `knowledge/`，并在 Commit Message 中标记当前的 HEAD Hash。
- **详见**: `src/digest/README.md`

### 2.3 MCP Server (`src/server.py`)
利用 FastMCP 暴露以下工具：
- `remember(content, topic, tags)`: 将记忆追加到 `logs/YYYY-MM-DD.md`。
- `recall(query, topic)`: (短期) 扫描所有日志文件；(长期) 优先查询 `knowledge/`，辅以 `logs/` 的最新增量。
- `history/graph`: *待重构，将基于 knowledge 目录实现。*

## 3. 演进逻辑
- **摄入策略**: 极简追加。用户无需关心文件名，所有上下文自动落盘到当日日志。
- **消化策略**: 异步增量。后台 Worker 负责“整理房间”，将散乱的日志整理为有序的知识。