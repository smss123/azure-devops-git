# Conflict Resolution Agent

Read this whenever `wait_for_pr()` returns `"conflicts"`, a rebase fails, or
a sub-agent reports a merge conflict it cannot resolve automatically.

---

## Conflict Triage — Which Type Is It?

```
Conflict detected
      │
      ├─ During: git rebase origin/develop
      │       └─ §1 — Rebase conflict
      │
      ├─ During: git merge (back-merge, release finish)
      │       └─ §2 — Merge conflict
      │
      ├─ ADO PR status = "conflicts" (auto-complete stalled)
      │       └─ §3 — PR conflict (remote)
      │
      └─ Two agents edited the same file independently
              └─ §4 — Parallel agent conflict
```

---

## §1 — Rebase Conflict (local / worktree)

```bash
# Identify conflicted files
git status --short | grep "^UU\|^AA\|^DD"

# View the conflict markers in a file
git diff --diff-filter=U

# Strategy A: Accept theirs (develop wins — use when your change is additive)
git checkout --theirs  path/to/file.py
git add path/to/file.py
git rebase --continue

# Strategy B: Accept ours (feature wins — use when develop change is unrelated)
git checkout --ours path/to/file.py
git add path/to/file.py
git rebase --continue

# Strategy C: Manual merge — read both versions, write correct result
# Edit the file to remove <<<< ==== >>>> markers, then:
git add path/to/file.py
git rebase --continue

# Abort if conflict is too complex — go back to safe state
git rebase --abort
# Then open a draft PR first, resolve via PR diff review
```

### Automated conflict classification

```python
import subprocess, re

def classify_conflict(wt_path: str, filepath: str) -> str:
    """
    Classify what kind of conflict this is so the agent
    can pick the right resolution strategy.
    """
    raw = open(f"{wt_path}/{filepath}").read()

    ours_lines   = re.findall(r"<<<<<<< HEAD\n(.*?)=======", raw, re.DOTALL)
    theirs_lines = re.findall(r"=======\n(.*?)>>>>>>>", raw, re.DOTALL)

    ours_text   = "\n".join(ours_lines)
    theirs_text = "\n".join(theirs_lines)

    # Both sides added lines in the same place → content conflict
    if ours_text.strip() and theirs_text.strip():
        return "content"

    # One side deleted what the other modified → delete/modify conflict
    if not ours_text.strip() or not theirs_text.strip():
        return "delete_modify"

    # Identical content on both sides → phantom conflict (safe to accept either)
    if ours_text.strip() == theirs_text.strip():
        return "phantom"

    return "unknown"
```

### Resolution strategy by conflict type

| Conflict type | Recommended action |
|---|---|
| `phantom` | `git checkout --theirs` — both sides identical |
| `delete_modify` | Human decision required — escalate |
| `content` in test file | Accept ours — test owns the assertion |
| `content` in config/version file | Accept theirs (develop) — canonical config lives on develop |
| `content` in source logic | Manual merge — agent must read both and synthesise |
| `content` in generated file | Regenerate the file from source, don't merge |

---

## §2 — Merge Conflict (back-merge / release finish)

Back-merging main → develop after a release is the most common merge conflict site.

```bash
# Start merge, let it fail on conflicts
git checkout develop
git merge --no-ff main

# See what's conflicted
git status | grep "both modified"

# Prefer main's version for version files, changelogs
git checkout --theirs VERSION CHANGELOG.md
git add VERSION CHANGELOG.md

# Prefer develop's version for in-progress feature work
git checkout --ours src/features/

# For everything else — manual edit, then:
git add <resolved-file>

# Finalise
git commit -m "chore: resolve back-merge conflicts v1.2.1 → develop"
git push origin develop
```

---

## §3 — PR Conflict (ADO auto-complete stalled)

When `wait_for_pr()` returns `"conflicts"`:

```python
def resolve_pr_conflict(pr_id: str, ticket_id: str, wt_path: str):
    """
    Full recovery flow for a PR stuck in conflict state.
    """
    import subprocess, json

    branch = f"feature/{ticket_id}"

    # Step 1: Fetch latest state
    subprocess.run(["git", "-C", wt_path, "fetch", "origin"], check=True)

    # Step 2: Rebase onto current develop
    result = subprocess.run(
        ["git", "-C", wt_path, "rebase", "origin/develop"],
        capture_output=True, text=True
    )

    if result.returncode != 0:
        # Rebase has conflicts — log them and escalate
        conflicts = subprocess.run(
            ["git", "-C", wt_path, "diff", "--name-only", "--diff-filter=U"],
            capture_output=True, text=True
        ).stdout.strip().split("\n")

        write_conflict_report(ticket_id, wt_path, conflicts)
        return {"status": "needs_human", "conflicts": conflicts}

    # Step 3: Force-push rebased branch
    subprocess.run(
        ["git", "-C", wt_path, "push", "--force-with-lease", "origin", branch],
        check=True
    )

    # Step 4: ADO will re-evaluate merge — wait again
    # PR retains its ID; policies re-run on new commit
    return {"status": "rebased_and_pushed", "branch": branch}


def write_conflict_report(ticket_id: str, wt_path: str, conflicts: list):
    import json, datetime
    report = {
        "ticket": ticket_id,
        "timestamp": datetime.datetime.utcnow().isoformat() + "Z",
        "status": "conflict",
        "conflicted_files": conflicts,
        "action_required": "manual_resolution",
        "worktree": wt_path
    }
    with open(f"{wt_path}/CONFLICT_REPORT.json", "w") as f:
        json.dump(report, f, indent=2)
```

### Orchestrator escalation on conflict

```python
# In your orchestrator collect loop, handle conflict results:
result = wait_for_pr(pr_id)

if result == "conflicts":
    resolution = resolve_pr_conflict(pr_id, ticket_id, wt_path)

    if resolution["status"] == "needs_human":
        # Park it — notify, don't block other agents
        escalation_queue.append({
            "ticket": ticket_id,
            "pr_id": pr_id,
            "files": resolution["conflicts"],
            "action": "manual_merge_required"
        })
        continue   # move on to next ticket

    elif resolution["status"] == "rebased_and_pushed":
        # Re-enter the wait loop for this PR
        result = wait_for_pr(pr_id, timeout_min=20)
```

---

## §4 — Parallel Agent Conflict (two agents, same file)

This happens when two agents branch from the same base commit and both modify the same file.

**Prevention (preferred over resolution):**

```python
def assign_file_ownership(tickets: list, repo_path: str) -> dict:
    """
    Before spawning agents, scan each ticket's diff preview
    and detect file overlaps. If overlap found, serialize those tickets
    (one after another) rather than running in parallel.
    """
    from collections import defaultdict
    file_owners = defaultdict(list)

    for ticket in tickets:
        # Estimate files each ticket will touch from work item description / labels
        estimated_files = get_estimated_files(ticket)
        for f in estimated_files:
            file_owners[f].append(ticket)

    conflicts = {f: owners for f, owners in file_owners.items() if len(owners) > 1}

    if conflicts:
        print(f"[WARN] File ownership conflicts detected: {conflicts}")
        print("[WARN] Serializing conflicting tickets instead of running in parallel")

    return conflicts
```

**Resolution (when it already happened):**

```bash
# Find the common ancestor commit of both agents' branches
BASE=$(git merge-base feature/TICKET-A feature/TICKET-B)

# See what each changed relative to base
git diff $BASE feature/TICKET-A -- shared_file.py
git diff $BASE feature/TICKET-B -- shared_file.py

# Three-way merge manually
git checkout -b merge/TICKET-A-B develop
git merge feature/TICKET-A --no-commit
git merge feature/TICKET-B --no-commit
# Resolve conflicts in shared_file.py
git add shared_file.py
git commit -m "merge: combine TICKET-A and TICKET-B changes in shared_file AB#A AB#B"
```

---

## Conflict Severity Matrix

| Severity | Files Affected | Strategy | Who Resolves |
|---|---|---|---|
| Low | Auto-generated, lock files, changelogs | Accept theirs or regenerate | Agent (automated) |
| Medium | Config files, version files | Accept develop version | Agent (rule-based) |
| High | Shared business logic, models, interfaces | Manual three-way merge | Human or senior agent |
| Critical | Database migrations, API contracts | Freeze both branches, human decision | Human only |

**Hard rule:** Never auto-resolve a conflict in a database migration file. Always escalate to human.

---

## Conflict Notifications

```python
def notify_conflict(ticket_id: str, files: list, pr_url: str):
    """Send Teams webhook notification for conflicts needing human attention."""
    import requests, os

    webhook_url = os.environ.get("TEAMS_WEBHOOK_URL")
    if not webhook_url:
        print(f"[CONFLICT] {ticket_id}: {files} — set TEAMS_WEBHOOK_URL to enable alerts")
        return

    payload = {
        "@type": "MessageCard",
        "@context": "http://schema.org/extensions",
        "themeColor": "E24B4A",
        "summary": f"Merge conflict in {ticket_id}",
        "sections": [{
            "activityTitle": f"⚠️ Conflict needs human review: {ticket_id}",
            "activityText": f"Files: {', '.join(files)}",
            "potentialAction": [{"@type": "OpenUri", "name": "Open PR",
                                 "targets": [{"os": "default", "uri": pr_url}]}]
        }]
    }
    requests.post(webhook_url, json=payload)
```
