---
title: GitMem Project Full Updated Content
description: Comprehensive update of GitMem backend and server code
date: 2026-01-08
author: GitMem Team
tags: [gitmem, backend, digest, server, git]
---

# GitMem Project Updates

## Updated Code Files

### `src/backend.py`
```python
import os
import sys
import asyncio
import logging
import tempfile
import hashlib
import shutil
from contextlib import asynccontextmanager
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from git import Repo

# Add project root to sys.path
project_root = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
if project_root not in sys.path:
    sys.path.insert(0, project_root)

from src.digest.worker import DigestWorker

# Setup Logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("gitmem-backend")

# Global Queue
digest_queue = asyncio.Queue()

class TriggerRequest(BaseModel):
    git_url: str
    git_token: str | None = None

# --- Helper ---
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

# --- Background Consumer ---
async def digest_consumer():
    """Consume tasks from the queue and run the worker."""
    while True:
        try:
            task = await digest_queue.get()
            git_url = task.get("git_url")
            git_token = task.get("git_token")
            
            if git_url:
                logger.info(f"Processing digest for: {git_url}")
                try:
                    # Prepare repo in thread to avoid blocking loop
                    repo_path = await asyncio.to_thread(_prepare_backend_repo, git_url, git_token)
                    
                    # Run Worker
                    worker = DigestWorker(repo_path)
                    await asyncio.to_thread(worker.run)
                    logger.info(f"Finished digest for: {git_url}")
                except Exception as e:
                    logger.error(f"Digest failed for {git_url}: {e}")
            
            digest_queue.task_done()
        except asyncio.CancelledError:
            logger.info("Digest consumer cancelled.")
            break
        except Exception as e:
            logger.error(f"Error in consumer: {e}")
            # Avoid queue lockup
            if 'task' in locals() and task: 
                digest_queue.task_done()

# --- Lifecycle ---
@asynccontextmanager
async def lifespan(app: FastAPI):
    # Start consumer on app startup
    consumer_task = asyncio.create_task(digest_consumer())
    yield
    # Cleanup on app shutdown
    consumer_task.cancel()
    await consumer_task

app = FastAPI(title="GitMem Backend", lifespan=lifespan)

@app.post("/trigger-digest")
async def trigger_digest(request: TriggerRequest):
    """Trigger a background digest for the given repo."""
    try:
        # Just enqueue parameters. Let the worker handle path and cloning.
        await digest_queue.put({
            "git_url": request.git_url, 
            "git_token": request.git_token
        })
        return {"status": "queued", "url": request.git_url}
    except Exception as e:
        logger.error(f"Trigger failed: {e}")
        raise HTTPException(status_code=500, detail=str(e))
```

### `src/digest/worker.py`
```python
import os
import re
import logging
from git import Repo
from typing import List, Dict, Optional

# Setup Logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("gitmem-worker")

class GitClient:
    def __init__(self, repo_path: str):
        self.repo = Repo(repo_path)
        self.repo_path = repo_path
        self.digest_file = os.path.join(repo_path, "digest.txt")
        self.last_processed_hash_file = os.path.join(repo_path, ".last_processed_hash")

    def get_last_processed_hash(self) -> Optional[str]:
        """Get the last commit hash that was processed by the digest."""
        if not os.path.exists(self.last_processed_hash_file):
            return None
        with open(self.last_processed_hash_file, "r") as f:
            return f.read().strip()

    def set_last_processed_hash(self, hash_str: str):
        """Set the last processed commit hash."""
        with open(self.last_processed_hash_file, "w") as f:
            f.write(hash_str)

    def get_pending_commits(self, last_hash: str) -> List[Repo.commit]:
        """Get all commits since the last processed hash."""
        if not last_hash:
            return list(self.repo.iter_commits())
        return list(self.repo.iter_commits(f"{last_hash}..HEAD"))

    def get_commit_content(self, commit: Repo.commit) -> str:
        """Get the content of the commit message and changes."""
        return f"Commit: {commit.hexsha}\nAuthor: {commit.author}\nDate: {commit.committed_datetime}\nMessage: {commit.message}\nChanges:\n{commit.stats}\n"

    def commit_work(self, ref_hash: str, message: str = "Digest: Updated knowledge base"):
        """Commit the digest work to the repo."""
        self.repo.index.add([self.digest_file, self.last_processed_hash_file])
        self.set_last_processed_hash(ref_hash)
        self.repo.index.commit(f"{message}\n\nRef-Commit: {ref_hash}")
        logger.info(f"Committed digest work. Ref: {ref_hash}")

    def push(self):
        try:
            origin = self.repo.remote(name='origin')
            origin.push()
            logger.info("Pushed changes to remote.")
        except Exception as e:
            logger.error(f"Push failed: {e}")

class DigestWorker:
    def __init__(self, repo_path: str):
        self.repo_path = repo_path
        self.git = GitClient(repo_path)
        self.digest_file = os.path.join(repo_path, "digest.txt")

    def run(self):
        """Run the digest process."""
        logger.info(f"Starting digest for repo: {self.repo_path}")
        
        last_hash = self.git.get_last_processed_hash()
        current_head = self.git.repo.head.commit.hexsha
        logger.info(f"Last Hash: {last_hash}, Current Head: {current_head}")
        
        if last_hash == current_head:
            logger.info("Up to date.")
            return

        commits = self.git.get_pending_commits(last_hash)
        logger.info(f"Found {len(commits)} pending commits.")
        if not commits:
            self.git.commit_work(current_head, "Digest: fast-forward")
            self.git.push()
            return
        
        # 1. Extract Raw Entries
        raw_entries = []
        for commit in commits:
            raw_entries.append(self.git.get_commit_content(commit))
        
        if not entries:
             self.git.commit_work(current_head, "Digest: no parseable content")
             self.git.push()
             return

        # 2. Group by Canonical Topic
        topic_groups = self._group_by_topic(raw_entries)
        
        # 3. Generate Final Digest
        final_digest = self._generate_final_digest(topic_groups)
        
        # 4. Write to Digest File
        with open(self.digest_file, "w") as f:
            f.write(final_digest)
        
        # 4. Commit & Push
        self.git.commit_work(current_head)
        self.git.push()

    def _group_by_topic(self, entries: List[str]) -> Dict[str, List[str]]:
        """Group entries by canonical topic."""
        topic_groups = {}
        for entry in entries:
            # Extract topic from entry (simplified for example)
            topic = self._extract_topic(entry)
            if topic not in topic_groups:
                topic_groups[topic] = []
            topic_groups[topic].append(entry)
        return topic_groups

    def _extract_topic(self, entry: str) -> str:
        """Extract canonical topic from entry (simplified)."""
        # Example: Extract topic from commit message or changes
        if "Topic:" in entry:
            return entry.split("Topic:")[1].split("\n")[0].strip()
        return "Uncategorized"

    def _generate_final_digest(self, topic_groups: Dict[str, List[str]]) -> str:
        """Generate final digest from topic groups."""
        final_text = "# GitMem Digest\n\n"
        for topic, entries in topic_groups.items():
            final_text += f"## {topic}\n\n"
            for entry in entries:
                final_text += f"- {entry}\n\n"
        return final_text

import argparse
import sys

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="GitMem Digest Worker")
    parser.add_argument("repo_path", help="Path to the Git repository")
    args = parser.parse_args()

    try:
        # 1. Initialize Worker
        worker = DigestWorker(args.repo_path)
        # 2. Run Digest
        worker.run()
    except Exception as e:
        logger.error(f"Worker failed: {e}")
```

### `src/server.py`
```python
import os
import sys
import subprocess
import httpx
from datetime import datetime
from mcp.server.fastmcp import FastMCP
from src.store import MemoryStore
from typing import List, Optional

# Add project root to sys.path
project_root = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
if project_root not in sys.path:
    sys.path.insert(0, project_root)

# Setup error logging for debugging startup
LOG_FILE = os.path.expanduser("~/.gitmem/error.log")
os.makedirs(os.path.dirname(LOG_FILE), exist_ok=True)

# Initialize Memory Store
store = MemoryStore()

# Context Sources Configuration
CONTEXT_SOURCES = {
    "project": [
        "*.py", "*.js", "*.ts", "*.md", "requirements.txt", "package.json"
    ],
    "user": [
        "~/.bashrc", "~/.zshrc", "~/.gitconfig", "~/.ssh/config"
    ],
    "system": [
        "/etc/hostname", "/etc/os-release"
    ]
}

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
    """
    Capture context from the specified scope and store it in GitMem.
    
    Args:
        scope: The scope to capture (all, project, user, system). Defaults to "all".
        git_url: Optional Git URL to override the default.
        git_token: Optional Git token to override the default.
    """
    logs = []
    scopes_to_process = []
    
    # Resolve config
    current_git_url = git_url or os.environ.get("GITMEM_URL")
    current_git_token = git_token or os.environ.get("GITMEM_TOKEN")

    if not current_git_url:
        return "Error: GITMEM_URL is missing."

    if scope == "all":
        scopes_to_process = ["project", "user", "system"]
    elif scope in CONTEXT_SOURCES:
        scopes_to_process = [scope]
    else:
        return f"Error: Invalid scope '{scope}'. Valid scopes: {list(CONTEXT_SOURCES.keys())}"

    # 1. Capture Files
    for s in scopes_to_process:
        patterns = CONTEXT_SOURCES.get(s, [])
        for pattern in patterns:
            full_path = os.path.expanduser(pattern)
            if not os.path.isabs(full_path):
                full_path = os.path.abspath(full_path)
            if os.path.exists(full_path) and os.path.isfile(full_path):
                try:
                    with open(full_path, 'r', encoding='utf-8') as f:
                        content = f.read()
                    filename = os.path.basename(full_path)
                    topic = f"context/{s}/{filename}"
                    res = store.remember(
                        content=content,
                        topic=topic,
                        tags=["context", s, "auto-capture"],
                        git_url=current_git_url,
                        git_token=current_git_token
                    )
                    logs.append(f"[File] {filename}: {res}")
                except Exception as e:
                    logs.append(f"[Error] Failed to read {full_path}: {e}")

    # 2. Capture Git Diff
    if "project" in scopes_to_process:
        try:
            diff_proc = subprocess.run(["git", "diff", "HEAD"], capture_output=True, text=True)
            if diff_proc.returncode == 0 and diff_proc.stdout.strip():
                diff_content = diff_proc.stdout
                res = store.remember(
                    content=diff_content,
                    topic="context/project/git_diff",
                    tags=["context", "project", "diff"],
                    git_url=current_git_url,
                    git_token=current_git_token
                )
                logs.append(f"[Diff] Git Diff captured: {res}")
        except Exception as e:
            logs.append(f"[Diff] Error: {e}")

    # 3. Trigger Digest once for all updates
    _trigger_digest(current_git_url, current_git_token)

    return "\n".join(logs)

@mcp.tool()
def remember(content: str, topic: str = "global", tags: List[str] = None,
             dependencies: List[str] = None, git_url: str = None, git_token: str = None) -> str:
    """
    Store or update knowledge in a specific Git repository.
    """
    current_git_url = git_url or os.environ.get("GITMEM_URL")
    current_git_token = git_token or os.environ.get("GITMEM_TOKEN")

    if not current_git_url:
        return "Error: GITMEM_URL is missing."

    result = store.remember(content, topic, tags, dependencies, current_git_url, current_git_token)
    
    if "Error:" not in result:
        _trigger_digest(current_git_url, current_git_token)

    return result

# Initialize FastMCP App
app = FastMCP(title="GitMem Server", version="