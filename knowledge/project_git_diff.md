---
title: project_git_diff
tags: [context, project, diff]
---

# project_git_diff

## Dependencies Added (pyproject.toml)
```toml
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
```

## New `capture_context` Tool (server.py)
```python
# Configuration for Context Sources
CONTEXT_SOURCES = {
    "project": [
        "GEMINI.md", "README.md", "DESIGN.md", "ARCHITECTURE.md",
        "pyproject.toml", "package.json", "requirements.txt"
    ],
    "user": [
        "~/.gemini/GEMINI.md",
        "~/.gemini/memory.md"
    ],
    "system": [
        ".cursorrules",
        ".editorconfig"
    ]
}

@mcp.tool()
def capture_context(scope: str = "all", git_url: str = None, git_token: str = None) -> str:
    """
    Auto-capture project context and working state into memory.
    
    Args:
        scope: 'project', 'user', 'system', or 'all'. Determines which files to scan.
        git_url: Remote Git URL for storage.
        git_token: Remote Git Token.
    """
    logs = []
    scopes_to_process = []
    
    if scope == "all":
        scopes_to_process = ["project", "user", "system"]
    elif scope in CONTEXT_SOURCES:
        scopes_to_process = [scope]
    else:
        return f"Error: Invalid scope '{scope}'. Valid scopes: {list(CONTEXT_SOURCES.keys())} or 'all'."

    # 1. Capture Files
    for s in scopes_to_process:
        patterns = CONTEXT_SOURCES.get(s, [])
        for pattern in patterns:
            # Handle user expansion and relative paths
            full_path = os.path.expanduser(pattern)
            if not os.path.isabs(full_path):
                full_path = os.path.abspath(full_path)
            
            if os.path.exists(full_path) and os.path.isfile(full_path):
                try:
                    with open(full_path, 'r', encoding='utf-8') as f:
                        content = f.read()
                    
                    # Store it
                    # Topic convention: context/{scope}/{filename}
                    filename = os.path.basename(full_path)
                    topic = f"context/{s}/{filename}"
                    
                    res = store.remember(
                        content=content,
                        topic=topic,
                        tags=["context", s, "auto-capture"],
                        git_url=git_url,
                        git_token=git_token
                    )
                    logs.append(f"[File] {filename}: {res}")
                except Exception as e:
                    logs.append(f"[Error] Failed to read {full_path}: {e}")

    # 2. Capture Git Diff (Only for project scope or all)
    if "project" in scopes_to_process:
        try:
            # Check for unstaged changes
            diff_proc = subprocess.run(["git", "diff", "HEAD"], capture_output=True, text=True)
            if diff_proc.returncode == 0 and diff_proc.stdout.strip():
                diff_content = diff_proc.stdout
                res = store.remember(
                    content=diff_content,
                    topic="context/project/git_diff",
                    tags=["context", "project", "diff"],
                    git_url=git_url,
                    git_token=git_token
                )
                logs.append(f"[Diff] Git Diff captured: {res}")
            else:
                logs.append("[Diff] No active changes found in git.")
        except Exception as e:
            logs.append(f"[Diff] Error capturing git diff: {e}")

    return "\n".join(logs)
```

## Updated `sync` Method (store.py)
```python
def sync(self, git_url: Optional[str] = None, git_token: Optional[str] = None) -> str:
    """Manually pull and push to the remote repository."""
    git_url = git_url or os.environ.get("GITMEM_URL")
    git_token = git_token or os.environ.get("GITMEM_TOKEN")
    if not git_url:
        return "Error: GITMEM_URL is not set."
    try:
        repo = self._prepare_repo(git_url, git_token)
        repo.remotes.origin.pull()
        repo.remotes.origin.push()
        return f"Successfully synced with {git_url}"
    except Exception as e:
        return f"Sync failed: {str(e)}"
```

## Other Changes
- Added `subprocess` import to `server.py`.
- Updated `server.py` main block to use a `main()` function:
  ```python
  def main():
      mcp.run()

  if __name__ == "__main__":
      main()
  ```