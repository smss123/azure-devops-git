# Long Task Management — Checkpoint & Resume Pattern

Use this for any ADO task that may exceed context limits, take >5 minutes,
or involve steps that could fail partway through.

---

## Core Principle: Write-Before-Act

Before doing any destructive or state-changing operation:
1. Write intent to checkpoint
2. Perform action
3. Write result to checkpoint

This allows resuming from exactly where it failed.

---

## Checkpoint File Schema

```json
{
  "task_id": "migrate-branches-2025-03-14",
  "task_type": "branch_migration",
  "created_at": "2025-03-14T09:00:00Z",
  "updated_at": "2025-03-14T09:15:32Z",
  "status": "in_progress",
  "config": {
    "source_branch": "master",
    "target_branch": "main",
    "repos": ["repo-a", "repo-b", "repo-c"]
  },
  "items": [
    {"id": "repo-a", "status": "done",    "result": "PR #42 created"},
    {"id": "repo-b", "status": "done",    "result": "PR #17 created"},
    {"id": "repo-c", "status": "pending", "result": null},
  ],
  "summary": {"done": 2, "pending": 1, "failed": 0}
}
```

---

## Python Checkpoint Manager

```python
# checkpoint.py
import json, os, datetime
from pathlib import Path

CHECKPOINT_DIR = Path("/tmp/ado_tasks")
CHECKPOINT_DIR.mkdir(exist_ok=True)

class TaskCheckpoint:
    def __init__(self, task_id: str):
        self.task_id = task_id
        self.path = CHECKPOINT_DIR / f"{task_id}.json"
        self.data = self._load()

    def _load(self) -> dict:
        if self.path.exists():
            with open(self.path) as f:
                data = json.load(f)
            print(f"[RESUME] Loaded checkpoint: {data['summary']}")
            return data
        return {"task_id": self.task_id, "items": [], "summary": {}}

    def save(self):
        self.data["updated_at"] = datetime.datetime.now(datetime.timezone.utc).isoformat().replace("+00:00", "Z")
        with open(self.path, "w") as f:
            json.dump(self.data, f, indent=2)

    def get_pending(self) -> list:
        """Return only item IDs not yet successfully completed."""
        # `self.data["items"]` tracks progress; `self.data["config"]["items"]` is the full list.
        done = {i["id"] for i in self.data.get("items", []) if i.get("status") == "done"}
        all_items = self.data.get("config", {}).get("items", [])
        return [x for x in all_items if x not in done]

    def mark_done(self, item_id: str, result: str):
        # Update or append
        for item in self.data["items"]:
            if item["id"] == item_id:
                item.update({"status": "done", "result": result})
                break
        else:
            self.data["items"].append({"id": item_id, "status": "done", "result": result})
        self._update_summary()
        self.save()

    def mark_failed(self, item_id: str, error: str):
        for item in self.data["items"]:
            if item["id"] == item_id:
                item.update({"status": "failed", "error": error})
                break
        else:
            self.data["items"].append({"id": item_id, "status": "failed", "error": error})
        self._update_summary()
        self.save()

    def _update_summary(self):
        statuses = [i["status"] for i in self.data["items"]]
        self.data["summary"] = {s: statuses.count(s) for s in set(statuses)}
```

## Long Task Execution Loop

```python
# long_task_runner.py
import time
from checkpoint import TaskCheckpoint

def run_with_checkpoint(task_id: str, all_items: list, process_fn, config: dict = {}):
    """
    Run a long task with full checkpoint/resume support.
    
    process_fn(item) -> str (result message) or raises Exception
    """
    cp = TaskCheckpoint(task_id)
    cp.data.setdefault("config", {}).update({"items": all_items, **config})
    cp.save()

    pending = cp.get_pending()
    total = len(all_items)
    done_count = total - len(pending)

    print(f"[TASK] {task_id}: {done_count}/{total} already done, {len(pending)} remaining")

    for item in pending:
        print(f"  [→] Processing: {item}")
        try:
            result = process_fn(item)
            cp.mark_done(item, result)
            print(f"  [✓] Done: {item} — {result}")
        except Exception as e:
            cp.mark_failed(item, str(e))
            print(f"  [✗] Failed: {item} — {e}")
        time.sleep(0.3)   # ADO rate limit buffer

    cp.data["status"] = "completed"
    cp.save()

    summary = cp.data["summary"]
    print(f"\n[COMPLETE] {task_id}: {summary}")
    return cp.data

# Example usage:
if __name__ == "__main__":
    repos = ["repo-alpha", "repo-beta", "repo-gamma", "repo-delta"]

    def migrate_branch(repo_name):
        # Your actual ADO operation here
        import subprocess
        result = subprocess.run(
            ["az", "repos", "pr", "create",
             "--repository", repo_name,
             "--source-branch", "master",
             "--target-branch", "main",
             "--title", "chore: rename master to main [automated]",
             "--auto-complete", "true", "--squash", "true"],
            capture_output=True, text=True
        )
        if result.returncode != 0:
            raise RuntimeError(result.stderr)
        return f"PR created"

    run_with_checkpoint(
        task_id="rename-master-to-main-2025-03-14",
        all_items=repos,
        process_fn=migrate_branch
    )
```

---

## Progress Reporting for Long Tasks

Always report progress so the user knows the task is alive:

```python
def progress_bar(done: int, total: int, width: int = 40) -> str:
    pct = done / total if total > 0 else 0
    filled = int(width * pct)
    bar = "█" * filled + "░" * (width - filled)
    return f"[{bar}] {done}/{total} ({pct:.0%})"

# Print every N items
if done_count % 5 == 0:
    print(progress_bar(done_count, total))
```

---

## Safety Checklist for Long Tasks

Before running any long destructive task:

- [ ] Checkpoint file path confirmed writable
- [ ] Dry-run mode tested first (`--dry-run` flag in your script)
- [ ] Rollback plan documented (can you undo this?)
- [ ] Rate limiting in place (`time.sleep(0.3)` minimum)
- [ ] Error handling logs but doesn't abort entire batch
- [ ] Sub-agents write to separate result files (no conflicts)
- [ ] Final merge step collects all chunk results
