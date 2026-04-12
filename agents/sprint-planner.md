# Sprint Planning Agent

Automates the full sprint kickoff: read the sprint backlog from ADO, resolve
dependencies, create branches and worktrees for each ticket, spawn sub-agents,
and report back with a live sprint dashboard. One command to go from "sprint
planned" to "all agents working".

---

## What This Agent Does

```
Sprint backlog (ADO)
        │
        ▼
1. Fetch all tickets in current sprint that are "Ready"
2. Detect dependency ordering (DEPS.json or work item links)
3. Create worktrees for all tickets
4. Generate sub-agent prompts for each ticket
5. Spawn sub-agents in batches (dependency order)
6. Watch via health monitor
7. Post sprint kickoff summary to Teams
8. Print live dashboard
```

---

## Sprint Kickoff Script

```python
# sprint_kickoff.py
# Usage: python3 sprint_kickoff.py --repo my-repo --sprint "Sprint 5"

import subprocess, json, os, argparse, datetime
from pathlib import Path

from audit_log          import AuditLog
from orchestrator       import Orchestrator
from dependency_ordering import build_start_order
from notification_templates import notify

WT_ROOT  = "/tmp/worktrees"
ADO_ORG  = os.environ["ADO_ORG"]
ADO_PAT  = os.environ["ADO_PAT"]
ADO_PROJECT = os.environ["ADO_PROJECT"]

audit = AuditLog(agent_id="sprint-kickoff")


def fetch_sprint_tickets(sprint_name: str, states: list = None) -> list[dict]:
    """
    Fetch all tickets in the given sprint that are in an actionable state.
    Default states: Ready, Active (not Done, not New — those haven't been groomed)
    """
    states = states or ["Ready", "Active"]
    state_filter = " OR ".join(f"[System.State] = '{s}'" for s in states)

    wiql = f"""
    SELECT [System.Id], [System.Title], [System.State],
           [System.AssignedTo], [Microsoft.VSTS.Scheduling.StoryPoints],
           [System.Description]
    FROM WorkItems
    WHERE [System.TeamProject] = '{ADO_PROJECT}'
      AND [System.IterationPath] CONTAINS '{sprint_name}'
      AND ({state_filter})
      AND [System.WorkItemType] IN ('User Story', 'Task', 'Bug')
    ORDER BY [Microsoft.VSTS.Common.StackRank]
    """

    result = subprocess.run([
        "az", "boards", "query", "--wiql", wiql, "-o", "json"
    ], capture_output=True, text=True, check=True)

    items = json.loads(result.stdout)
    tickets = []

    for item in items:
        fields = item.get("fields", item)  # format varies
        wi_id  = fields.get("System.Id") or item.get("id")
        title  = fields.get("System.Title", "")
        state  = fields.get("System.State", "")
        points = fields.get("Microsoft.VSTS.Scheduling.StoryPoints", "?")

        tickets.append({
            "ticket":  f"TICKET-{wi_id}",
            "wi_id":   wi_id,
            "title":   title,
            "state":   state,
            "points":  points,
        })

    return tickets


def detect_dependencies_from_ado(tickets: list[dict]) -> dict[str, list[str]]:
    """
    Read ADO work item links to find parent/child and Predecessor relationships.
    Returns {ticket_id: [depends_on_ticket_id, ...]}
    """
    deps = {}

    for t in tickets:
        wi_id = t["wi_id"]
        result = subprocess.run([
            "az", "boards", "work-item", "show",
            "--id", str(wi_id),
            "--expand", "relations",
            "-o", "json"
        ], capture_output=True, text=True)

        if result.returncode != 0:
            continue

        wi = json.loads(result.stdout)
        relations = wi.get("relations", [])

        predecessors = []
        for rel in relations:
            if "Predecessor" in rel.get("rel", ""):
                # Extract the related work item ID from the URL
                related_id = rel["url"].split("/")[-1]
                # Check if this related WI is in our sprint
                if any(str(t2["wi_id"]) == related_id for t2 in tickets):
                    predecessors.append(f"TICKET-{related_id}")

        if predecessors:
            deps[t["ticket"]] = predecessors

    return deps


def create_deps_files(deps: dict[str, list[str]], tickets: list[dict]):
    """Write DEPS.json to each worktree that has dependencies."""
    for ticket_id, dep_list in deps.items():
        wt_path = f"{WT_ROOT}/{ticket_id}"
        Path(wt_path).mkdir(parents=True, exist_ok=True)

        dep_type = "hard"  # conservative default
        deps_data = {
            "ticket": ticket_id,
            "depends_on": dep_list,
            "dependency_type": dep_type,
        }
        with open(f"{wt_path}/DEPS.json", "w") as f:
            json.dump(deps_data, f, indent=2)

    print(f"[SPRINT] Dependency files written for {len(deps)} tickets")


def setup_all_worktrees(repo: str, tickets: list[dict], base_branch: str = "develop"):
    """Create worktrees for all sprint tickets."""
    bare_dir = f"/workspace/{repo}.git"

    # Ensure bare clone exists
    if not os.path.exists(bare_dir):
        print(f"[SPRINT] Cloning {repo}...")
        subprocess.run([
            "git", "clone", "--bare",
            f"https://anything:{ADO_PAT}@dev.azure.com/"
            f"{ADO_ORG.split('/')[-1]}/{ADO_PROJECT}/_git/{repo}",
            bare_dir
        ], check=True)

    # Fix fetch config (dark corner #1 from decision-engine.md)
    subprocess.run([
        "git", "-C", bare_dir, "config",
        "remote.origin.fetch", "+refs/heads/*:refs/remotes/origin/*"
    ], check=True)
    subprocess.run(["git", "-C", bare_dir, "fetch", "--all", "--prune"], check=True)

    created = []
    for t in tickets:
        ticket_id = t["ticket"]
        wt_path   = f"{WT_ROOT}/{ticket_id}"
        branch    = f"feature/{ticket_id}"

        if os.path.exists(wt_path):
            print(f"  [skip] {ticket_id} — worktree already exists")
            created.append({"ticket": ticket_id, "path": wt_path, "branch": branch})
            continue

        try:
            subprocess.run([
                "git", "-C", bare_dir,
                "worktree", "add", "-b", branch,
                wt_path, f"origin/{base_branch}"
            ], check=True)
            print(f"  [wt] {ticket_id} → {wt_path}")
            created.append({"ticket": ticket_id, "path": wt_path, "branch": branch,
                            "repo": repo, "task": t["title"]})
            audit.log("worktree_create", target=ticket_id,
                      detail=f"branch={branch} base={base_branch}")
        except subprocess.CalledProcessError as e:
            print(f"  [err] {ticket_id}: {e}")

    # Write manifest
    with open(f"{WT_ROOT}/manifest.json", "w") as f:
        json.dump(created, f, indent=2)

    return created


def build_sub_agent_prompt(spec: dict) -> str:
    """Generate the canonical sub-agent prompt for a ticket."""
    return f"""You are sub-agent for {spec['ticket']}.

ENVIRONMENT:
  ADO_ORG:     {ADO_ORG}
  ADO_PROJECT: {ADO_PROJECT}
  ADO_PAT:     [already set in env]

YOUR WORKSPACE:
  Worktree: {WT_ROOT}/{spec['ticket']}
  Branch:   feature/{spec['ticket']}
  Repo:     /workspace/{spec.get('repo', 'REPO')}.git

TASK: {spec.get('task', 'Implement the acceptance criteria in the work item')}
WORK ITEM: AB#{spec['ticket'].replace('TICKET-', '')}

RULES:
  1. Work ONLY in your worktree — never touch other agents' paths
  2. Only push to feature/{spec['ticket']}
  3. NEVER push to develop, main, release/*, or hotfix/*
  4. Write HEARTBEAT.json every 60s
  5. Write RESULT.json when done
  6. On conflict: write CONFLICT_REPORT.json, stop

USE agents/decision-engine.md for all git timing decisions.
USE agents/conflict-resolver.md if you hit a merge conflict.
"""


def print_sprint_dashboard(tickets: list[dict], sprint_name: str):
    """Print a live sprint status dashboard."""
    print(f"\n{'='*70}")
    print(f"  SPRINT DASHBOARD — {sprint_name}")
    print(f"  {datetime.datetime.now(datetime.timezone.utc).strftime('%Y-%m-%d %H:%M UTC')}")
    print(f"{'='*70}")
    print(f"  {'TICKET':<15} {'PTS':<5} {'STATUS':<14} {'TITLE'[:35]}")
    print(f"  {'-'*65}")

    total_pts = 0
    done_pts  = 0

    for t in tickets:
        tid    = t["ticket"]
        pts    = t.get("points", "?")
        title  = t.get("title", "")[:35]

        result_f    = f"{WT_ROOT}/{tid}/RESULT.json"
        hb_f        = f"{WT_ROOT}/{tid}/HEARTBEAT.json"
        conflict_f  = f"{WT_ROOT}/{tid}/CONFLICT_REPORT.json"
        escalation_f = f"{WT_ROOT}/{tid}/ESCALATION.json"

        if os.path.exists(result_f):
            status = "✓ done"
            if isinstance(pts, (int, float)):
                done_pts += pts
        elif os.path.exists(escalation_f):
            status = "✗ escalated"
        elif os.path.exists(conflict_f):
            status = "⚠ conflict"
        elif os.path.exists(hb_f):
            hb = json.load(open(hb_f))
            status = f"⟳ {hb.get('phase','?')}"
        else:
            status = "○ queued"

        if isinstance(pts, (int, float)):
            total_pts += pts

        print(f"  {tid:<15} {str(pts):<5} {status:<14} {title}")

    print(f"  {'-'*65}")
    print(f"  Total: {len(tickets)} tickets  |  {done_pts}/{total_pts} pts done")
    print(f"{'='*70}\n")


def run_sprint_kickoff(repo: str, sprint_name: str, dry_run: bool = False):
    """Full sprint kickoff orchestration."""
    print(f"\n[SPRINT] Kicking off {sprint_name} on {repo}")

    # 1. Fetch sprint backlog
    tickets = fetch_sprint_tickets(sprint_name)
    print(f"[SPRINT] Found {len(tickets)} actionable tickets")
    if not tickets:
        print("[SPRINT] No tickets to work on — exiting")
        return

    # 2. Detect dependencies
    deps = detect_dependencies_from_ado(tickets)
    if deps:
        print(f"[SPRINT] Dependencies detected: {deps}")

    if dry_run:
        print("\n[DRY RUN] Would create worktrees for:")
        for t in tickets:
            print(f"  {t['ticket']}: {t['title']}")
        print(f"\n[DRY RUN] Execution order:")
        try:
            order = build_start_order([t["ticket"] for t in tickets])
            for i, batch in enumerate(order):
                print(f"  Batch {i+1}: {batch}")
        except Exception as e:
            print(f"  Could not determine order: {e}")
        return

    # 3. Write DEPS.json files
    create_deps_files(deps, tickets)

    # 4. Create worktrees
    specs = setup_all_worktrees(repo, tickets)

    # 5. Build sub-agent prompts and save
    prompts_dir = f"/tmp/sprint_{sprint_name.replace(' ', '_')}_prompts"
    Path(prompts_dir).mkdir(exist_ok=True)
    for spec in specs:
        prompt = build_sub_agent_prompt(spec)
        with open(f"{prompts_dir}/{spec['ticket']}.txt", "w") as f:
            f.write(prompt)

    print(f"[SPRINT] Sub-agent prompts written to {prompts_dir}/")
    print(f"[SPRINT] Spawn sub-agents using these prompts, then run the dashboard")

    # 6. Notify team
    ticket_list = "\n".join(f"  • {t['ticket']}: {t['title']}" for t in tickets)
    notify(
        title=f"Sprint kickoff: {sprint_name}",
        message=f"{len(tickets)} tickets starting now:\n{ticket_list}",
        priority="info"
    )

    # 7. Print initial dashboard
    print_sprint_dashboard(tickets, sprint_name)

    audit.log("agent_start", target=sprint_name,
              detail=f"kickoff: {len(tickets)} tickets, {len(deps)} dependency chains")

    return specs


if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--repo",    required=True, help="ADO repository name")
    parser.add_argument("--sprint",  required=True, help="Sprint name, e.g. 'Sprint 5'")
    parser.add_argument("--dry-run", action="store_true", help="Plan only, no changes")
    args = parser.parse_args()

    run_sprint_kickoff(args.repo, args.sprint, dry_run=args.dry_run)
```

---

## Live Dashboard (standalone)

Run this at any time during a sprint to see current agent status:

```bash
# watch -n 30 python3 sprint_dashboard.py --sprint "Sprint 5"
python3 - <<'EOF'
import json, os, glob, datetime

WT_ROOT = "/tmp/worktrees"
SPRINT  = "Sprint 5"

tickets = []
for manifest_path in glob.glob(f"{WT_ROOT}/manifest.json"):
    with open(manifest_path) as f:
        tickets = json.load(f)

if not tickets:
    print("No manifest.json found. Run sprint_kickoff.py first.")
    exit()

print(f"\n{'='*70}")
print(f"  SPRINT DASHBOARD — {SPRINT}")
print(f"  {datetime.datetime.now(datetime.timezone.utc).strftime('%Y-%m-%d %H:%M UTC')}")
print(f"{'='*70}")
print(f"  {'TICKET':<16} {'STATUS':<15} {'PHASE':<12} {'BRANCH'}")
print(f"  {'-'*65}")

done = fail = escalated = active = queued = 0

for t in tickets:
    tid    = t["ticket"]
    branch = t.get("branch", f"feature/{tid}")

    result_f     = f"{WT_ROOT}/{tid}/RESULT.json"
    hb_f         = f"{WT_ROOT}/{tid}/HEARTBEAT.json"
    conflict_f   = f"{WT_ROOT}/{tid}/CONFLICT_REPORT.json"
    escalation_f = f"{WT_ROOT}/{tid}/ESCALATION.json"

    if os.path.exists(result_f):
        status, phase = "✓ done", "—"
        done += 1
    elif os.path.exists(escalation_f):
        status, phase = "✗ ESCALATED", "—"
        escalated += 1
    elif os.path.exists(conflict_f):
        status, phase = "⚠ CONFLICT", "—"
        fail += 1
    elif os.path.exists(hb_f):
        with open(hb_f) as f: hb = json.load(f)
        import time
        dt = hb.get("last_update", "")
        if dt:
            from datetime import timezone
            age = (datetime.datetime.now(timezone.utc) -
                   datetime.datetime.fromisoformat(dt.replace("Z","+00:00"))).total_seconds()
            stale = " ⚠️" if age > 300 else ""
        else:
            stale = ""
        status = f"⟳ working{stale}"
        phase  = hb.get("phase","?")
        active += 1
    else:
        status, phase = "○ queued", "—"
        queued += 1

    print(f"  {tid:<16} {status:<15} {phase:<12} {branch}")

print(f"  {'-'*65}")
print(f"  done={done}  active={active}  queued={queued}  conflict={fail}  escalated={escalated}")
print(f"{'='*70}")
EOF
```

---

## Sprint Velocity Report

Run at sprint end to summarise what was delivered:

```python
# sprint_velocity.py
import json, os, glob, subprocess

WT_ROOT = "/tmp/worktrees"

results = []
for result_file in glob.glob(f"{WT_ROOT}/*/RESULT.json"):
    with open(result_file) as f:
        results.append(json.load(f))

escalations = list(glob.glob(f"{WT_ROOT}/*/ESCALATION.json"))
conflicts   = list(glob.glob(f"{WT_ROOT}/*/CONFLICT_REPORT.json"))

print(f"\n=== SPRINT VELOCITY REPORT ===")
print(f"Tickets completed:  {sum(1 for r in results if r.get('status')=='done')}")
print(f"Tickets failed:     {sum(1 for r in results if r.get('status')!='done')}")
print(f"Escalations:        {len(escalations)}")
print(f"Unresolved conflicts: {len(conflicts)}")

print("\nPRs created:")
for r in results:
    if r.get("pr_id"):
        print(f"  PR #{r['pr_id']} — {r.get('ticket','?')}: {r.get('detail','')}")
```
