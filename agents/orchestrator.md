# Sub-Agent Orchestrator — Azure DevOps

Use this pattern when a task spans multiple repos, many PRs, or requires parallel work
that would be too slow or risky to do sequentially in one context.

This file is the **central integration hub** — it wires together all other agent files.
Read the relevant specialist file for each concern, then use this file for the coordination layer.

---

## Integration Map

```
orchestrator.md  (YOU ARE HERE)
        │
        ├── spawns ──► sub-agents (each with own worktree — git-worktree.md)
        │                   │
        │                   ├── writes heartbeats ──► health-monitor.md (watchdog)
        │                   ├── writes audit events ──► audit-log.md
        │                   ├── hits conflicts ──► conflict-resolver.md
        │                   └── writes RESULT.json
        │
        ├── orders by dependency ──► dependency-ordering.md
        ├── needs multi-repo atomic ──► multi-repo-saga.md
        ├── long task resilience ──► long-tasks.md
        └── final: PR automation ──► pr-review-automation.md
```

---

## When to Spawn Sub-Agents

| Situation | Sub-Agent Strategy | Max parallel |
|---|---|---|
| Apply change to N repos | 1 sub-agent per repo | 10 |
| Create PRs across many repos | 1 sub-agent per repo | 10 |
| Audit branch policies org-wide | 1 agent per project | 5 |
| Mass work item updates | 1 agent per 50 items | 8 |
| Sprint: implement N tickets | 1 agent per ticket | 6 |
| Pipeline health check | 1 agent per pipeline group | 5 |
| Multi-repo atomic change | use multi-repo-saga.md | 1 saga |

**Token budget rule:** Each sub-agent should handle work that fits in ~4000 tokens of output.
Anything larger — split into smaller chunks or use the long-tasks checkpoint pattern.

---

## Full Orchestrator Class

```python
# orchestrator.py — complete orchestrator with all integrations
import json, os, time, threading, datetime
from pathlib import Path
from typing import Callable

from audit_log        import AuditLog
from health_monitor   import run_watchdog
from dependency_ordering import build_start_order
from long_tasks       import TaskCheckpoint

WT_ROOT  = "/tmp/worktrees"
ORG_DIR  = "/tmp/ado_orchestration"
Path(ORG_DIR).mkdir(exist_ok=True)

audit = AuditLog(agent_id="orchestrator")


class Orchestrator:
    def __init__(self, task_id: str, tickets: list[dict], spawn_fn: Callable):
        """
        tickets: [{"ticket": "TICKET-101", "repo": "my-repo", "task": "description"}]
        spawn_fn(ticket_spec) → starts a sub-agent for that ticket
        """
        self.task_id  = task_id
        self.tickets  = tickets
        self.spawn_fn = spawn_fn
        self.cp       = TaskCheckpoint(task_id)
        self.cp.data.setdefault("config", {})["tickets"] = [t["ticket"] for t in tickets]
        self.cp.save()

    # ── Partition ──────────────────────────────────────────────────────────

    def build_execution_plan(self) -> list[list[dict]]:
        """
        Sort tickets by dependency (topological), group into parallel batches.
        Falls back to single flat batch if no DEPS.json files exist.
        """
        try:
            ticket_ids = [t["ticket"] for t in self.tickets]
            ordered_ids = build_start_order(ticket_ids)   # [[A], [B,C], [D]]
            id_to_spec  = {t["ticket"]: t for t in self.tickets}
            return [[id_to_spec[tid] for tid in batch] for batch in ordered_ids]
        except Exception as e:
            print(f"[ORCH] Dependency sort failed ({e}) — running all in parallel")
            return [self.tickets]

    # ── Execute ────────────────────────────────────────────────────────────

    def run(self, max_parallel: int = 6) -> dict:
        """Run all batches in dependency order. Returns final summary."""
        plan = self.build_execution_plan()
        audit.log("agent_start", target=self.task_id,
                  detail=f"{len(self.tickets)} tickets, {len(plan)} batches")

        all_results = {"succeeded": [], "failed": [], "escalated": []}

        for batch_num, batch in enumerate(plan):
            print(f"\n[ORCH] Batch {batch_num+1}/{len(plan)}: {[t['ticket'] for t in batch]}")
            batch_results = self._run_batch(batch, max_parallel)

            for r in batch_results:
                all_results[r["status"]].append(r)

            # Stop if any escalation in this batch — don't start next batch
            if any(r["status"] == "escalated" for r in batch_results):
                print(f"[ORCH] Escalation detected — pausing execution")
                break

        self._write_final_report(all_results)
        audit.log("agent_done", target=self.task_id,
                  detail=f"succeeded={len(all_results['succeeded'])} "
                         f"failed={len(all_results['failed'])} "
                         f"escalated={len(all_results['escalated'])}")
        return all_results

    def _run_batch(self, batch: list[dict], max_parallel: int) -> list[dict]:
        """Spawn all tickets in one batch, watch them, collect results."""

        # Cap parallelism
        active = batch[:max_parallel]
        queued = batch[max_parallel:]

        for spec in active:
            self.spawn_fn(spec)
            audit.log("agent_start", target=spec["ticket"],
                      detail=f"worktree={WT_ROOT}/{spec['ticket']}")

        # Start watchdog for this batch
        active_ids = [s["ticket"] for s in active]
        wd = threading.Thread(
            target=run_watchdog,
            args=(active_ids, lambda tid: self.spawn_fn({"ticket": tid})),
            kwargs={"poll_interval": 30, "max_restarts": 2},
            daemon=True
        )
        wd.start()

        # Wait for completion
        self._wait_for_all(active_ids)

        # Handle queued overflow (sequential)
        for spec in queued:
            self.spawn_fn(spec)
            self._wait_for_all([spec["ticket"]])

        # Collect
        return self._collect_results([s["ticket"] for s in batch])

    def _wait_for_all(self, ticket_ids: list, timeout_sec: int = 3600):
        deadline = time.time() + timeout_sec
        while time.time() < deadline:
            pending = [
                tid for tid in ticket_ids
                if not os.path.exists(f"{WT_ROOT}/{tid}/RESULT.json")
                and not os.path.exists(f"{WT_ROOT}/{tid}/ESCALATION.json")
            ]
            if not pending:
                return
            time.sleep(15)
        raise TimeoutError(f"Batch timed out: {ticket_ids}")

    def _collect_results(self, ticket_ids: list) -> list[dict]:
        results = []
        for tid in ticket_ids:
            result_f    = f"{WT_ROOT}/{tid}/RESULT.json"
            escalation_f = f"{WT_ROOT}/{tid}/ESCALATION.json"
            conflict_f  = f"{WT_ROOT}/{tid}/CONFLICT_REPORT.json"

            if os.path.exists(escalation_f):
                with open(escalation_f) as f:
                    r = json.load(f)
                r["status"] = "escalated"
                results.append(r)
            elif os.path.exists(conflict_f):
                with open(conflict_f) as f:
                    r = json.load(f)
                r["status"] = "failed"
                results.append(r)
            elif os.path.exists(result_f):
                with open(result_f) as f:
                    r = json.load(f)
                results.append(r)
            else:
                results.append({"ticket": tid, "status": "failed",
                                 "error": "no RESULT.json found"})
        return results

    def _write_final_report(self, results: dict):
        report = {
            "task_id":   self.task_id,
            "timestamp": datetime.datetime.now(datetime.timezone.utc).isoformat().replace("+00:00", "Z"),
            "summary":   {k: len(v) for k, v in results.items()},
            "details":   results
        }
        with open(f"{ORG_DIR}/{self.task_id}_report.json", "w") as f:
            json.dump(report, f, indent=2)
        print(f"\n[ORCH] Report: {ORG_DIR}/{self.task_id}_report.json")
```

---

## Sub-Agent Prompt Template (canonical)

Give every sub-agent this exact context block when spawning:

```
You are sub-agent for ticket {TICKET_ID}.

ENVIRONMENT:
  ADO_ORG:     {ORG}
  ADO_PROJECT: {PROJECT}
  ADO_PAT:     [already set in env]

YOUR WORKSPACE:
  Worktree path: /tmp/worktrees/{TICKET_ID}
  Branch:        feature/{TICKET_ID}
  Bare repo:     /workspace/{REPO}.git

TASK: {TASK_DESCRIPTION}

RULES (non-negotiable):
  1. Work ONLY in your worktree path — never touch other agents' paths
  2. Only push to your own branch: feature/{TICKET_ID}
  3. NEVER push to develop, main, release/*, or hotfix/*
  4. Write /tmp/worktrees/{TICKET_ID}/HEARTBEAT.json every 60s
  5. Write /tmp/worktrees/{TICKET_ID}/RESULT.json when done
  6. On conflict: write CONFLICT_REPORT.json, set status="conflict", stop

RESULT.json schema:
  {"ticket": "{TICKET_ID}", "branch": "feature/{TICKET_ID}",
   "status": "done|error|conflict", "commit": "<sha>", "pr_id": "<id>",
   "detail": "<summary of what was done>"}

USE: agents/decision-engine.md for commit/push/PR timing decisions
USE: agents/conflict-resolver.md if you hit a merge conflict
```

---

## Parallel Repo Update (quick pattern)

For simple bulk changes that don't need full orchestrator overhead:

```bash
#!/bin/bash
# parallel_repo_update.sh
set -euo pipefail
REPOS=$(az repos list --query "[].name" -o tsv)
MAX_PARALLEL=5
PIDS=()

for REPO in $REPOS; do
  (
    echo "[START] $REPO"
    git clone --quiet \
      -c "url.https://anything:$ADO_PAT@dev.azure.com.insteadOf=https://dev.azure.com" \
      "https://dev.azure.com/$ORG/$PROJECT/_git/$REPO" "/tmp/$REPO"
    cd "/tmp/$REPO"
    # === YOUR CHANGE HERE ===
    cp /path/to/.gitignore .
    git diff --quiet && { echo "[SKIP] $REPO"; exit 0; }
    git add .gitignore
    git commit -m "chore: update .gitignore [automated]"
    git push
    # === END CHANGE ===
    echo "[DONE] $REPO"
  ) &
  PIDS+=($!)
  if (( ${#PIDS[@]} >= MAX_PARALLEL )); then
    wait "${PIDS[0]}"; PIDS=("${PIDS[@]:1}")
  fi
done
for PID in "${PIDS[@]}"; do wait "$PID"; done
echo "All repos processed."
```

---

## Filesystem Layout (all agents combined)

```
/workspace/
└── {REPO}.git/              ← bare clone (shared by all worktrees)

/tmp/worktrees/
├── manifest.json            ← written by setup_worktrees.sh
├── TICKET-101/
│   ├── <working files>
│   ├── DEPS.json            ← dependency declaration
│   ├── HEARTBEAT.json       ← written every 60s by sub-agent
│   ├── RESULT.json          ← written on completion
│   ├── CONFLICT_REPORT.json ← written if conflict
│   └── ESCALATION.json      ← written by watchdog if agent dies
│
└── TICKET-102/ ...

/tmp/ado_orchestration/
├── {task_id}_report.json    ← final merged results
└── {task_id}.json           ← checkpoint (from long-tasks.md)

/tmp/ado_audit/
└── audit_{date}.jsonl       ← append-only event log
```
