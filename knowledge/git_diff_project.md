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
            if 'origin' in repo.remotes:
                repo.remotes.origin.set_url(auth_url)
            repo.remotes.origin.pull()
        except Exception as e:
            logger.error(f"Pull failed: {e}")
            logger.warning("Nuking corrupt repo and re-cloning...")
            shutil.rmtree(repo_path)
            Repo.clone_from(auth_url, repo_path)
            
    # Configure Bot Identity
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
- **Git URL Validation**: Ensures `GITMEM_URL` is set (env or explicit arg)
- **Error Logging**: Logs backend trigger failures to `~/.gitmem/trigger_error.log` with timestamps
- **Resolved Credentials**: Uses `current_git_url`/`token` (resolved from env/args) instead of direct inputs

### Code Snippet (Validation & Logging):
```python
# 1. Strict Configuration Validation
current_git_url = git_url or os.environ.get("GITMEM_URL")
current_git_token = git_token or os.environ.get("GITMEM_TOKEN")

if not current_git_url:
    return "Error: GITMEM_URL is missing. Please configure it in your environment or pass 'git_url' explicitly."

# ...

# 3. Trigger Remote Backend Digestion
if "Error:" not in result:
    backend_url = os.environ.get("GITMEM_BACKEND_URL", "http://localhost:8000")
    try:
        with httpx.Client(timeout=1.0) as client:
            payload = {"git_url": current_git_url, "git_token": current_git_token}
            resp = client.post(f"{backend_url}/trigger-digest", json=payload)
            
            if resp.status_code != 200:
                with open(os.path.expanduser("~/.gitmem/trigger_error.log"), "a") as f:
                    f.write(f"[{datetime.now()}] Backend returned {resp.status_code}: {resp.text}\n")
    except Exception as e:
        with open(os.path.expanduser("~/.gitmem/trigger_error.log"), "a") as f:
            f.write(f"[{datetime.now()}] Trigger failed: {str(e)}\n")
```

## Removed Files
- `sse_server.log` deleted (resolved `ModuleNotFoundError` for `src` module)