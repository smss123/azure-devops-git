# Agent Decision Engine — When to Act

This file answers the questions a sub-agent or orchestrator must ask at every step:
- **When do I commit?**
- **When do I open / close a feature branch?**
- **When do I create / destroy a worktree?**
- **When do I merge, and which strategy?**

Read this file whenever you are about to take any git action and are unsure whether
the moment is right. Every decision here is deterministic — there is a clear YES/NO
answer for each situation. Use the flowcharts, then the detailed rules below.

---

## Master Decision Flow

```
Agent has some work to do
        │
        ▼
┌───────────────────────────────────┐
│  Is there already an open feature │
│  branch for this ticket/task?     │
└───────────────┬───────────────────┘
               NO │                  YES
                  ▼                   ▼
        OPEN FEATURE           Is there already
        (see §1)               a worktree for it?
                                │
                          NO ───┴─── YES
                          ▼          ▼
                   CREATE         USE existing
                   WORKTREE       worktree
                   (see §3)       (see §3)
                          │
                          ▼
                  Do the work
                          │
                          ▼
              ┌───────────────────────┐
              │  Is this a good       │
              │  commit point?        │
              │  (see §2)             │
              └───────┬───────────────┘
                     YES│              NO
                        ▼               ▼
                    COMMIT          Keep working
                        │
                        ▼
              ┌───────────────────────┐
              │  Is the feature       │
              │  complete?            │
              │  (see §1 — Finish)    │
              └───────┬───────────────┘
                     YES│              NO
                        ▼               ▼
                FINISH FEATURE      Continue work
                OPEN PR             loop above
                        │
                        ▼
              ┌───────────────────────┐
              │  PR merged?           │
              └───────┬───────────────┘
                     YES│
                        ▼
                CLOSE WORKTREE
                (see §3 — Teardown)
```

---

## §1 — Feature Branch Lifecycle

### When to OPEN a feature branch

Open a feature branch when **ALL** of these are true:

| Check | Condition |
|---|---|
| Scope is clear | You know what files/systems will change |
| Ticket exists | There is an ADO work item (`AB#NNNN`) to link |
| Base is fresh | You have just fetched and `develop` (or `main` for hotfix) is up to date |
| No existing branch | `git branch -r | grep feature/TICKET` returns nothing |
| Task is isolated | It does not depend on another in-flight feature that hasn't merged yet |

**If a dependency hasn't merged yet:** wait, or branch off that feature branch and rebase chain once it merges.

```python
def should_open_feature(ticket_id, base_branch="develop") -> bool:
    checks = {
        "ticket_exists":    work_item_exists(ticket_id),
        "base_is_fresh":    seconds_since_last_fetch() < 300,   # 5 min
        "no_existing_branch": not remote_branch_exists(f"feature/{ticket_id}"),
        "no_blocking_deps": all_dependencies_merged(ticket_id),
    }
    failures = [k for k, v in checks.items() if not v]
    if failures:
        log(f"[WAIT] Cannot open feature: {failures}")
        return False
    return True
```

### When to FINISH (close) a feature branch

Close a feature by opening a PR when **ALL** of these are true:

| Check | Condition |
|---|---|
| All acceptance criteria met | Cross off every item in the work item description |
| Tests pass locally | `pytest` / `dotnet test` exits 0 in the worktree |
| No untracked work | `git status` is clean — nothing left unstaged |
| Branch is rebased | No divergence from `origin/develop` (`git log origin/develop..HEAD` shows only your commits) |
| No WIP markers | No `TODO(wip)`, `FIXME`, `console.log`, `debugger`, `pdb.set_trace()` in changed files |
| Commit count is sane | Between 1 and ~15 commits — if more, consider interactive rebase before PR |

```python
def should_finish_feature(wt_path: str, ticket_id: str) -> tuple[bool, list]:
    reasons_to_wait = []

    if not tests_pass(wt_path):
        reasons_to_wait.append("tests failing")
    if has_wip_markers(wt_path):
        reasons_to_wait.append("WIP markers found")
    if is_diverged_from_develop(wt_path):
        reasons_to_wait.append("diverged from develop — rebase first")
    if not work_item_criteria_met(ticket_id):
        reasons_to_wait.append("acceptance criteria incomplete")

    return (len(reasons_to_wait) == 0, reasons_to_wait)
```

**Never open a PR while:**
- You are mid-rebase (detached HEAD)
- Another PR for the same ticket is already open
- The CI pipeline hasn't run at least once on this branch

---

## §2 — Commit Decision Rules

Commits are the hardest decision to get right. Too few = risky, losing work, hard to review. Too many = noise, hard to bisect.

### Commit Trigger Matrix

| Situation | Commit? | Message prefix |
|---|---|---|
| Completed one logical unit of work (function, endpoint, migration) | ✅ YES | `feat:` / `fix:` |
| All tests for the current unit pass | ✅ YES | same as above |
| About to start a risky refactor | ✅ YES (safety checkpoint) | `chore: checkpoint before refactor` |
| Context switch — switching to another ticket | ✅ YES (WIP commit) | `wip: partial — switching context` |
| About to run a destructive command (`git rebase`, `rm`, DB migration) | ✅ YES | `chore: checkpoint` |
| End of work session (no matter what) | ✅ YES | `wip: EOD checkpoint` |
| File saved but feature is half-done and tests are red | ⚠️ WIP ONLY | `wip:` prefix — rebase later |
| Whitespace / formatting only | ✅ YES (separate) | `style:` |
| Multiple unrelated changes mixed together | ❌ NO | Split with `git add -p` first |
| Generated/compiled files only (no source change) | ❌ NO | Don't commit build artifacts |

### Commit Message Rules

```
<type>(<scope>): <short summary>  AB#TICKET

[optional body — WHY, not WHAT]

[optional footer: breaking changes, co-authors]
```

Types: `feat` `fix` `chore` `refactor` `style` `test` `docs` `ci` `wip`

**The AB#NNNN link is mandatory on all non-wip commits.** ADO uses it to auto-link the commit to the work item.

### Commit Granularity by Task Size

| Task Size | Target Commits | Strategy |
|---|---|---|
| < 2h (small fix) | 1 squashed commit | Work freely, squash at finish |
| 2–8h (feature) | 3–8 logical commits | Commit per unit of work |
| > 1 day (complex feature) | Commit ≥ once per hour of work | WIP commits OK, clean up with `rebase -i` before PR |
| Sub-agent parallel task | 1 commit per agent deliverable | Keep it atomic for easy revert |

### Automated Commit Quality Check

```python
def commit_is_ready(wt_path: str, staged_files: list) -> tuple[bool, list]:
    issues = []

    # 1. Are there actually staged changes?
    if not staged_files:
        issues.append("nothing staged")

    # 2. Are there WIP markers in staged files?
    for f in staged_files:
        content = read_file(f"{wt_path}/{f}")
        for marker in ["TODO(wip)", "FIXME(wip)", "console.log", "pdb.set_trace()", "debugger"]:
            if marker in content:
                issues.append(f"WIP marker '{marker}' in {f}")

    # 3. Are we in a mid-rebase state?
    if os.path.exists(f"{wt_path}/.git/rebase-merge"):
        issues.append("mid-rebase — resolve before committing")

    # 4. Are we committing binary blobs that should be in LFS?
    for f in staged_files:
        if is_large_binary(f"{wt_path}/{f}") and not is_tracked_by_lfs(wt_path, f):
            issues.append(f"large binary not in LFS: {f}")

    return (len(issues) == 0, issues)
```

---

## §3 — Worktree Lifecycle

### When to CREATE a worktree

Create a worktree when **any** of these is true:

| Trigger | Rationale |
|---|---|
| A sub-agent is about to be spawned for a ticket | Each agent needs isolation |
| You need to test a branch without losing your current working state | No stash risk |
| Reviewing a PR while your own feature is in-flight | Parallel checkout |
| Running a long background task (build, test suite) on a branch | Doesn't block foreground work |
| Bisecting a bug on a different branch | Isolated investigation |

**Do NOT create a worktree when:**
- The task takes < 5 minutes (just stash + checkout is fine)
- The branch is already checked out in another worktree (git will error)
- Disk space is tight and the repo is large (worktrees still copy working files)

### Worktree Naming Convention

```
/tmp/worktrees/{TICKET_ID}          # feature work by ticket
/tmp/worktrees/review-{PR_ID}       # PR review
/tmp/worktrees/bisect-{ISSUE}       # bug investigation
/tmp/worktrees/release-{VERSION}    # release branch work
/tmp/worktrees/hotfix-{TICKET}      # hotfix
```

Always use the ticket/PR ID so any agent can unambiguously locate it from the manifest.

### Worktree State Machine

```
[PLANNED]
    │  orchestrator calls: git worktree add ...
    ▼
[CREATED]
    │  sub-agent starts work, writes status: "running"
    ▼
[ACTIVE]
    │  commits happening, rebasing as needed
    ▼
[READY_TO_MERGE]
    │  all checks pass, PR opened
    ▼
[PR_OPEN]
    │  CI runs, reviewers approve
    ▼
[PR_MERGED]  ◄──── if PR fails: back to [ACTIVE]
    │
    ▼
[CLOSED]     git worktree remove + git worktree prune
```

### When to CLOSE (remove) a worktree

Close a worktree when **ALL** of these are true:

| Check | Condition |
|---|---|
| PR is merged | `az repos pr show --id PR_ID --query status -o tsv` returns `completed` |
| No uncommitted changes | `git -C WT_PATH status --porcelain` returns empty |
| Branch is deleted on remote | `az repos ref list --filter heads/feature/TICKET` returns nothing |
| RESULT.json written | Agent result file exists and status = `done` |

```python
def should_close_worktree(wt_path: str, pr_id: str, ticket_id: str) -> bool:
    import subprocess, json

    # Check PR merged
    pr = subprocess.run(
        ["az", "repos", "pr", "show", "--id", pr_id, "--query", "status", "-o", "tsv"],
        capture_output=True, text=True
    )
    if pr.stdout.strip() != "completed":
        return False

    # Check clean working tree
    dirty = subprocess.run(
        ["git", "-C", wt_path, "status", "--porcelain"],
        capture_output=True, text=True
    ).stdout.strip()
    if dirty:
        log(f"[WARN] Worktree {wt_path} has uncommitted changes — not removing")
        return False

    # Check RESULT.json exists
    result_file = f"{wt_path}/RESULT.json"
    if not os.path.exists(result_file):
        log(f"[WARN] No RESULT.json in {wt_path} — not removing")
        return False

    return True

def close_worktree(wt_path: str, bare_repo: str):
    subprocess.run(["git", "worktree", "remove", "--force", wt_path], check=True)
    subprocess.run(["git", "worktree", "prune"], cwd=bare_repo, check=True)
    log(f"[CLOSED] Worktree {wt_path} removed")
```

### When to MERGE a PR (and which strategy)

| Branch type | Target | Strategy | Who triggers |
|---|---|---|---|
| `feature/*` | `develop` | **Squash** | Auto-complete via policy |
| `release/*` | `main` | **Merge commit** (no squash — preserve history) | Human reviewer |
| `release/*` back-merge | `develop` | **Merge commit** | Orchestrator after tag |
| `hotfix/*` | `main` | **Squash** | Human reviewer (expedited) |
| `hotfix/*` back-merge | `develop` | **Merge commit** | Orchestrator after tag |
| `bugfix/*` | `develop` | **Squash** | Auto-complete |

**Rule of thumb:** Squash when the branch is short-lived and the commits are messy. Merge commit when the branch history must be preserved (releases, hotfixes going into develop).

---

## §4 — Dark Corners & Pitfalls

Things the rest of this skill doesn't cover but that will bite you in production.

### 1. Bare Clone + Worktree: fetchconfig mismatch

```bash
# Bare clones don't set up remote tracking refs by default.
# Without this, `git fetch --all` in the bare clone won't update worktrees.
git -C /workspace/REPO.git config remote.origin.fetch "+refs/heads/*:refs/remotes/origin/*"
git -C /workspace/REPO.git fetch --all
```
**Do this immediately after every bare clone.**

### 2. Rebase mid-flight breaks other agents

If agent A rebases `develop` while agent B is in the middle of a rebase onto `develop`, agent B's rebase will produce wrong results (it replays commits on a different base).

**Fix:** Agents must never rebase `develop` itself. Agents only `rebase origin/develop` onto their own feature branch. The `develop` branch is append-only from agent perspective.

### 3. WIP commit left in PR

A `wip:` commit that gets squashed into main is harmless — but if auto-complete is set to **merge commit** (not squash), wip commits go to develop verbatim.

**Fix:** Before finishing any feature, run:
```bash
git log --oneline origin/develop..HEAD | grep -i "^[a-f0-9]* wip"
# If any output → rebase -i and fixup/squash wip commits first
```

### 4. PAT expiry mid-long-task

ADO PATs can be 1-day to 1-year. If a PAT expires mid-task, every `git push` and `az` call silently fails or returns 401.

**Fix:** Check PAT validity at task start AND checkpoint:
```python
def validate_pat():
    import requests
    r = requests.get(
        f"{ADO_ORG}/_apis/projects?api-version=7.1",
        headers={"Authorization": f"Basic {b64encode(f':{ADO_PAT}'.encode()).decode()}"}
    )
    if r.status_code == 401:
        raise RuntimeError("ADO PAT is expired or invalid — refresh before continuing")
    return True
```

### 5. Branch policy blocks auto-complete for WIP PR

If you open a PR before CI has run once, ADO branch policy will block auto-complete indefinitely on some policy configurations (not "waiting" — permanently blocked).

**Fix:** Never open a PR with auto-complete until at least one CI run has been queued:
```bash
# Push the branch first
git push -u origin feature/TICKET-123

# Wait for CI to queue (ADO triggers on push, usually < 30s)
sleep 45

# Now open PR with auto-complete
az repos pr create ... --auto-complete true
```

### 6. Concurrent agents racing on develop back-merge

After a release is tagged, both the orchestrator and a stale sub-agent may attempt to back-merge into `develop` simultaneously, causing a merge conflict on develop itself.

**Fix:** Only the orchestrator ever writes to `develop` or `main`. Sub-agents only write to their own feature branches. Enforce this in sub-agent prompts explicitly:
```
HARD RULE: You may ONLY push to your own branch: feature/{TICKET_ID}
NEVER push to develop, main, release/*, or hotfix/*
Those branches are owned by the orchestrator.
```

### 7. Worktree disk exhaustion on large monorepos

10 worktrees × 2GB repo = 20GB in `/tmp`. This will silently fail mid-checkout.

**Fix:**
```bash
# Check available space before creating worktrees
AVAIL=$(df -BG /tmp | awk 'NR==2{print $4}' | tr -d 'G')
REPO_SIZE=$(du -sG /workspace/REPO.git | awk '{print $1}')
NEEDED=$(( REPO_SIZE * NUM_AGENTS ))
if (( NEEDED > AVAIL )); then
    echo "ERROR: Need ~${NEEDED}G, only ${AVAIL}G available"
    exit 1
fi
```

### 8. ADO PR auto-complete + squash deletes the branch but worktree stays

When ADO auto-completes a PR with "delete source branch", the remote branch is gone — but the local worktree still points to it. `git fetch --prune` from the bare clone won't remove the worktree directory.

**Fix:** The orchestrator must explicitly call `should_close_worktree()` and `close_worktree()` after detecting PR completion. Never rely on garbage collection.

### 9. Detached HEAD after release tag in worktree

```bash
git worktree add /tmp/wt/release-1.2.0 v1.2.0
# This creates a DETACHED HEAD — you can't commit to a tag
```

**Fix:** Always add worktrees pointing to a branch, not a tag:
```bash
git worktree add /tmp/wt/release-1.2.0 release/1.2.0  # ✅ branch
# NOT: git worktree add /tmp/wt/release-1.2.0 v1.2.0  # ❌ detached
```

### 10. Silent merge conflict in `--auto-complete` PR

ADO's auto-complete will attempt the merge when all policies pass. If a conflict exists, it silently fails with status `conflicts` — no notification is sent unless you've configured it.

**Fix:** Poll for conflict state after opening auto-complete PRs:
```python
import time, subprocess, json

def wait_for_pr(pr_id: str, timeout_min: int = 30):
    deadline = time.time() + timeout_min * 60
    while time.time() < deadline:
        pr = json.loads(subprocess.check_output([
            "az", "repos", "pr", "show", "--id", pr_id, "-o", "json"
        ]))
        status = pr["status"]
        merge_status = pr.get("mergeStatus", "")

        if status == "completed":
            return "merged"
        if merge_status == "conflicts":
            return "conflicts"          # orchestrator must resolve
        if status == "abandoned":
            return "abandoned"

        time.sleep(30)
    return "timeout"
```
