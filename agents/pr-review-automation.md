# PR Review Automation Agent

Automate the mechanical parts of PR review: assign reviewers by code ownership,
label PRs by size, post a diff summary, and enforce quality signals before human
review begins.

---

## Auto-Assign Reviewers via CODEOWNERS

### CODEOWNERS file (place at repo root or `.github/CODEOWNERS`)

```
# Format: path  owner1  owner2
# ADO uses this file to auto-assign reviewers

*                           @team-leads           # default: all files
src/auth/                   @auth-team
src/payments/               @payments-team
src/api/                    @backend-team @api-reviewers
*.sql                       @dba-team
Dockerfile                  @devops-team
azure-pipelines*.yml        @devops-team
```

### Enforce CODEOWNERS in ADO branch policy

```bash
REPO_ID=$(az repos show -r MY_REPO --query id -o tsv)

# Require CODEOWNERS approval
az repos policy required-reviewer create \
  --branch main \
  --repository-id $REPO_ID \
  --required-reviewer-ids "GROUP_ID_OR_USER_ID" \
  --message "CODEOWNERS approval required" \
  --blocking true
```

---

## Auto-Assign Reviewers (programmatic)

```python
# reviewer_assigner.py
import subprocess, json, re, os

CODEOWNERS_PATH = "CODEOWNERS"

def parse_codeowners(repo_path: str) -> list[tuple]:
    """Parse CODEOWNERS into (pattern, [owners]) pairs."""
    rules = []
    codeowners_file = f"{repo_path}/{CODEOWNERS_PATH}"
    if not os.path.exists(codeowners_file):
        return rules

    for line in open(codeowners_file):
        line = line.strip()
        if not line or line.startswith("#"):
            continue
        parts = line.split()
        pattern = parts[0]
        owners = [o.lstrip("@") for o in parts[1:]]
        rules.append((pattern, owners))
    return rules

def get_reviewers_for_pr(pr_id: str, wt_path: str) -> list[str]:
    """
    Given a PR, find the right reviewers based on CODEOWNERS
    and the files changed in the PR.
    """
    # Get changed files
    changed = subprocess.run(
        ["git", "-C", wt_path, "diff", "--name-only", "origin/develop...HEAD"],
        capture_output=True, text=True
    ).stdout.strip().split("\n")

    rules = parse_codeowners(wt_path)
    reviewers = set()

    for filepath in changed:
        for pattern, owners in reversed(rules):   # last matching rule wins
            if pattern == "*" or filepath.startswith(pattern.lstrip("/")):
                reviewers.update(owners)
                break

    return list(reviewers)

def assign_reviewers(pr_id: str, reviewers: list[str]):
    """Add reviewers to a PR."""
    for reviewer in reviewers:
        subprocess.run([
            "az", "repos", "pr", "reviewer", "add",
            "--id", pr_id,
            "--reviewer", reviewer
        ], check=True)
        print(f"[REVIEWER] Added {reviewer} to PR #{pr_id}")
```

---

## PR Size Labelling

```python
# pr_labeller.py

SIZES = [
    ("XS",  0,    10),
    ("S",   10,   50),
    ("M",   50,   200),
    ("L",   200,  500),
    ("XL",  500,  float("inf")),
]

def get_pr_diff_stats(pr_id: str) -> dict:
    """Get lines added/removed/files changed for a PR."""
    pr = json.loads(subprocess.check_output([
        "az", "repos", "pr", "show", "--id", pr_id, "-o", "json"
    ]))
    repo = pr["repository"]["name"]
    source = pr["sourceRefName"].replace("refs/heads/", "")
    target = pr["targetRefName"].replace("refs/heads/", "")

    diff = subprocess.run(
        ["git", "diff", "--stat", f"origin/{target}...origin/{source}"],
        capture_output=True, text=True, cwd=f"/tmp/review-{pr_id}"
    ).stdout

    # Parse: "5 files changed, 123 insertions(+), 45 deletions(-)"
    files   = int(re.search(r"(\d+) files? changed", diff).group(1) or 0)
    added   = int((re.search(r"(\d+) insertion", diff) or re.match(r"0", "0")).group(1) or 0)
    removed = int((re.search(r"(\d+) deletion",  diff) or re.match(r"0", "0")).group(1) or 0)

    return {"files": files, "added": added, "removed": removed, "total": added + removed}

def get_size_label(total_lines: int) -> str:
    for label, low, high in SIZES:
        if low <= total_lines < high:
            return f"size/{label}"
    return "size/XL"

def label_pr(pr_id: str, label: str):
    """Add a label tag to a PR via ADO REST."""
    import requests
    from base64 import b64encode

    headers = {
        "Authorization": f"Basic {b64encode(f':{ADO_PAT}'.encode()).decode()}",
        "Content-Type": "application/json"
    }
    requests.patch(
        f"{ADO_ORG}/{ADO_PROJECT}/_apis/git/repositories/{REPO_ID}/pullRequests/{pr_id}/labels?api-version=7.1",
        headers=headers,
        json={"name": label}
    )
    print(f"[LABEL] PR #{pr_id} labelled: {label}")
```

---

## Automated PR Summary Comment

Post a structured summary comment when a PR is opened, so reviewers know exactly what to look at.

```python
# pr_summary.py

def generate_pr_summary(pr_id: str, ticket_id: str, wt_path: str) -> str:
    """Generate a markdown summary comment for the PR."""

    # Diff stats
    stats = get_pr_diff_stats(pr_id)
    size_label = get_size_label(stats["total"])

    # Commits on this branch
    commits = subprocess.run(
        ["git", "-C", wt_path, "log", "--oneline", "origin/develop..HEAD"],
        capture_output=True, text=True
    ).stdout.strip().split("\n")

    # Changed files grouped by directory
    changed_files = subprocess.run(
        ["git", "-C", wt_path, "diff", "--name-only", "origin/develop...HEAD"],
        capture_output=True, text=True
    ).stdout.strip().split("\n")

    by_dir = {}
    for f in changed_files:
        d = os.path.dirname(f) or "root"
        by_dir.setdefault(d, []).append(os.path.basename(f))

    # Build comment
    lines = [
        f"## PR Summary — {ticket_id}",
        "",
        f"**Size:** {size_label} (+{stats['added']} / -{stats['removed']} lines across {stats['files']} files)",
        "",
        "### Commits",
    ]
    for c in commits[:10]:
        lines.append(f"- `{c}`")
    if len(commits) > 10:
        lines.append(f"- _(+{len(commits)-10} more)_")

    lines += ["", "### Changed files"]
    for d, files in sorted(by_dir.items()):
        lines.append(f"- **{d}/**: {', '.join(files)}")

    lines += [
        "",
        "### Review checklist",
        "- [ ] Logic correct",
        "- [ ] Error handling complete",
        "- [ ] Tests cover the change",
        "- [ ] No leftover debug code",
    ]

    return "\n".join(lines)

def post_pr_comment(pr_id: str, comment: str):
    """Post a comment to a PR."""
    subprocess.run([
        "az", "repos", "pr", "comment", "add",
        "--id", pr_id,
        "--comment", comment
    ], check=True)
    print(f"[COMMENT] Summary posted to PR #{pr_id}")
```

---

## Full PR Automation Pipeline

Call this once per PR, immediately after creation:

```python
def automate_pr(pr_id: str, ticket_id: str, wt_path: str):
    """Run all PR automation steps after a PR is created."""

    print(f"[PR-AUTO] Automating PR #{pr_id} for {ticket_id}")

    # 1. Size label
    stats = get_pr_diff_stats(pr_id)
    label = get_size_label(stats["total"])
    label_pr(pr_id, label)

    # 2. Auto-assign reviewers from CODEOWNERS
    reviewers = get_reviewers_for_pr(pr_id, wt_path)
    if reviewers:
        assign_reviewers(pr_id, reviewers)
    else:
        print(f"[PR-AUTO] No CODEOWNERS match — default reviewers will apply")

    # 3. Post summary comment
    summary = generate_pr_summary(pr_id, ticket_id, wt_path)
    post_pr_comment(pr_id, summary)

    # 4. Update work item to "In Review"
    subprocess.run([
        "az", "boards", "work-item", "update",
        "--id", ticket_id.split("-")[1],
        "--state", "In Review"
    ], check=True)

    print(f"[PR-AUTO] Done — PR #{pr_id} ready for review")
```

---

## XL PR Warning

PRs over 500 lines are hard to review well. Automatically warn:

```python
if stats["total"] > 500:
    warning = (
        f"⚠️ **This PR is XL ({stats['total']} lines changed).**\n\n"
        f"Consider breaking it into smaller PRs. Large PRs have higher defect rates "
        f"and slower review times. Suggested splits:\n"
        f"- Separate refactoring from new behaviour\n"
        f"- Separate DB migration from application logic\n"
        f"- Separate test additions from source changes"
    )
    post_pr_comment(pr_id, warning)
```
