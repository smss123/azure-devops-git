# Agent Audit Log

Every agent action that changes state (commits, pushes, PR creation, work item
updates, merges) must be written to the audit log. This enables post-hoc review,
incident investigation, and compliance reporting.

---

## Audit Log Schema

```json
{
  "event_id":    "evt_20250314_102345_abc123",
  "timestamp":   "2025-03-14T10:23:45.123Z",
  "agent_id":    "orchestrator | sub-agent-TICKET-123 | watchdog",
  "ticket_id":   "TICKET-123",
  "action":      "git_push",
  "target":      "feature/TICKET-123",
  "status":      "success | failed | skipped",
  "detail":      "pushed 3 commits to origin/feature/TICKET-123",
  "sha":         "abc1234",
  "pr_id":       null,
  "duration_ms": 1240,
  "error":       null
}
```

### Action vocabulary

| action | When to use |
|---|---|
| `worktree_create` | git worktree add |
| `worktree_close` | git worktree remove |
| `branch_create` | new branch pushed |
| `git_commit` | commit made |
| `git_push` | branch pushed to remote |
| `git_rebase` | rebase executed |
| `pr_create` | PR opened |
| `pr_merge` | PR merged/completed |
| `pr_abandon` | PR abandoned |
| `conflict_detected` | merge conflict found |
| `conflict_resolved` | conflict fixed by agent |
| `conflict_escalated` | conflict needs human |
| `agent_start` | sub-agent started |
| `agent_done` | sub-agent finished |
| `agent_restart` | watchdog restarted agent |
| `agent_escalated` | watchdog gave up on agent |
| `rollback_revert` | git revert executed |
| `workitem_update` | ADO work item updated |
| `policy_drift` | policy violation found |

---

## AuditLog Class

```python
# audit_log.py
import json, uuid, time, datetime, os, stat, threading
from pathlib import Path

# AUDIT_LOG_DIR: override in production to a path with restricted permissions.
# /tmp is world-readable on most systems — suitable for dev/CI only.
# In production, use a dedicated directory owned by the agent user (chmod 700).
AUDIT_DIR  = Path(os.environ.get("AUDIT_LOG_DIR", "/tmp/ado_audit"))
AUDIT_DIR.mkdir(mode=0o700, exist_ok=True)
AUDIT_FILE = AUDIT_DIR / f"audit_{datetime.date.today().isoformat()}.jsonl"

_lock = threading.Lock()   # thread-safe for parallel agents

class AuditLog:
    def __init__(self, agent_id: str, ticket_id: str = None):
        self.agent_id  = agent_id
        self.ticket_id = ticket_id

    def log(
        self,
        action:   str,
        target:   str = None,
        status:   str = "success",
        detail:   str = None,
        sha:      str = None,
        pr_id:    str = None,
        error:    str = None,
        duration_ms: int = None,
    ) -> dict:
        entry = {
            "event_id":    f"evt_{datetime.datetime.now(datetime.timezone.utc).strftime('%Y%m%d_%H%M%S')}_{uuid.uuid4().hex[:6]}",
            "timestamp":   datetime.datetime.now(datetime.timezone.utc).isoformat().replace("+00:00", "Z"),
            "agent_id":    self.agent_id,
            "ticket_id":   self.ticket_id,
            "action":      action,
            "target":      target,
            "status":      status,
            "detail":      detail,
            "sha":         sha,
            "pr_id":       pr_id,
            "duration_ms": duration_ms,
            "error":       error,
        }
        # Remove None values for cleaner logs
        entry = {k: v for k, v in entry.items() if v is not None}

        with _lock:
            with open(AUDIT_FILE, "a") as f:
                f.write(json.dumps(entry) + "\n")

        return entry

    def timed(self, action: str, target: str = None):
        """Context manager that auto-times and logs an action."""
        return _TimedAction(self, action, target)

class _TimedAction:
    def __init__(self, audit: AuditLog, action: str, target: str):
        self.audit = audit
        self.action = action
        self.target = target
        self.start = None

    def __enter__(self):
        self.start = time.monotonic()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        elapsed_ms = int((time.monotonic() - self.start) * 1000)
        if exc_type:
            self.audit.log(
                action=self.action, target=self.target,
                status="failed", error=str(exc_val),
                duration_ms=elapsed_ms
            )
        else:
            self.audit.log(
                action=self.action, target=self.target,
                status="success", duration_ms=elapsed_ms
            )
        return False   # don't suppress exceptions
```

---

## Usage in Sub-Agent

```python
# In sub_agent_work.py
from audit_log import AuditLog

audit = AuditLog(agent_id=f"sub-agent-{TICKET_ID}", ticket_id=TICKET_ID)

# Log a simple action
audit.log("agent_start", target=WT_PATH, detail=f"starting work on {TICKET_ID}")

# Log with timing
with audit.timed("git_rebase", target="origin/develop"):
    subprocess.run(["git", "-C", WT_PATH, "rebase", "origin/develop"], check=True)

# Log a commit
sha = git("rev-parse --short HEAD")
audit.log("git_commit", target=BRANCH, sha=sha, detail=commit_message)

# Log a push
audit.log("git_push", target=BRANCH, sha=sha)

# Log PR creation
audit.log("pr_create", pr_id=str(pr_id), target=BRANCH,
          detail=f"PR targeting develop, size={size_label}")

# Log completion
audit.log("agent_done", target=WT_PATH, detail=f"all work complete for {TICKET_ID}")
```

---

## Audit Log Query Tool

```python
# query_audit.py — investigate what happened
import json, sys
from pathlib import Path

AUDIT_DIR = Path("/tmp/ado_audit")

def load_logs(date: str = None) -> list[dict]:
    """Load audit entries for a date (YYYY-MM-DD) or all dates."""
    entries = []
    pattern = f"audit_{date}.jsonl" if date else "audit_*.jsonl"
    for f in sorted(AUDIT_DIR.glob(pattern)):
        for line in open(f):
            try:
                entries.append(json.loads(line))
            except json.JSONDecodeError as e:
                print(f"[AUDIT] Skipping malformed log entry in {f.name}: {e}", flush=True)
    return entries

def query(
    ticket_id: str = None,
    action: str = None,
    status: str = None,
    agent_id: str = None,
    date: str = None
) -> list[dict]:
    entries = load_logs(date)
    if ticket_id: entries = [e for e in entries if e.get("ticket_id") == ticket_id]
    if action:    entries = [e for e in entries if e.get("action") == action]
    if status:    entries = [e for e in entries if e.get("status") == status]
    if agent_id:  entries = [e for e in entries if e.get("agent_id") == agent_id]
    return entries

def print_timeline(ticket_id: str):
    """Print a timeline of all actions for a ticket."""
    entries = query(ticket_id=ticket_id)
    print(f"\nTimeline for {ticket_id}:")
    print(f"{'TIME':<25} {'ACTION':<20} {'STATUS':<10} {'DETAIL'}")
    print("─" * 85)
    for e in entries:
        t = e["timestamp"][11:19]   # HH:MM:SS
        print(f"{t:<25} {e['action']:<20} {e['status']:<10} {e.get('detail','')[:40]}")

def summarise_run(date: str = None) -> dict:
    """Summary statistics for a run."""
    entries = load_logs(date)
    by_status = {}
    by_action = {}
    errors = []

    for e in entries:
        by_status[e["status"]] = by_status.get(e["status"], 0) + 1
        by_action[e["action"]] = by_action.get(e["action"], 0) + 1
        if e["status"] == "failed":
            errors.append(e)

    return {
        "total_events": len(entries),
        "by_status": by_status,
        "by_action": by_action,
        "errors": errors
    }
```

---

## Compliance Report Export

```python
def export_compliance_report(date_from: str, date_to: str, output_path: str):
    """
    Export a compliance report showing all state-changing actions
    for auditors / security review.
    """
    import csv

    REPORTABLE_ACTIONS = {
        "pr_create", "pr_merge", "pr_abandon",
        "branch_create", "git_push",
        "conflict_resolved", "conflict_escalated",
        "rollback_revert", "workitem_update"
    }

    all_entries = []
    for date_str in _date_range(date_from, date_to):
        all_entries.extend(load_logs(date_str))

    reportable = [e for e in all_entries if e.get("action") in REPORTABLE_ACTIONS]

    with open(output_path, "w", newline="") as f:
        writer = csv.DictWriter(f, fieldnames=[
            "timestamp", "agent_id", "ticket_id", "action",
            "target", "status", "sha", "pr_id", "detail"
        ])
        writer.writeheader()
        for e in reportable:
            writer.writerow({k: e.get(k, "") for k in writer.fieldnames})

    print(f"[AUDIT] Exported {len(reportable)} reportable events to {output_path}")

def _date_range(from_date: str, to_date: str):
    from datetime import date, timedelta
    start = date.fromisoformat(from_date)
    end   = date.fromisoformat(to_date)
    while start <= end:
        yield start.isoformat()
        start += timedelta(days=1)
```
