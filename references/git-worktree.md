# Git Worktrees for Sub-Agent Parallel Work

`git worktree` lets multiple branches be checked out simultaneously in separate
directories from a **single** clone. This is the ideal primitive for sub-agents:
each agent gets its own isolated workspace with no locking conflicts.

---

## Why Worktrees for Sub-Agents

| Without Worktrees | With Worktrees |
|---|---|
| Each agent clones repo (~minutes for large repos) | One clone, instant worktree add (~seconds) |
| `.git/index` lock conflicts when agents run concurrently | Each worktree has its own index — zero conflicts |
| Agents clobber each other's working directory | Fully isolated filesystems |
| Stash/checkout thrash to switch context | Each agent stays on its branch, permanently |
| Disk: N × repo size | Disk: 1× repo + N × working files only |

---

## Core Worktree Commands

```bash
# Add a worktree for an existing branch
git worktree add /tmp/wt/feature-A feature/TICKET-001-auth

# Add a worktree AND create a new branch
git worktree add -b feature/TICKET-002-payments /tmp/wt/feature-B develop

# List all active worktrees
git worktree list

# Remove a worktree (after work is done)
git worktree remove /tmp/wt/feature-A
git worktree prune    # clean up stale .git/worktrees entries

# Move a worktree to a new path
git worktree move /tmp/wt/feature-A /workspace/feature-A
```

---

## Standard Worktree Layout for Sub-Agents

```
/workspace/
└── repo-name/              ← main clone (bare or normal)
    └── .git/
        └── worktrees/      ← git manages these automatically
/tmp/worktrees/
├── agent-0/                ← sub-agent 0's workspace (feature/TICKET-101)
├── agent-1/                ← sub-agent 1's workspace (feature/TICKET-102)
├── agent-2/                ← sub-agent 2's workspace (bugfix/TICKET-103)
└── orchestrator/           ← orchestrator reads/writes plan files here
```

### Using a Bare Clone (recommended for orchestrators)

```bash
# Bare clone — no working files, lightweight, only .git internals
# Inject PAT safely via insteadOf to avoid it appearing in .git/config
git \
  -c "url.https://anything:$ADO_PAT@dev.azure.com.insteadOf=https://dev.azure.com" \
  clone --bare https://dev.azure.com/$ORG/$PROJECT/_git/$REPO \
  /workspace/$REPO.git

cd /workspace/$REPO.git

# Spin up worktrees for each sub-agent
git worktree add /tmp/worktrees/agent-0 -b feature/TICKET-101 origin/develop
git worktree add /tmp/worktrees/agent-1 -b feature/TICKET-102 origin/develop
git worktree add /tmp/worktrees/agent-2 -b bugfix/TICKET-103  origin/main
```

---

## Orchestrator Setup Script

```bash
#!/bin/bash
# setup_worktrees.sh
# Call this once before spawning sub-agents.
# Usage: setup_worktrees.sh REPO_URL BASE_BRANCH TICKET1 TICKET2 ...

set -euo pipefail

REPO_URL=$1
BASE_BRANCH=$2
shift 2
TICKETS=("$@")

BARE_DIR="/workspace/$(basename $REPO_URL .git).git"
WT_ROOT="/tmp/worktrees"

# Clone bare repo once
if [ ! -d "$BARE_DIR" ]; then
  echo "[clone] Bare clone → $BARE_DIR"
  git clone --bare "$REPO_URL" "$BARE_DIR"
fi

cd "$BARE_DIR"
git fetch --all --prune

mkdir -p "$WT_ROOT"

# Create one worktree per ticket
for TICKET in "${TICKETS[@]}"; do
  WT_PATH="$WT_ROOT/$TICKET"
  BRANCH="feature/$TICKET"

  if [ -d "$WT_PATH" ]; then
    echo "[skip] Worktree already exists: $WT_PATH"
    continue
  fi

  git worktree add "$WT_PATH" -b "$BRANCH" "origin/$BASE_BRANCH"
  echo "[ready] $TICKET → $WT_PATH (branch: $BRANCH)"
done

# Write manifest for orchestrator/sub-agents to read
python3 - <<EOF
import json, os, glob
wts = []
for d in glob.glob("$WT_ROOT/*/"):
    ticket = os.path.basename(d.rstrip("/"))
    branch = open(f"{d}/.git/HEAD").read().strip().replace("ref: refs/heads/","") \
             if os.path.isfile(f"{d}/.git/HEAD") else "unknown"
    wts.append({"ticket": ticket, "path": d, "branch": branch})
with open("$WT_ROOT/manifest.json","w") as f:
    json.dump(wts, f, indent=2)
print(f"Manifest: $WT_ROOT/manifest.json ({len(wts)} worktrees)")
EOF
```

---

## Sub-Agent Prompt Template (with worktree)

When spawning a sub-agent, include its dedicated worktree path:

```
You are sub-agent for ticket {TICKET_ID}.

YOUR WORKSPACE: {WORKTREE_PATH}
YOUR BRANCH:    feature/{TICKET_ID}
REPO ROOT:      /workspace/{REPO}.git  (bare clone — for git operations)

RULES:
- All file edits go in {WORKTREE_PATH} ONLY
- Run git commands from {WORKTREE_PATH}, not the bare root
- Never touch other agents' worktree paths
- When done: git add, git commit, git push origin feature/{TICKET_ID}
- Write your result to: /tmp/worktrees/{TICKET_ID}/RESULT.json

TASK: {TASK_DESCRIPTION}
```

---

## Sub-Agent Work Pattern

```python
# sub_agent_work.py  — runs inside each sub-agent
import os, subprocess, json

TICKET   = os.environ["TICKET_ID"]
WT_PATH  = f"/tmp/worktrees/{TICKET}"
BRANCH   = f"feature/{TICKET}"
ADO_PAT  = os.environ["ADO_PAT"]

def git(cmd: str, cwd: str = WT_PATH) -> str:
    """Run a git command in the worktree directory."""
    result = subprocess.run(
        ["git"] + cmd.split(),
        cwd=cwd, capture_output=True, text=True
    )
    if result.returncode != 0:
        raise RuntimeError(f"git {cmd} failed:\n{result.stderr}")
    return result.stdout.strip()

def do_work():
    # 1. Ensure worktree is up to date
    git("fetch origin")
    git(f"rebase origin/develop")

    # 2. Make your changes here
    # ... edit files in WT_PATH ...

    # 3. Commit
    git("add -A")
    git(f'commit -m "feat: implement {TICKET} AB#{TICKET.split("-")[1]}"')

    # 4. Push branch
    git(f"push origin {BRANCH}")

    # 5. Write result
    result = {
        "ticket": TICKET,
        "branch": BRANCH,
        "status": "done",
        "commit": git("rev-parse --short HEAD")
    }
    with open(f"{WT_PATH}/RESULT.json", "w") as f:
        json.dump(result, f)

    return result

if __name__ == "__main__":
    print(json.dumps(do_work(), indent=2))
```

---

## Orchestrator Collect & PR Creation

```python
# collect_and_pr.py — run after all sub-agents finish
import json, glob, subprocess

WT_ROOT = "/tmp/worktrees"
DEVELOP = "develop"

results = []
for result_file in glob.glob(f"{WT_ROOT}/*/RESULT.json"):
    with open(result_file) as f:
        results.append(json.load(f))

print(f"\nCollected {len(results)} results")

for r in results:
    if r["status"] != "done":
        print(f"[SKIP] {r['ticket']} — status={r['status']}")
        continue

    # Create PR for each finished branch
    pr = subprocess.run([
        "az", "repos", "pr", "create",
        "--source-branch", r["branch"],
        "--target-branch", DEVELOP,
        "--title", f"feat: {r['ticket']}",
        "--auto-complete", "true",
        "--squash", "true",
        "--delete-source-branch", "true",
        "-o", "json"
    ], capture_output=True, text=True)

    if pr.returncode == 0:
        pr_id = json.loads(pr.stdout)["pullRequestId"]
        print(f"[PR] #{pr_id} created for {r['branch']}")
    else:
        print(f"[ERROR] PR failed for {r['branch']}: {pr.stderr}")

# Cleanup worktrees
for r in results:
    subprocess.run(["git", "worktree", "remove", "--force",
                    f"{WT_ROOT}/{r['ticket']}"])
subprocess.run(["git", "worktree", "prune"],
               cwd=f"/workspace/REPO.git")
print("\nWorktrees cleaned up.")
```

---

## Full Worktree + Git Flow Integration

Combining both patterns for a sprint's worth of features in parallel:

```bash
# 1. Orchestrator sets up worktrees (one per ticket, branching from develop)
./setup_worktrees.sh https://.../$REPO develop TICKET-101 TICKET-102 TICKET-103

# 2. Spawn one sub-agent per worktree (parallel)
# Each agent works in /tmp/worktrees/TICKET-NNN/
# Each agent follows git flow: feature branch → PR → develop

# 3. After all agents report done, collect results and create PRs
python3 collect_and_pr.py

# 4. PRs auto-complete into develop via branch policy
# 5. When sprint ends, cut release from develop per git-flow.md
./gitflow.sh release-start 2.0.0
```

---

## Troubleshooting Worktrees

| Problem | Fix |
|---|---|
| `fatal: 'path' is already checked out` | That branch is in another worktree — use a different path or `git worktree remove` first |
| Stale `.git/worktrees/` entries after crash | `git worktree prune` from the main/bare repo |
| Sub-agent can't push (auth) | Confirm `remote.origin.url` includes PAT: `git remote set-url origin https://x:$PAT@dev.azure.com/...` |
| Diverged from develop mid-sprint | `cd /tmp/worktrees/TICKET && git fetch origin && git rebase origin/develop` |
| Bare clone missing worktree config | In bare repos, ensure `core.bare=false` is NOT set — worktrees work with both bare and normal clones |
