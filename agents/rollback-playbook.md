# Rollback & Recovery Playbook

Use this when something has gone wrong and you need to undo or recover.
Every scenario has a safe path that avoids force-pushing shared branches.

---

## Triage First — What Went Wrong?

```
Something is broken
        │
        ├─ Bad commit on MY feature branch (not merged yet)
        │       └─ §1 — Local/branch rollback — safe, no impact on others
        │
        ├─ Bad PR merged into develop
        │       └─ §2 — Revert PR — create a new revert commit
        │
        ├─ Bad release merged into main + tagged
        │       └─ §3 — Hotfix or revert release
        │
        ├─ Pipeline deployed bad build to an environment
        │       └─ §4 — Pipeline rollback / re-deploy previous artifact
        │
        └─ Wrong work item state / corrupt ADO data
                └─ §5 — ADO data recovery
```

---

## §1 — Branch Rollback (before merge)

```bash
# Undo last N commits but keep the changes staged
git reset --soft HEAD~N

# Undo last N commits and discard ALL changes (DESTRUCTIVE)
git reset --hard HEAD~N

# Remove a specific commit from history (interactive rebase)
git rebase -i HEAD~10
# Mark the bad commit as 'drop' in the editor

# If already pushed to remote — force push YOUR branch only
# (Never force-push develop or main)
git push --force-with-lease origin feature/TICKET-123

# Recover a dropped commit via reflog (within 30 days)
git reflog | grep "commit before the bad one"
git checkout -b recovery/TICKET-123 abc1234
```

---

## §2 — Revert a Merged PR (develop or main)

**Never force-push develop or main.** Always use `git revert` to create a new undo commit.

```bash
# Find the merge commit SHA
git log --oneline --merges develop | head -10

# Revert a squash-merged commit (single commit on develop)
git checkout develop && git pull origin develop
git revert ABC1234 --no-edit
git push origin develop

# Revert a merge commit (--mainline 1 = keep develop's parent)
git revert -m 1 MERGE_COMMIT_SHA --no-edit
git push origin develop

# Then open a revert PR if your branch policy requires it
az repos pr create \
  --source-branch develop \
  --target-branch main \
  --title "revert: undo PR #NNN — REASON" \
  --description "Reverts merge commit MERGE_SHA due to REASON"
```

### Revert via ADO (preferred for audited projects)

```python
def revert_pr(pr_id: str, reason: str):
    """Create a revert of a merged PR via ADO REST API."""
    import subprocess, json

    # Get the merge commit from the completed PR
    pr_data = json.loads(subprocess.check_output([
        "az", "repos", "pr", "show", "--id", pr_id, "-o", "json"
    ]))

    merge_commit = pr_data.get("lastMergeCommit", {}).get("commitId")
    repo = pr_data["repository"]["name"]
    target = pr_data["targetRefName"].replace("refs/heads/", "")

    if not merge_commit:
        raise ValueError(f"PR #{pr_id} has no merge commit — was it completed?")

    # Create revert branch
    subprocess.run([
        "git", "checkout", target
    ], check=True)
    subprocess.run(["git", "pull", "origin", target], check=True)
    revert_branch = f"revert/pr-{pr_id}"
    subprocess.run(["git", "checkout", "-b", revert_branch], check=True)
    subprocess.run(["git", "revert", "-m", "1", merge_commit, "--no-edit"], check=True)
    subprocess.run(["git", "push", "-u", "origin", revert_branch], check=True)

    # Open revert PR
    result = subprocess.run([
        "az", "repos", "pr", "create",
        "--repository", repo,
        "--source-branch", revert_branch,
        "--target-branch", target,
        "--title", f"revert: PR #{pr_id} — {reason}",
        "--description", f"Reverts merge commit {merge_commit[:8]}.\nReason: {reason}",
        "-o", "json"
    ], capture_output=True, text=True)

    pr = json.loads(result.stdout)
    print(f"[REVERT PR] #{pr['pullRequestId']} created: {pr['url']}")
    return pr
```

---

## §3 — Rollback a Release (main + tag)

```bash
# DO NOT delete the tag — it's an audit record
# DO NOT force-push main

# Option A: Revert the release merge commit on main
git checkout main && git pull origin main
git revert -m 1 RELEASE_MERGE_SHA --no-edit
git push origin main

# Create a new patch tag to document the rollback
git tag -a v1.2.0-reverted -m "v1.2.0 reverted due to REASON"
git push origin v1.2.0-reverted

# Option B: Hotfix (preferred if you know exactly what's broken)
# See references/git-flow.md §4 — Hotfix
# This is cleaner: fix the bug, release v1.2.1

# Back-merge the revert into develop too
git checkout develop && git pull origin develop
git merge --no-ff main -m "chore: sync revert of v1.2.0 into develop"
git push origin develop
```

---

## §4 — Pipeline / Deployment Rollback

### Re-deploy the previous successful artifact

```bash
# Find last successful run before the bad one
LAST_GOOD=$(az pipelines runs list \
  --pipeline-name "MyApp-CD" \
  --result succeeded \
  --query "sort_by([?finishTime<'BAD_RUN_FINISH_TIME'], &finishTime)[-1].id" \
  -o tsv)

echo "Last good run: $LAST_GOOD"

# Re-trigger that run's artifact through a new run
# (Preferred: create a parameterised pipeline that accepts a build ID)
az pipelines run \
  --name "MyApp-CD" \
  --variables buildId=$LAST_GOOD environment=production
```

### Rollback pipeline YAML

```yaml
# Add a rollback stage to your CD pipeline
- stage: Rollback
  displayName: Rollback to previous
  condition: failed()
  jobs:
    - deployment: RollbackDeploy
      environment: production
      strategy:
        runOnce:
          deploy:
            steps:
              - task: AzureWebApp@1
                inputs:
                  appName: $(appName)
                  package: $(Pipeline.Workspace)/previous-drop/**/*.zip
```

### Environment-specific rollback via ADO Environments

```bash
# List recent deployments to an environment
az devops invoke \
  --area distributedtask \
  --resource environments \
  --route-parameters project=$ADO_PROJECT environmentId=ENV_ID \
  --api-version 7.1

# The ADO portal "Environments" page shows deployment history
# and has a "Re-deploy" button for previous successful deployments
```

---

## §5 — ADO Data Recovery

### Restore accidentally deleted work items

```python
def restore_deleted_work_item(wi_id: int):
    """ADO keeps deleted work items in recycle bin for 30 days."""
    import requests
    from base64 import b64encode

    headers = {
        "Authorization": f"Basic {b64encode(f':{ADO_PAT}'.encode()).decode()}",
        "Content-Type": "application/json"
    }

    # Check recycle bin
    r = requests.get(
        f"{ADO_ORG}/{ADO_PROJECT}/_apis/wit/recyclebin/{wi_id}?api-version=7.1",
        headers=headers
    )
    if r.status_code == 404:
        raise ValueError(f"Work item {wi_id} not in recycle bin (>30 days or never deleted)")

    # Restore it
    restore = requests.patch(
        f"{ADO_ORG}/{ADO_PROJECT}/_apis/wit/recyclebin/{wi_id}?api-version=7.1",
        headers=headers,
        json={"isDeleted": False}
    )
    restore.raise_for_status()
    print(f"[RESTORED] Work item #{wi_id}")
```

### Recover a deleted branch (within retention window)

```bash
# ADO keeps deleted branches for 14 days in the "deleted branches" view
# Restore via portal: Repos → Branches → Deleted tab → Restore

# Or via REST API:
curl -X POST \
  "$ADO_ORG/$ADO_PROJECT/_apis/git/repositories/$REPO_ID/refs?api-version=7.1" \
  -H "Authorization: Basic $B64_PAT" \
  -H "Content-Type: application/json" \
  -d '[{
    "name": "refs/heads/feature/TICKET-123",
    "newObjectId": "LAST_KNOWN_SHA",
    "oldObjectId": "0000000000000000000000000000000000000000"
  }]'
```

---

## Rollback Decision Matrix

| Situation | Safe Action | Forbidden |
|---|---|---|
| Bad commit, not pushed | `git reset --hard` | Nothing — local only |
| Bad commit, pushed to feature branch | `git push --force-with-lease` | Force-push develop/main |
| Bad PR merged into develop | `git revert COMMIT` | Force-push develop |
| Bad release on main | `git revert -m 1 MERGE_SHA` | Delete the tag |
| Bad deployment | Re-deploy previous artifact | Revert infrastructure changes without approval |
| Deleted work item | Restore from recycle bin (≤30 days) | Recreate manually (loses history) |

---

## Post-Rollback Checklist

After any rollback, always complete these steps:

- [ ] Revert commit / revert PR is merged and CI passes
- [ ] Tag created documenting the rollback (e.g. `v1.2.0-reverted`)
- [ ] Develop is in sync with main (back-merge if needed)
- [ ] All open feature branches rebased onto updated develop
- [ ] Work items updated: add comment explaining what happened
- [ ] Pipeline re-run confirms clean deployment
- [ ] Retrospective note added to sprint work item or wiki
- [ ] TEAMS_WEBHOOK_URL notification sent to team
