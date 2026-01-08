---
title: storage_architecture
tags: []
---

# storage_architecture

## 1. 存储架构 (Storage Schema)

### 1.1 目录结构
我们采用 **"双层存储架构"**，区分原始摄入与加工知识。
```text
storage/
├── logs/                 # [User Input] 原始记忆 (Raw Data)
│   ├── YYYY-MM-DD.md     # 按日归档的流水账
│   └── ...
├── knowledge/            # [System Output] 归纳后的知识库 (Digested Data)
│   ├── [topic].md        # 特定话题的结构化知识
│   └── ...
```

### 1.2 Markdown 格式规范 (Raw Logs)
每日日志文件 (`logs/YYYY-MM-DD.md`) 采用追加模式，通过二级标题区分条目。
```markdown
---
date: 2026-01-07
tags: [cumulative, tags, list]
---

## 条目标题
条目内容...
```