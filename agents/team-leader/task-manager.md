# Task Manager

The TL dynamically manages the task pool: reassigns failed work, splits
oversized tasks, merges duplicate work, and decides when to escalate to human.

---

## Task Lifecycle

```
PENDING → ENHANCED → RUNNING → DONE
                  ↓         ↓
              SPLIT      CONFLICT → conflict-resolver.md
                  ↓         ↓
            PENDING x2   FAILED → REASSIGNED → RUNNING
                                      ↓
                                  ESCALATED (human)
```

---

## Decision Matrix: What to Do When an Agent Has a Problem

| Situation | Condition | TL Action |
|---|---|---|
| Agent failed once | Retries < 2 | Restart with enhanced prompt |
| Agent failed twice | Retries = 2 | Reassign to different agent |
| Agent failed 3 times | Retries = 3 | Escalate to human |
| Agent timed out | Running > 1h | Kill, split task, restart parts |
| Agent sent `split_request` | Agent self-reports too large | Split into 2–3 sub-tasks |
| Two agents doing same work | Knowledge store detects duplicate | Merge back, cancel one |
| Agent blocked by unmerged dep | `blocker_report` + dep not merged | Queue after dep, don't restart |
| Agent hit conflict | `CONFLICT_REPORT.json` exists | Run conflict-resolver.md |
| Task requires human decision | `human_request` from agent | Notify + pause agent |

---

## Task Registry

```python
# task_manager.py
import json, os, datetime, threading
from pathlib import Path
from audit_log import AuditLog

TASK_REGISTRY = Path("/tmp/tl_task_registry.json")
WT_ROOT       = "/tmp/worktrees"
_lock         = threading.Lock()
audit         = AuditLog(agent_id="team-leader/task-manager")


class TaskRegistry:
    def __init__(self):
        self.tasks = self._load()

    def _load(self) -> dict:
        if TASK_REGISTRY.exists():
            with open(TASK_REGISTRY) as f:
                return json.load(f)
        return {}

    def _save(self):
        with _lock:
            with open(TASK_REGISTRY, "w") as f:
                json.dump(self.tasks, f, indent=2)

    def register(self, ticket_id: str, task: str, agent_role: str = "implementer",
                 depends_on: list = None, original_ticket: str = None):
        self.tasks[ticket_id] = {
            "ticket_id":       ticket_id,
            "task":            task,
            "agent_role":      agent_role,
            "status":          "pending",
            "depends_on":      depends_on or [],
            "original_ticket": original_ticket,   # set if this is a split
            "retries":         0,
            "created_at":      datetime.datetime.utcnow().isoformat() + "Z",
            "enhanced_prompt": None,
        }
        self._save()

    def update_status(self, ticket_id: str, status: str, detail: str = None):
        if ticket_id in self.tasks:
            self.tasks[ticket_id]["status"] = status
            self.tasks[ticket_id]["updated_at"] = datetime.datetime.utcnow().isoformat() + "Z"
            if detail:
                self.tasks[ticket_id]["last_detail"] = detail
            self._save()
            audit.log("workitem_update", target=ticket_id,
                      detail=f"status→{status}" + (f": {detail}" if detail else ""))

    def increment_retry(self, ticket_id: str) -> int:
        if ticket_id in self.tasks:
            self.tasks[ticket_id]["retries"] += 1
            self._save()
            return self.tasks[ticket_id]["retries"]
        return 0

    def get_task(self, ticket_id: str) -> dict | None:
        return self.tasks.get(ticket_id)

    def get_pending(self) -> list[dict]:
        return [t for t in self.tasks.values() if t["status"] == "pending"]

    def get_running(self) -> list[dict]:
        return [t for t in self.tasks.values() if t["status"] == "running"]

    def all_done(self) -> bool:
        terminal = ("done", "escalated", "cancelled")
        return all(t["status"] in terminal for t in self.tasks.values())


registry = TaskRegistry()
```

---

## Split Task

When a task is too large for one agent:

```python
def split_task(ticket_id: str, reason: str, num_parts: int = 2) -> list[str]:
    """
    Split a ticket into N smaller sub-tasks.
    Returns the new ticket IDs.
    The original ticket is marked 'split' and cancelled.
    """
    original = registry.get_task(ticket_id)
    if not original:
        return []

    task_text = original["task"]
    split_ids = []

    # TL generates split descriptions based on the task
    # In production, call LLM here to suggest sensible splits
    splits = _suggest_splits(task_text, num_parts)

    for i, sub_task in enumerate(splits):
        new_id = f"{ticket_id}-PART{i+1}"
        registry.register(
            ticket_id    = new_id,
            task         = sub_task,
            agent_role   = original["agent_role"],
            depends_on   = original["depends_on"],
            original_ticket = ticket_id
        )
        split_ids.append(new_id)
        audit.log("agent_start", target=new_id,
                  detail=f"split from {ticket_id}")

    registry.update_status(ticket_id, "split",
                           detail=f"→ {split_ids}")
    print(f"[TM] Split {ticket_id} into: {split_ids}")

    # Spawn new worktrees for the parts
    _spawn_split_worktrees(split_ids, ticket_id)

    return split_ids


def _suggest_splits(task_text: str, num_parts: int) -> list[str]:
    """
    Suggest how to split a task. Rule-based fallback:
    - Split implementation from tests
    - Split by file/module if multiple mentioned
    In production: enhance with LLM call.
    """
    # Look for natural split points
    if "and" in task_text.lower():
        parts = task_text.split(" and ", 1)
        if len(parts) == num_parts:
            return parts

    if "test" in task_text.lower() and num_parts == 2:
        non_test = task_text.replace("and add tests", "").replace("with tests", "").strip()
        test_part = f"Add tests for: {non_test}"
        return [non_test, test_part]

    # Fallback: split by portion
    words     = task_text.split()
    mid       = len(words) // num_parts
    return [
        " ".join(words[:mid]) + " (first half of original task)",
        " ".join(words[mid:]) + " (second half of original task)"
    ]


def _spawn_split_worktrees(new_ids: list[str], original_id: str):
    """Create worktrees for split tasks, branching from the original's worktree."""
    import subprocess
    bare_dir = _find_bare_repo()
    if not bare_dir:
        return

    for new_id in new_ids:
        wt_path = f"{WT_ROOT}/{new_id}"
        if os.path.exists(wt_path):
            continue
        try:
            subprocess.run([
                "git", "-C", bare_dir, "worktree", "add",
                "-b", f"feature/{new_id}", wt_path,
                f"feature/{original_id}"   # branch from original
            ], check=True)
            audit.log("worktree_create", target=new_id,
                      detail=f"split from {original_id}")
        except Exception as e:
            print(f"[TM] Failed to create worktree for {new_id}: {e}")


def _find_bare_repo() -> str | None:
    """Find the bare clone in /workspace/."""
    import glob
    bare_repos = glob.glob("/workspace/*.git")
    return bare_repos[0] if bare_repos else None
```

---

## Reassign Failed Task

```python
def reassign_task(ticket_id: str, reason: str) -> str | None:
    """
    Reassign a failed task. Creates a fresh worktree with an enhanced prompt.
    Returns the new ticket_id (same ID, fresh start).
    """
    task = registry.get_task(ticket_id)
    if not task:
        return None

    retries = registry.increment_retry(ticket_id)

    if retries > 3:
        escalate_to_human(ticket_id, f"Failed {retries} times: {reason}")
        return None

    print(f"[TM] Reassigning {ticket_id} (attempt {retries+1}): {reason}")

    # Clean the old worktree state (keep the branch, reset to clean state)
    _reset_worktree(ticket_id)

    # Re-enhance the prompt with the failure as additional context
    from prompt_enhancer import enhance_prompt
    original_prompt = task["task"]
    enhanced = enhance_prompt(
        ticket_id  = ticket_id,
        raw_task   = original_prompt + f"\n\nPREVIOUS ATTEMPT FAILED: {reason}\n"
                     f"Pay special attention to avoiding this failure mode.",
        repo_path  = f"{WT_ROOT}/{ticket_id}"
    )

    registry.tasks[ticket_id]["enhanced_prompt"] = enhanced
    registry.update_status(ticket_id, "pending",
                           detail=f"reassigned after failure: {reason}")

    return ticket_id


def _reset_worktree(ticket_id: str):
    """Clean worktree state for a retry — remove result/heartbeat/conflict files."""
    import subprocess
    wt_path = f"{WT_ROOT}/{ticket_id}"
    for fname in ("RESULT.json", "HEARTBEAT.json", "CONFLICT_REPORT.json", "ESCALATION.json"):
        fpath = f"{wt_path}/{fname}"
        if os.path.exists(fpath):
            os.remove(fpath)
    # Reset to clean git state
    try:
        subprocess.run(["git", "-C", wt_path, "checkout", "."], check=True)
        subprocess.run(["git", "-C", wt_path, "clean", "-fd"], check=True)
    except Exception:
        pass
```

---

## Merge Duplicate Work

```python
def merge_duplicate_tasks(ticket_a: str, ticket_b: str):
    """
    Two agents produced the same artifact. Cancel B, keep A.
    Notify the team about what was merged.
    """
    print(f"[TM] Merging duplicate: {ticket_a} and {ticket_b}")

    # Cancel B
    registry.update_status(ticket_b, "cancelled",
                           detail=f"duplicate of {ticket_a}")
    audit.log("pr_abandon", target=ticket_b,
              detail=f"duplicate of {ticket_a}")

    # If B has an open PR, abandon it
    task_b = registry.get_task(ticket_b)
    if task_b and task_b.get("pr_id"):
        import subprocess
        subprocess.run([
            "az", "repos", "pr", "update",
            "--id", task_b["pr_id"],
            "--status", "abandoned"
        ], check=False)

    from notification_templates import notify
    notify(
        title=f"Duplicate work merged: {ticket_b} → {ticket_a}",
        message=f"Both agents produced the same artifact. "
                f"{ticket_b} cancelled, {ticket_a} continues.",
        priority="info"
    )
```

---

## Escalate to Human

```python
def escalate_to_human(ticket_id: str, reason: str, question: str = None):
    """Final escalation: mark task as escalated, notify human."""
    registry.update_status(ticket_id, "escalated", detail=reason)

    pr_url = _get_pr_url(ticket_id)
    message = f"Task {ticket_id} cannot be completed automatically.\n\nReason: {reason}"
    if question:
        message += f"\n\nQuestion for human: {question}"

    from notification_templates import notify
    notify(
        title=f"Human intervention needed — {ticket_id}",
        message=message,
        priority="critical",
        link_url=pr_url,
        link_text="Open PR" if pr_url else None
    )
    audit.log("agent_escalated", target=ticket_id, detail=reason)
    print(f"[TM] {ticket_id} escalated: {reason}")


def _get_pr_url(ticket_id: str) -> str | None:
    task = registry.get_task(ticket_id)
    pr_id = task.get("pr_id") if task else None
    if pr_id:
        org = os.environ.get("ADO_ORG", "")
        proj = os.environ.get("ADO_PROJECT", "")
        return f"{org}/{proj}/_git/REPO/pullrequest/{pr_id}"
    return None
```

---

## TL Main Loop: Event-Driven Manager

```python
# team_leader.py — run the full TL loop
import time, os, threading
from pathlib import Path
from knowledge_store import KnowledgeStore
from message_broker  import process_inbox
from task_manager    import registry, reassign_task, escalate_to_human

WT_ROOT = "/tmp/worktrees"
store   = KnowledgeStore()


def run_team_leader(active_tickets: list[str], spawn_fn, poll_sec: int = 15):
    """
    Main TL event loop. Runs alongside sub-agents.

    spawn_fn(ticket_id, enhanced_prompt) — starts a sub-agent
    """
    # Start broker in background
    broker_thread = threading.Thread(target=process_inbox, daemon=True)
    broker_thread.start()

    print(f"[TL] Team Leader active for {len(active_tickets)} tickets")

    while not registry.all_done():
        for ticket_id in active_tickets:
            task = registry.get_task(ticket_id)
            if not task or task["status"] in ("done", "escalated", "cancelled", "split"):
                continue

            result_f     = f"{WT_ROOT}/{ticket_id}/RESULT.json"
            conflict_f   = f"{WT_ROOT}/{ticket_id}/CONFLICT_REPORT.json"
            escalation_f = f"{WT_ROOT}/{ticket_id}/ESCALATION.json"

            # Agent completed successfully
            if os.path.exists(result_f) and task["status"] != "done":
                registry.update_status(ticket_id, "done")
                store.ingest_result(ticket_id, result_f)
                conflicts = store.check_conflicts()
                if conflicts:
                    print(f"[TL] Knowledge conflicts: {conflicts}")
                print(f"[TL] ✓ {ticket_id} done")

            # Agent hit a conflict
            elif os.path.exists(conflict_f) and task["status"] != "conflict":
                registry.update_status(ticket_id, "conflict")
                # Conflict-resolver handles it — TL just tracks the state
                print(f"[TL] ⚠ {ticket_id} has a conflict — conflict-resolver.md")

            # Agent was escalated by watchdog
            elif os.path.exists(escalation_f) and task["status"] != "escalated":
                with open(escalation_f) as f:
                    esc = json.load(f)
                reassign_task(ticket_id, esc.get("reason", "escalated by watchdog"))

            # Agent needs reassignment (task_manager moved it back to pending)
            elif task["status"] == "pending" and task.get("enhanced_prompt"):
                registry.update_status(ticket_id, "running")
                spawn_fn(ticket_id, task["enhanced_prompt"])

        time.sleep(poll_sec)

    print(f"[TL] All tasks terminal. Generating summary...")
    print(store.get_team_summary())
```
