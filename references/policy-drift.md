# Branch Policy Drift Detection

Branch policies get silently weakened — someone disables "require CI" for a
hotfix and forgets to re-enable it, or a new repo is created without policies.
This reference provides a scanner and enforcement pattern.

---

## Policy Baseline (define your standard)

```json
// policy_baseline.json — commit this to your infrastructure repo
{
  "main": {
    "required_reviewers": 2,
    "ci_required": true,
    "comment_resolution_required": true,
    "work_item_link_required": true,
    "merge_strategies": ["squash"],
    "allow_direct_push": false
  },
  "develop": {
    "required_reviewers": 1,
    "ci_required": true,
    "comment_resolution_required": true,
    "work_item_link_required": true,
    "merge_strategies": ["squash", "rebase"],
    "allow_direct_push": false
  },
  "release/*": {
    "required_reviewers": 2,
    "ci_required": true,
    "comment_resolution_required": true,
    "work_item_link_required": false,
    "merge_strategies": ["merge", "squash"],
    "allow_direct_push": false
  }
}
```

---

## Policy Scanner

```python
# policy_scanner.py — audit all repos against the baseline
import subprocess, json, re, datetime, os
from base64 import b64encode
import requests

ADO_ORG     = os.environ["ADO_ORG"]
ADO_PROJECT = os.environ["ADO_PROJECT"]
ADO_PAT     = os.environ["ADO_PAT"]

def ado_headers():
    return {"Authorization": f"Basic {b64encode(f':{ADO_PAT}'.encode()).decode()}"}

def get_all_repos() -> list:
    r = requests.get(
        f"{ADO_ORG}/{ADO_PROJECT}/_apis/git/repositories?api-version=7.1",
        headers=ado_headers()
    )
    r.raise_for_status()
    return r.json()["value"]

def get_branch_policies(repo_id: str) -> list:
    r = requests.get(
        f"{ADO_ORG}/{ADO_PROJECT}/_apis/policy/configurations?api-version=7.1",
        headers=ado_headers()
    )
    r.raise_for_status()
    # Filter to policies for this repo
    return [
        p for p in r.json()["value"]
        if p.get("settings", {}).get("scope", [{}])[0].get("repositoryId") == repo_id
    ]

def extract_policy_state(policies: list, branch: str) -> dict:
    """Summarise active policies for a specific branch."""
    state = {
        "required_reviewers": 0,
        "ci_required": False,
        "comment_resolution_required": False,
        "work_item_link_required": False,
        "merge_strategies": [],
        "allow_direct_push": True,
    }

    branch_ref = f"refs/heads/{branch}"

    for policy in policies:
        if not policy.get("isEnabled") or not policy.get("isBlocking"):
            continue

        scope_ref = policy.get("settings", {}).get("scope", [{}])[0].get("refName", "")
        if scope_ref != branch_ref:
            continue

        ptype = policy.get("type", {}).get("displayName", "")

        if "Minimum number of reviewers" in ptype:
            state["required_reviewers"] = policy["settings"].get("minimumApproverCount", 0)
            state["allow_direct_push"] = False

        elif "Build" in ptype:
            state["ci_required"] = True

        elif "Comment requirements" in ptype:
            state["comment_resolution_required"] = True

        elif "Work item linking" in ptype:
            state["work_item_link_required"] = True

        elif "Require a merge strategy" in ptype or "Limit merge types" in ptype:
            settings = policy["settings"]
            strategies = []
            if settings.get("allowSquash"):          strategies.append("squash")
            if settings.get("allowNoFastForward"):   strategies.append("merge")
            if settings.get("allowRebase"):          strategies.append("rebase")
            if settings.get("allowRebaseMerge"):     strategies.append("rebaseMerge")
            state["merge_strategies"] = strategies

    return state

def check_drift(actual: dict, baseline: dict, branch: str, repo: str) -> list:
    """Compare actual policies vs baseline. Returns list of violations."""
    violations = []

    if actual["required_reviewers"] < baseline.get("required_reviewers", 0):
        violations.append(
            f"{repo} / {branch}: reviewers={actual['required_reviewers']} "
            f"(required ≥{baseline['required_reviewers']})"
        )
    if baseline.get("ci_required") and not actual["ci_required"]:
        violations.append(f"{repo} / {branch}: CI policy missing or disabled")

    if baseline.get("comment_resolution_required") and not actual["comment_resolution_required"]:
        violations.append(f"{repo} / {branch}: comment resolution policy missing")

    if baseline.get("work_item_link_required") and not actual["work_item_link_required"]:
        violations.append(f"{repo} / {branch}: work item linking policy missing")

    if baseline.get("allow_direct_push") is False and actual["allow_direct_push"]:
        violations.append(f"{repo} / {branch}: direct push to branch is allowed")

    required_strategies = set(baseline.get("merge_strategies", []))
    actual_strategies = set(actual.get("merge_strategies", []))
    if required_strategies and not actual_strategies.issuperset(required_strategies):
        missing = required_strategies - actual_strategies
        violations.append(f"{repo} / {branch}: missing merge strategies: {missing}")

    return violations

def scan_all_repos(baseline_file: str = "policy_baseline.json") -> dict:
    """Scan all repos and return a drift report."""
    try:
        with open(baseline_file) as f:
            baseline = json.load(f)
    except FileNotFoundError:
        raise FileNotFoundError(
            f"Policy baseline file '{baseline_file}' not found. "
            "Create it from the template in this file and commit it to your infrastructure repo."
        )
    except json.JSONDecodeError as e:
        raise ValueError(f"Policy baseline file '{baseline_file}' is not valid JSON: {e}")

    repos = get_all_repos()
    all_violations = []
    compliant = []

    for repo in repos:
        repo_id = repo["id"]
        repo_name = repo["name"]
        policies = get_branch_policies(repo_id)

        for branch_pattern, branch_baseline in baseline.items():
            actual = extract_policy_state(policies, branch_pattern.replace("/*", ""))
            violations = check_drift(actual, branch_baseline, branch_pattern, repo_name)
            all_violations.extend(violations)

        if not any(v.startswith(repo_name) for v in all_violations):
            compliant.append(repo_name)

    report = {
        "timestamp": datetime.datetime.now(datetime.timezone.utc).isoformat().replace("+00:00", "Z"),
        "total_repos": len(repos),
        "compliant_repos": len(compliant),
        "violations_count": len(all_violations),
        "violations": all_violations,
        "compliant": compliant
    }

    return report
```

---

## Auto-Remediation

When drift is detected, optionally auto-fix it:

```python
def remediate_violations(violations: list, dry_run: bool = True):
    """
    Attempt to auto-fix policy violations.
    dry_run=True: report only. dry_run=False: apply fixes.
    """
    from policy_baseline import baseline  # your baseline dict

    for violation in violations:
        # Parse violation string to get repo + branch + issue
        # (simplified — real impl would use structured objects)
        print(f"[REMEDIATE{'(dry)' if dry_run else ''}] {violation}")

        if not dry_run:
            # Re-apply the branch policy for the affected repo/branch
            # Use az repos policy commands from references/repos-branches.md
            pass
```

---

## Schedule the Scanner

```yaml
# In your governance pipeline
schedules:
  - cron: "0 6 * * 1"    # Every Monday 6am
    displayName: Weekly policy drift scan
    branches:
      include: [main]
    always: true

jobs:
  - job: PolicyDriftScan
    steps:
      - script: |
          python3 policy_scanner.py > drift_report.json
          cat drift_report.json
          VIOLATIONS=$(python3 -c "import json; d=json.load(open('drift_report.json')); print(d['violations_count'])")
          echo "Violations found: $VIOLATIONS"
          [ "$VIOLATIONS" -eq 0 ] || exit 1
        displayName: Scan for policy drift
        env:
          ADO_PAT: $(ADO_PAT)
          TEAMS_WEBHOOK_URL: $(TEAMS_WEBHOOK_URL)
```

---

## Drift Report Output

```json
{
  "timestamp": "2025-03-14T06:00:00Z",
  "total_repos": 12,
  "compliant_repos": 10,
  "violations_count": 3,
  "violations": [
    "repo-payments / main: reviewers=1 (required ≥2)",
    "repo-legacy / develop: CI policy missing or disabled",
    "repo-frontend / main: direct push to branch is allowed"
  ],
  "compliant": ["repo-api", "repo-workers", "repo-auth", ...]
}
```
