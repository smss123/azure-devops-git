# Repositories & Branches Reference

## Repository Management

```bash
# Create new repo
az repos create --name my-new-repo --project $ADO_PROJECT

# List all repos with sizes
az repos list --query "[].{name:name,size:size,default:defaultBranch}" -o table

# Update default branch
az repos update --repository MY_REPO --default-branch main

# Clone all repos in a project (bulk)
az repos list --query "[].remoteUrl" -o tsv | while read url; do
  git clone "$url"
done
```

## Branch Naming Convention

```
main                        # production
develop                     # integration
release/v1.2.0              # release candidate
feature/TICKET-123-short-desc
bugfix/TICKET-456-fix-login
hotfix/TICKET-789-critical-patch
chore/update-dependencies
```

## Branch Policies via CLI

```bash
REPO_ID=$(az repos show -r MY_REPO --query id -o tsv)

# Require build to pass before merge
az repos policy build create \
  --branch main \
  --repository-id $REPO_ID \
  --build-definition-id PIPELINE_ID \
  --queue-on-source-update-only false \
  --manual-queue-only false \
  --display-name "CI Build Required" \
  --valid-duration 720 \
  --blocking true

# Require minimum reviewers
az repos policy approver-count create \
  --branch main \
  --repository-id $REPO_ID \
  --minimum-approver-count 2 \
  --creator-vote-counts false \
  --reset-on-source-push true \
  --blocking true

# Limit merge types (enforce squash only)
az repos policy merge-strategy create \
  --branch main \
  --repository-id $REPO_ID \
  --allow-squash true \
  --allow-no-fast-forward false \
  --allow-rebase false \
  --allow-rebase-merge false \
  --blocking true
```

## Bulk Branch Operations

```bash
# List all branches with last commit date
az repos ref list --repository MY_REPO --filter heads/ \
  --query "[].{name:name,commit:objectId}" -o table

# Find stale branches (no commits in 90 days) using REST API for commit dates
# Use rest-api.md for the full paginated helper; simplified example below:
python3 << 'EOF'
import subprocess, json, datetime

# Fetch branches with last-commit metadata via az REST
raw = subprocess.check_output([
    "az", "repos", "ref", "list",
    "--repository", "MY_REPO",
    "--filter", "heads/",
    "--query", "[].{name:name,objectId:objectId}",
    "-o", "json"
])
refs = json.loads(raw)
cutoff = datetime.datetime.now(datetime.timezone.utc) - datetime.timedelta(days=90)

for ref in refs:
    name = ref["name"].replace("refs/heads/", "")
    if name in ("main", "develop"):
        continue
    # Use git log to get the last commit date for this ref's commit SHA
    date_raw = subprocess.check_output(
        ["git", "log", "-1", "--format=%aI", ref["objectId"]],
        stderr=subprocess.DEVNULL
    ).decode().strip()
    if not date_raw:
        continue
    last_commit = datetime.datetime.fromisoformat(date_raw)
    if last_commit.tzinfo is None:
        last_commit = last_commit.replace(tzinfo=datetime.timezone.utc)
    if last_commit < cutoff:
        days_old = (datetime.datetime.now(datetime.timezone.utc) - last_commit).days
        print(f"{name}  (last commit: {days_old} days ago)")
EOF
```

## Gitignore Template for ADO Projects

```gitignore
# Build outputs
bin/
obj/
dist/
*.user

# IDE
.vs/
.vscode/settings.json
*.suo
.idea/

# ADO pipeline cache
.pipeline-cache/

# Secrets — NEVER commit these
*.env
local.settings.json
appsettings.Development.json
# Case-insensitive secret patterns (add both cases or use detect-secrets pre-commit hook)
*[Ss]ecret*
*[Pp]assword*
*[Cc]redential*
*[Pp]rivate[Kk]ey*

# OS
.DS_Store
Thumbs.db
```
