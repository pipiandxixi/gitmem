To understand the **key changes** in the Digest Worker (and related components) and their impact on knowledge management, here's a structured breakdown:


## **1. Core Shift: Topic-Based → Pillar-Based Organization**
The system moves from **ad-hoc topic files** to **7 predefined cognitive pillars** for categorizing content. This simplifies maintenance and improves retrieval.


### **2. Key Enhancements in `src/digest/worker.py`**
#### **A. LLMClient: Pillar Categorization**
- **Added `CATEGORIES` Dictionary**: Defines 7 pillars with clear descriptions (critical for consistent categorization):
  ```python
  CATEGORIES = {
      "context": "事实背景: 客观存在的静态设定（项目简介、技术栈等）",
      "preferences": "规则偏好: 用户主观约束（代码风格、设计原则等）",
      "goals": "目标需求: 未来期望（产品愿景、Roadmap等）",
      "progress": "状态经历: 动态进度（已完成功能、Debug历史等）",
      "todos": "待办提示: 短期行动（TODO List、Bug提醒等）",
      "reference": "文档参考: 外部知识（第三方文档、常用命令等）",
      "misc": "其他碎片: 暂时无法归类的内容"
  }
  ```
- **Updated `route_topic` Method**:
  - Now categorizes content into pillars (not canonical topics).
  - Cleans LLM output to ensure valid pillar keys (e.g., `context` instead of `Category: context`).
- **Updated `digest_content` Method**:
  - Takes a `category` parameter to guide merging.
  - New prompt with **strict schema rules** (critical for knowledge maintainers):
    1. **Structure**: Use `##` for sections, `###` for subsections.
    2. **Deduplication**: Merge new facts into existing sections; update if newer/accurate.
    3. **Context**: Preserve source context (e.g., `> Source: project/GEMINI.md`).
    4. **No Hallucinations**: Do not invent information.
    5. **Full Rewrite**: Output the complete file (not just diffs).


#### **B. DigestWorker: Pillar File Management**
- **Removed Index Handling**: No more `index.yaml` (replaced by pillars).
- **Pillar Grouping**: Entries are grouped by pillar key (e.g., `context` → `context.md`).
- **Pillar File Creation**: If a pillar file doesn’t exist, it’s auto-generated with a standard front matter:
  ```markdown
  ---
  title: Context
  tags: []
  ---
  # Context
  ```


### **3. Server.py: Project Context Injection**
- **Context Enrichment**: Adds `[Context: project_name]` to all content (e.g., `[Context: gitmem]`).
- **Tag Injection**: Appends `project:project_name` to tags (e.g., `project:gitmem`).
- **Git Diff Context**: Includes project name in diff snapshots (avoids cross-project confusion).


### **4. Impact on Knowledge Maintenance**
As a **Knowledge Maintainer**, you should:
- **Follow Pillar Rules**: Always categorize content into the 7 pillars (use the `CATEGORIES` descriptions as a guide).
- **Preserve Context**: Include source references (e.g., `> Source: pyproject.toml`).
- **Deduplicate Smartly**: Merge new facts into existing sections; update if the new version is more recent/accurate.
- **Avoid Hallucinations**: Never invent information—stick strictly to input data.
- **Maintain Format**: Use `##` for sections, `###` for subsections, and preserve code blocks exactly.


## **Example Workflow**
1. **User Adds Content**: "Python version requirement: 3.10+" → tagged with `project:gitmem`.
2. **LLM Routes**: Categorizes into `context` pillar.
3. **Merge**: Added to `context.md` under `## Project Configuration` with source reference:
   ```markdown
   ## Project Configuration
   > Source: pyproject.toml
   
   - Python Version: 3.10+
   - Dependencies: fastapi, uvicorn
   ```


## **Summary**
The changes **simplify knowledge management** by:
- Using a fixed set of pillars (no more arbitrary topic files).
- Enforcing strict schema rules for consistency.
- Adding project context to avoid cross-project confusion.

This makes the system more maintainable, scalable, and easier to retrieve information from.