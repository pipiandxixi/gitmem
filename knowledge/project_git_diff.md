---
title: GitMem Project Documentation
description: A hybrid tool for capturing, storing, and digesting context from Git repositories using a backend-worker architecture
author: GitMem Team
date: 2026-01-08
version: 1.1.0
---

# GitMem Project Documentation

GitMem is a tool designed to help developers capture, store, and digest context from their Git repositories. It uses a hybrid architecture with a server for user interactions, a backend for processing, and a worker for digesting changes.


## Configuration

GitMem requires several environment variables for proper operation:

| Variable               | Description                                                                 | Required |
|-------------------------|-----------------------------------------------------------------------------|----------|
| `GITMEM_URL`            | HTTPS URL of your Git repository (used for storing context)                 | Yes      |
| `GITMEM_TOKEN`          | Personal Access Token (PAT) for authenticating with your Git repository     | Yes      |
| `GITMEM_BACKEND_URL`    | URL of the GitMem backend service (default: `http://localhost:8000`)        | No       |

**Note:** These variables can be set in your environment or passed as arguments to the CLI functions.


## Core Components

GitMem consists of three main components: **Server**, **Backend**, and **Digest Worker**.

### 1. Server (`src/server.py`)
The server handles user interactions (CLI or API) and triggers context capture and storage.

**Key Changes:**
- Added `_trigger_digest` helper function to notify the backend of new content.
- Resolved configuration (uses `GITMEM_URL`/`GITMEM_TOKEN` from environment or arguments).
- Added error handling for missing `GITMEM_URL`.
- Triggers digest **once** after all context captures (instead of per-file).
- Removed redundant push logic from the `remember` function.

**Code Snippet (Modified `capture_context`):**
```python
def capture_context(scope: str = "all", git_url: str = None, git_token: str = None) -> str:
    # Resolve config
    current_git_url = git_url or os.environ.get("GITMEM_URL")
    current_git_token = git_token or os.environ.get("GITMEM_TOKEN")

    if not current_git_url:
        return "Error: GITMEM_URL is missing."
    
    # ... (rest of the function)
    
    # Trigger Digest once for all updates
    _trigger_digest(current_git_url, current_git_token)
```

### 2. Backend (`src/backend.py`)
The backend manages the digest queue and processes tasks asynchronously.

**Key Changes:**
- Removed dependency on `BackgroundTasks` (uses async queue instead).
- Added `_prepare_backend_repo` function to handle temp repo cloning/updating:
  - Clones repo to a temp directory (isolated from local config).
  - Authenticates using `x-access-token` for HTTPS URLs.
  - Recovers from corrupt repos by re-cloning.
- Modified `digest_consumer` to process `git_url` and `git_token` (instead of `repo_path`).
- Updated `trigger_digest` endpoint to enqueue `git_url` and `token`.

**Code Snippet (New `_prepare_backend_repo`):**
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

### 3. Digest Worker (`src/digest/worker.py`)
The worker processes pending commits and generates a digest of changes.

**Key Changes:**
- Added `push` method to `GitClient` class (pushes changes to remote).
- Modified `DigestWorker.run()` to call `self.git.push()` after committing changes.
- Removed redundant push logic from the main block.
- Added logging for:
  - Last processed commit hash vs current head.
  - Number of pending commits.
- Improved error handling for corrupt repos.

**Code Snippet (New `push` Method):**
```python
class GitClient:
    # ... (other methods)
    
    def push(self):
        try:
            origin = self.repo.remote(name='origin')
            origin.push()
            logger.info("Pushed changes to remote.")
        except Exception as e:
            logger.error(f"Push failed: {e}")
```


## API Endpoints

### Backend Endpoints
The backend exposes a single endpoint to trigger digest processing.

| Endpoint               | Method | Description                                                                 | Request Body                                  |
|-------------------------|--------|-----------------------------------------------------------------------------|------------------------------------------------|
| `/trigger-digest`       | POST   | Triggers a background digest for the specified Git repository.              | `{"git_url": "<repo-url>", "git_token": "<token>"}` |

**Response:**
- Success: `{"status": "queued", "url": "<repo-url>"}`
- Error: `{"detail": "<error-message>"}` (status code 500)


## Usage

### CLI Functions
GitMem provides two main CLI functions: `capture_context` and `remember`.

#### 1. `capture_context`
Captures context from files and Git diffs and stores them in your Git repository.

**Arguments:**
- `scope`: Scope of context to capture (`all`, `project`, `user`, `system`). Default: `all`.
- `git_url`: HTTPS URL of your Git repository (overrides `GITMEM_URL`).
- `git_token`: Personal Access Token (overrides `GITMEM_TOKEN`).

**Example:**
```bash
gitmem capture_context --scope project
```

#### 2. `remember`
Stores or updates knowledge in your Git repository.

**Arguments:**
- `content`: Text content to remember.
- `topic`: General topic (e.g., `project_alpha`). Default: `global`.
- `tags`: Optional list of tags (comma-separated).
- `dependencies`: Optional list of related topics (comma-separated).
- `git_url`: HTTPS URL of your Git repository (overrides `GITMEM_URL`).
- `git_token`: Personal Access Token (overrides `GITMEM_TOKEN`).

**Example:**
```bash
gitmem remember --content "How to set up Redis for caching" --topic "redis" --tags "caching,setup"
```


## Troubleshooting

### Common Issues
1. **Missing `GITMEM_URL`:**
   - **Error:** `Error: GITMEM_URL is missing.`
   - **Fix:** Set the `GITMEM_URL` environment variable or pass it as an argument.

2. **Backend Trigger Failure:**
   - **Error:** Check `~/.gitmem/trigger_error.log` for details.
   - **Fix:** Ensure the backend is running at the correct URL (check `GITMEM_BACKEND_URL`).

3. **Corrupt Repo:**
   - **Error:** Backend logs show `Nuking corrupt repo and re-cloning...`.
   - **Fix:** No action needed—backend automatically recovers by re-cloning the repo.

### Deleted Files
- `sse_server.log`: Removed due to `ModuleNotFoundError` (no longer used).


## Changelog (v1.1.0)
- **Added:** `_trigger_digest` helper function in server.
- **Added:** Temp repo management in backend.
- **Added:** `push` method in GitClient.
- **Improved:** Configuration resolution (env vars + args).
- **Improved:** Error logging for backend triggers.
- **Removed:** Redundant push logic in worker.
- **Removed:** Dependency on `BackgroundTasks` in backend.


## License
GitMem is licensed under the MIT License. See `LICENSE` for details.