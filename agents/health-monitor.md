# Agent Health Monitor — Watchdog Pattern

Sub-agents can crash silently, hang on git locks, stall mid-rebase, or exceed
context limits. Without a watchdog, the orchestrator waits forever. This file
provides the complete watchdog pattern.

---

## Health Status Protocol

Every sub-agent MUST write a heartbeat file every 60 seconds while working.
The orchestrator reads these files to detect stalls and crashes.

### Heartbeat file: `/tmp/worktrees/{TICKET}/HEARTBEAT.json`

```json
{
  "ticket": "TICKET-123",
  "pid": 98234,
  "status": "working",
  "phase": "rebasing",
  "progress": 3,
  "total": 8,
  "last_commit": "abc1234",
  "last_update": "2025-03-14T10:23:45Z",
  "started_at": "2025-03-14T10:00:00Z"
}
```

Valid `status` values: `starting` | `working` | `committing` | `pushing` | `done` | `error` | `conflict`

Valid `phase` values: `fetch` | `rebase` | `coding` | `testing` | `committing` | `pushing` | `pr_open` | `done`

---

## Sub-Agent Heartbeat Writer

Add this to every sub-agent script — call `heartbeat()` in your main loop:

```python
# heartbeat.py — import and call from sub-agents
import json, os, datetime, threading, time

TICKET_ID = os.environ["TICKET_ID"]
WT_PATH   = f"/tmp/worktrees/{TICKET_ID}"
HB_FILE   = f"{WT_PATH}/HEARTBEAT.json"

_state = {
    "ticket": TICKET_ID,
    "pid": os.getpid(),
    "status": "starting",
    "phase": "fetch",
    "progress": 0,
    "total": 0,
    "last_commit": None,
    "started_at": datetime.datetime.utcnow().isoformat() + "Z"
}

def heartbeat(status=None, phase=None, progress=None, total=None, last_commit=None):
    """Update heartbeat file. Call at every major step."""
    if status:     _state["status"] = status
    if phase:      _state["phase"] = phase
    if progress:   _state["progress"] = progress
    if total:      _state["total"] = total
    if last_commit: _state["last_commit"] = last_commit
    _state["last_update"] = datetime.datetime.utcnow().isoformat() + "Z"

    os.makedirs(WT_PATH, exist_ok=True)
    with open(HB_FILE, "w") as f:
        json.dump(_state, f)

def start_background_heartbeat(interval_sec=30):
    """Writes heartbeat every N seconds automatically (no-op ping)."""
    def _loop():
        while _state["status"] not in ("done", "error"):
            heartbeat()
            time.sleep(interval_sec)
    t = threading.Thread(target=_loop, daemon=True)
    t.start()
    return t

# Usage in sub-agent:
# from heartbeat import heartbeat, start_background_heartbeat
# start_background_heartbeat()
# heartbeat(status="working", phase="rebase", progress=1, total=5)
```

---

## Orchestrator Watchdog

Run this in a background thread while sub-agents are active:

```python
# watchdog.py
import json, os, glob, time, datetime, subprocess
from typing import Callable

WT_ROOT        = "/tmp/worktrees"
STALL_TIMEOUT  = 300   # seconds — agent is stalled if no heartbeat update
TOTAL_TIMEOUT  = 3600  # seconds — agent is dead if running longer than 1 hour

def read_heartbeat(ticket_id: str) -> dict | None:
    hb_path = f"{WT_ROOT}/{ticket_id}/HEARTBEAT.json"
    if not os.path.exists(hb_path):
        return None
    try:
        with open(hb_path) as f:
            return json.load(f)
    except Exception:
        return None

def seconds_since(iso_timestamp: str) -> float:
    dt = datetime.datetime.fromisoformat(iso_timestamp.replace("Z", "+00:00"))
    return (datetime.datetime.now(datetime.timezone.utc) - dt).total_seconds()

def agent_is_stalled(hb: dict) -> bool:
    if not hb or "last_update" not in hb:
        return True
    return seconds_since(hb["last_update"]) > STALL_TIMEOUT

def agent_is_timed_out(hb: dict) -> bool:
    if not hb or "started_at" not in hb:
        return False
    return seconds_since(hb["started_at"]) > TOTAL_TIMEOUT

def kill_agent(ticket_id: str, hb: dict):
    """Kill a hung agent process."""
    pid = hb.get("pid") if hb else None
    if pid:
        try:
            os.kill(pid, 9)
            print(f"[WATCHDOG] Killed PID {pid} for {ticket_id}")
        except ProcessLookupError:
            pass   # already dead

def restart_agent(ticket_id: str, spawn_fn: Callable):
    """Restart a failed agent from its last checkpoint."""
    print(f"[WATCHDOG] Restarting {ticket_id}")
    # Reset heartbeat
    hb_path = f"{WT_ROOT}/{ticket_id}/HEARTBEAT.json"
    if os.path.exists(hb_path):
        os.remove(hb_path)
    spawn_fn(ticket_id)

def run_watchdog(
    active_tickets: list,
    spawn_fn: Callable,
    poll_interval: int = 30,
    max_restarts: int = 2
):
    """
    Main watchdog loop. Call from orchestrator in a background thread.

    spawn_fn(ticket_id) — function that starts a sub-agent for the given ticket
    """
    restart_counts = {t: 0 for t in active_tickets}
    escalated = set()

    while True:
        all_done = True
        now = datetime.datetime.utcnow().isoformat() + "Z"

        for ticket_id in active_tickets:
            result_file = f"{WT_ROOT}/{ticket_id}/RESULT.json"
            if os.path.exists(result_file):
                continue   # this agent is done
            all_done = False

            hb = read_heartbeat(ticket_id)

            if hb and hb.get("status") in ("done", "error", "conflict"):
                continue   # terminal state, orchestrator handles it

            stalled   = agent_is_stalled(hb)
            timed_out = agent_is_timed_out(hb)

            if timed_out or stalled:
                phase = hb.get("phase", "unknown") if hb else "no heartbeat"
                reason = "timeout" if timed_out else "stall"
                print(f"[WATCHDOG] {ticket_id} {reason} at phase={phase}")

                if ticket_id in escalated:
                    continue   # already escalated, skip

                if restart_counts[ticket_id] < max_restarts:
                    kill_agent(ticket_id, hb)
                    time.sleep(2)
                    restart_agent(ticket_id, spawn_fn)
                    restart_counts[ticket_id] += 1
                    print(f"[WATCHDOG] {ticket_id} restart #{restart_counts[ticket_id]}")
                else:
                    # Max restarts exceeded — escalate to human
                    escalated.add(ticket_id)
                    write_escalation(ticket_id, hb, reason)
                    print(f"[WATCHDOG] {ticket_id} escalated after {max_restarts} restarts")

        if all_done:
            print("[WATCHDOG] All agents done — shutting down watchdog")
            break

        time.sleep(poll_interval)


def write_escalation(ticket_id: str, hb: dict | None, reason: str):
    import json, datetime
    report = {
        "ticket": ticket_id,
        "reason": reason,
        "last_heartbeat": hb,
        "timestamp": datetime.datetime.utcnow().isoformat() + "Z",
        "action_required": "human_intervention"
    }
    with open(f"{WT_ROOT}/{ticket_id}/ESCALATION.json", "w") as f:
        json.dump(report, f, indent=2)
```

---

## Orchestrator Integration

```python
import threading
from watchdog import run_watchdog

# After spawning all agents:
active_tickets = ["TICKET-101", "TICKET-102", "TICKET-103"]

watchdog_thread = threading.Thread(
    target=run_watchdog,
    args=(active_tickets, spawn_agent_fn),
    kwargs={"poll_interval": 30, "max_restarts": 2},
    daemon=True
)
watchdog_thread.start()

# Wait for all agents to finish (poll RESULT.json files)
wait_for_all_agents(active_tickets)

# After done, collect escalations
escalated = []
for ticket in active_tickets:
    esc = f"{WT_ROOT}/{ticket}/ESCALATION.json"
    if os.path.exists(esc):
        with open(esc) as f:
            escalated.append(json.load(f))

if escalated:
    print(f"[WARN] {len(escalated)} tickets need human attention:")
    for e in escalated:
        print(f"  {e['ticket']}: {e['reason']}")
```

---

## Health Dashboard (terminal)

```python
def print_health_dashboard(active_tickets: list):
    """Print a live status table for all running agents."""
    import datetime

    print(f"\n{'TICKET':<20} {'STATUS':<12} {'PHASE':<12} {'PROGRESS':<12} {'LAST UPDATE'}")
    print("─" * 75)

    for ticket in active_tickets:
        hb = read_heartbeat(ticket)
        result = f"{WT_ROOT}/{ticket}/RESULT.json"
        escalation = f"{WT_ROOT}/{ticket}/ESCALATION.json"

        if os.path.exists(result):
            print(f"{ticket:<20} {'✓ done':<12} {'—':<12} {'—':<12}")
            continue
        if os.path.exists(escalation):
            print(f"{ticket:<20} {'✗ ESCALATED':<12} {'—':<12} {'—':<12}")
            continue
        if not hb:
            print(f"{ticket:<20} {'? no hb':<12} {'—':<12} {'—':<12}")
            continue

        age = int(seconds_since(hb["last_update"]))
        progress_str = f"{hb.get('progress',0)}/{hb.get('total',0)}"
        staleness = " ⚠️" if age > STALL_TIMEOUT else ""
        print(f"{ticket:<20} {hb['status']:<12} {hb['phase']:<12} {progress_str:<12} {age}s ago{staleness}")
```

---

## Common Hang Causes & Fixes

| Symptom | Likely Cause | Fix |
|---|---|---|
| Stalled at phase=`rebase` | Git conflict waiting for human input | Kill + conflict-resolver.md |
| Stalled at phase=`pushing` | ADO rate limit / network timeout | Kill + restart — push is idempotent |
| Stalled at phase=`testing` | Test suite hung (infinite loop, port busy) | Kill + investigate test output |
| No heartbeat file at all | Agent crashed before first heartbeat | Check RESULT.json, check stderr log |
| Heartbeat present but PID dead | Agent process was killed externally | Restart from checkpoint |
| Rapid restarts, always fails | Fundamental code error (syntax etc.) | Escalate immediately |
