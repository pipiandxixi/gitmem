---
title: git_diff_project
tags: []
---

# git_diff_project

## Recent Log Changes
- **backend.log updates**:
  - Process ID changed from `[8580]` to `[35868]`
  - Digest consumer start timestamp updated from `2026-01-07 21:50:19` to `2026-01-08 16:33:37`

## Backend API Updates (src/backend.py)
### Key Changes:
- **New Imports**: `tempfile`, `hashlib`, `shutil`, `git.Repo` (removed `BackgroundTasks` and `MemoryStore`)
- **Repo Preparation Helper**: Added `_prepare_backend_repo` (isolated temp clones, auth handling, bot identity)
- **Digest Consumer**: Uses `git_url`/`token` instead of precomputed paths; auto-recovers corrupt repos
- **Trigger Endpoint**: Enqueues raw credentials; returns `url` instead of `repo_path`

### Code Snippet (Repo Helper):
```python
def _prepare_backend_repo(git_url: str, git_token: str | None) -> str:
    """
    Prepare a temporary clone of the repo in the system temp directory.
    This ensures isolation from any local user config or other services.
    """
    # 1. Calculate Temp Path
    url_hash = hashlib.sha256(git_url.encode()).hexdigest()[:12]
    base_tmp = os.path.join(tempfile.gettempdir(), "gitmem-backend")
    repo_path = os.path.join(base_tmp, url_hash)
    
    os.makedirs(base_tmp, exist_ok=True)
    
    # 2. Authenticate URL
    auth_url = git_url
    if git_token and git_url.startswith("https://"):
        auth_url = git_url.replace("https://", f"https://x-access-token:{git_token}@")

    # 3. Clone or Update
    if not os.path.exists(repo_path):
        logger.info(f"Cloning to temp: {repo_path}")
        try:
            Repo.clone_from(auth_url, repo_path)
        except Exception as e:
            logger.error(f"Clone failed: {e}")
            raise e
    else:
        logger.info(f"Updating temp repo: {repo_path}")
        try:
            repo = Repo(repo_path)
            # Update remote in case token changed
            if 'origin' in repo.remotes:
                repo.remotes.origin.set_url(auth_url)
            repo.remotes.origin.pull()
        except Exception as e:
            logger.error(f"Pull failed: {e}")
            # If repo is corrupt, nuking it is a valid backend strategy
            logger.warning("Nuking corrupt repo and re-cloning...")
            shutil.rmtree(repo_path)
            Repo.clone_from(auth_url, repo_path)
            
    # Configure Bot Identity for this temp repo
    repo = Repo(repo_path)
    with repo.config_writer() as git_config:
        if not git_config.has_option('user', 'name'):
            git_config.set_value('user', 'name', "GitMem Backend")
            git_config.set_value('user', 'email', "backend@gitmem.cloud")

    return repo_path
```

## Digest Worker Updates (src/digest/worker.py)
### Key Changes:
- **Push Method**: Added `GitClient.push()` to push changes to remote
- **Enhanced Logging**: Logs last hash, current head, and pending commit count
- **Auto-Push**: DigestWorker pushes immediately after committing (removed manual push from main block)
- **Error Handling**: Logs push failures without breaking workflow

### Code Snippet (Push Method):
```python
def push(self):
    try:
        origin = self.repo.remote(name='origin')
        origin.push()
        logger.info("Pushed changes to remote.")
    except Exception as e:
        logger.error(f"Push failed: {e}")
```

## Server CLI Updates (src/server.py)
### Key Changes:
- **Git URL Validation**: Ensures `GITMEM_URL` is set (env or explicit arg) in both `capture_context` and `remember` functions
- **Error Logging**: Logs backend trigger failures to `~/.gitmem/trigger_error.log` with timestamps (via helper)
- **Resolved Credentials**: Uses `current_git_url`/`token` (resolved from env/args) instead of direct inputs
- **Helper Refactor**: Extracted backend trigger logic into `_trigger_digest` helper function
- **Proxy Avoidance**: Added `trust_env=False` to backend HTTP calls to bypass system proxies for localhost communication
- **Unified Trigger**: Calls `_trigger_digest` once per capture/remember operation (instead of per file/diff)

### Code Snippet (Helper & Usage):
```python
def _trigger_digest(git_url: str, git_token: Optional[str] = None):
    """Notify the backend that new content is available for digestion."""
    backend_url = os.environ.get("GITMEM_BACKEND_URL", "http://localhost:8000")
    try:
        # Force no proxy for localhost communication to avoid 503 errors
        # when a system proxy is configured (e.g. http_proxy=...)
        with httpx.Client(timeout=1.0, trust_env=False) as client:
            payload = {"git_url": git_url, "git_token": git_token}
            resp = client.post(f"{backend_url}/trigger-digest", json=payload)
            if resp.status_code != 200:
                with open(os.path.expanduser("~/.gitmem/trigger_error.log"), "a") as f:
                    f.write(f"[{datetime.now()}] Backend returned {resp.status_code}: {resp.text}\n")
    except Exception as e:
        with open(os.path.expanduser("~/.gitmem/trigger_error.log"), "a") as f:
            f.write(f"[{datetime.now()}] Trigger failed: {str(e)}\n")

@mcp.tool()
def capture_context(scope: str = "all", git_url: str = None, git_token: str = None) -> str:
    # ... (other code)
    # Resolve config
    current_git_url = git_url or os.environ.get("GITMEM_URL")
    current_git_token = git_token or os.environ.get("GITMEM_TOKEN")

    if not current_git_url:
        return "Error: GITMEM_URL is missing."
    # ... (other code)
    # 3. Trigger Digest once for all updates
    _trigger_digest(current_git_url, current_git_token)

def remember(content: str, topic: str = "global", tags: List[str] = None,
             dependencies: List[str] = None, git_url: str = None, git_token: str = None) -> str:
    # ... (other code)
    current_git_url = git_url or os.environ.get("GITMEM_URL")
    current_git_token = git_token or os.environ.get("GITMEM_TOKEN")

    if not current_git_url:
        return "Error: GITMEM_URL is missing."
    # ... (other code)
    if "Error:" not in result:
        _trigger_digest(current_git_url, current_git_token)
```

## Removed Files
- `sse_server.log` deleted (resolved `ModuleNotFoundError: No module named 'src'` for SSE server initialization)
- Binary file `src/digest/__pycache__/worker.cpython-313.pyc` updated (no human-readable changes)