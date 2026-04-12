# Feature Dependency Ordering

When feature B depends on feature A (uses code A introduces), they cannot be
developed in parallel without a chained branch strategy. This file provides
the full pattern for detecting, declaring, and resolving dependency chains.

---

## Dependency Types

| Type | Example | Strategy |
|---|---|---|
| **Hard** | B calls a function introduced by A | Chain branches: B branches from A |
| **Soft** | B and A touch the same file but don't interact | Serialise: A merges first, then B rebases |
| **API contract** | B consumes an API endpoint A creates | Stub A's contract; develop in parallel |
| **Database** | B reads a column A's migration adds | A's migration must merge first |
| **None** | Completely independent files/modules | Full parallel — no constraint |

---

## Declaring Dependencies

Add a `DEPENDENCIES` section to the work item description and to the worktree:

```json
// /tmp/worktrees/TICKET-102/DEPS.json
{
  "ticket": "TICKET-102",
  "depends_on": ["TICKET-101"],
  "dependency_type": "hard",
  "dependency_files": ["src/auth/oauth.py"],
  "status": "waiting"
}
```

---

## Dependency Resolution — Decision Logic

```python
# dependency_manager.py

import json, os, subprocess

WT_ROOT = "/tmp/worktrees"

def get_deps(ticket_id: str) -> dict:
    deps_file = f"{WT_ROOT}/{ticket_id}/DEPS.json"
    if not os.path.exists(deps_file):
        return {"depends_on": [], "dependency_type": "none"}
    with open(deps_file) as f:
        return json.load(f)

def is_merged_to_develop(ticket_id: str) -> bool:
    """Check if a ticket's feature branch has been squash-merged into develop."""
    result = subprocess.run(
        ["git", "log", "origin/develop", "--oneline", "--grep", ticket_id],
        capture_output=True, text=True
    )
    return bool(result.stdout.strip())

def can_start(ticket_id: str) -> tuple[bool, list]:
    """
    Returns (can_start, blocking_tickets).
    An agent should NOT start until all blocking tickets are merged.
    """
    deps = get_deps(ticket_id)
    blocking = []

    for dep_ticket in deps.get("depends_on", []):
        if not is_merged_to_develop(dep_ticket):
            blocking.append(dep_ticket)

    return (len(blocking) == 0, blocking)

def build_start_order(tickets: list) -> list[list]:
    """
    Topological sort of tickets by dependency.
    Returns list of batches — each batch can run in parallel.

    Example:
      tickets = [A, B(→A), C(→A), D(→B,→C)]
      result  = [[A], [B, C], [D]]
    """
    deps_map = {t: set(get_deps(t).get("depends_on", [])) for t in tickets}
    batches = []
    remaining = set(tickets)

    while remaining:
        # Find all tickets whose dependencies are already in completed batches
        completed = {t for batch in batches for t in batch}
        ready = {t for t in remaining if deps_map[t].issubset(completed)}

        if not ready:
            unresolved = {t: deps_map[t] - completed for t in remaining}
            raise ValueError(f"Circular or unresolvable dependency: {unresolved}")

        batches.append(sorted(ready))
        remaining -= ready

    return batches
```

---

## Chained Branch Strategy (Hard Dependency)

When B hard-depends on A, B branches from A — not from develop:

```bash
# TICKET-101 is in-flight, not merged yet
# TICKET-102 needs code from TICKET-101

# Step 1: Branch TICKET-102 from TICKET-101's branch
git checkout feature/TICKET-101
git pull origin feature/TICKET-101
git checkout -b feature/TICKET-102

# Step 2: TICKET-102 agent works normally on its own branch
# TICKET-102 now has all of TICKET-101's changes available

# Step 3: When TICKET-101 merges to develop, rebase TICKET-102
git fetch origin
git rebase origin/develop   # or: git rebase origin/feature/TICKET-101 then origin/develop

# Step 4: TICKET-102 PR now targets develop (not TICKET-101's branch)
# The chain is flattened — develop gets A+B cleanly
```

### Chained worktree setup

```bash
# In the orchestrator, after creating TICKET-101's worktree:
git -C /workspace/REPO.git worktree add \
  /tmp/worktrees/TICKET-102 \
  -b feature/TICKET-102 \
  feature/TICKET-101    # ← branch from dependency, not develop

echo "TICKET-102 branches from TICKET-101 (hard dep)"
```

---

## Contract Stub Strategy (API Dependency)

When B needs an API endpoint that A will create, don't block B — stub the contract:

```python
# TICKET-101 will create: POST /api/payments/charge
# TICKET-102 needs to call it

# In TICKET-102's branch, add a stub:
# src/stubs/payments_stub.py
def charge(amount: float, currency: str) -> dict:
    """STUB — replace when TICKET-101 merges."""
    return {"status": "ok", "transaction_id": "stub-001", "amount": amount}

# TICKET-102 imports from stub:
# from stubs.payments_stub import charge  # TODO: replace AB#101

# When TICKET-101 merges, TICKET-102 updates its import:
# from payments.api import charge
# Then removes the stub file and updates tests
```

---

## Orchestrator: Batch Execution with Dependency Ordering

```python
# ordered_orchestrator.py
from dependency_manager import build_start_order, can_start
from watchdog import run_watchdog
import threading, time

def run_ordered(all_tickets: list, spawn_fn, collect_fn):
    """
    Execute tickets in dependency order.
    Tickets in the same batch run in parallel.
    """
    try:
        batches = build_start_order(all_tickets)
    except ValueError as e:
        print(f"[ERROR] Dependency resolution failed: {e}")
        return

    print(f"[ORDER] Execution plan: {batches}")

    all_results = []

    for batch_num, batch in enumerate(batches):
        print(f"\n[BATCH {batch_num+1}/{len(batches)}] Starting: {batch}")

        # Verify all dependencies are actually merged before starting
        for ticket in batch:
            ok, blocking = can_start(ticket)
            if not ok:
                print(f"[WARN] {ticket} still blocked by: {blocking} — waiting 30s")
                time.sleep(30)
                ok, blocking = can_start(ticket)
                if not ok:
                    raise RuntimeError(f"{ticket} blocked even after wait: {blocking}")

        # Spawn all tickets in this batch in parallel
        for ticket in batch:
            spawn_fn(ticket)

        # Watch them
        wd = threading.Thread(
            target=run_watchdog,
            args=(batch, spawn_fn),
            daemon=True
        )
        wd.start()

        # Wait for batch to complete before starting next batch
        wait_for_all_agents(batch)
        batch_results = collect_fn(batch)
        all_results.extend(batch_results)

        print(f"[BATCH {batch_num+1}] Complete: {[r['status'] for r in batch_results]}")

    return all_results
```

---

## Dependency Visualisation (terminal)

```python
def print_dep_graph(tickets: list):
    """Print ASCII dependency graph before execution."""
    from dependency_manager import get_deps

    print("\nDependency graph:")
    for ticket in tickets:
        deps = get_deps(ticket).get("depends_on", [])
        if deps:
            for dep in deps:
                print(f"  {dep} ──► {ticket}")
        else:
            print(f"  (root) ──► {ticket}")
    print()
```

---

## Common Dependency Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| B branches from develop while A is in-flight | Merge conflict when A merges | Branch B from A, rebase after A merges |
| Two agents both update `package.json` independently | Lock file conflict | One ticket owns dependency updates |
| Database migration in A, schema read in B | Runtime error if B deploys before A | A's migration must be in a prior release |
| Stub never replaced after A merges | Bug silently lives in develop | Add TODO(AB#101) and a failing test that checks for the real import |
