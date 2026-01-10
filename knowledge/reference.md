The provided diff shows a **major refactor** of the GitMem project's digest system, shifting from **topic-based** to **pillar-based** knowledge organization. Here's a breakdown of key changes:


### 1. **Pillar-Based Categorization (New Core)**
A new `CATEGORIES` dictionary in `LLMClient` defines 7 standardized pillars for organizing knowledge:
```python
CATEGORIES = {
    "context": "事实背景: 客观存在的静态设定（项目简介、技术栈等）",
    "preferences": "规则偏好: 用户主观约束（代码风格、交互习惯等）",
    "goals": "目标需求: 未来期望（产品愿景、Roadmap等）",
    "progress": "状态经历: 开发进度（已完成功能、Debug历史等）",
    "todos": "待办提示: 短期任务（TODO、Bug提醒等）",
    "reference": "文档参考: 外部知识（第三方文档、命令等）",
    "misc": "其他碎片: 暂无法归类的内容"
}
```


### 2. **LLM Routing: From Topics to Pillars**
The `route_topic` method now **categorizes content into pillars** (instead of matching to canonical topics):
- Uses the `CATEGORIES` description to generate a prompt.
- Returns a pillar key (e.g., `context`, `preferences`) instead of a topic name.
- Defaults to `misc` if unsure.


### 3. **Digest Content: Strict Schema & Merging**
The `digest_content` method now:
- Takes a `category` parameter to target the correct pillar file.
- Uses a detailed prompt with **schema rules** (structure, deduplication, context preservation).
- Example output format enforces consistent Markdown structure with front matter.


### 4. **Digest Worker: Pillar File Management**
The `DigestWorker` class:
- **Removed** the `load_index` method (no longer uses `index.yaml` for topics).
- **Group by Pillar**: Entries are grouped by pillar key (e.g., `context`, `preferences`).
- **File Naming**: Uses pillar names for files (e.g., `context.md` instead of topic-specific files).
- **Initial File Setup**: Creates pillar files with standardized front matter if missing.


### 5. **Project Context Injection**
The `server.py` adds **project context** to all entries:
- Injects the project name (from current working directory) into content and tags.
- Example: `[Context: gitmem] Source: pyproject.toml` or `project:gitmem` tag.
- Ensures entries are scoped to the correct project for multi-project support.


### 6. **Storage Submodule Update**
The `storage` submodule was updated (new commit hash), likely reflecting changes in the underlying storage repo.


### Key Benefits of This Refactor
- **Scalability**: Pillars provide a consistent, scalable way to organize knowledge (no need for manual topic management).
- **Clarity**: Standardized pillars make it easier to find and understand content.
- **Simplification**: Removes the `index.yaml` dependency and simplifies routing logic.
- **Project Isolation**: Context injection ensures entries are scoped to the correct project.


This refactor aligns with GitMem's goal of **structured, maintainable knowledge management** for AI agents, making it easier to organize and retrieve information.