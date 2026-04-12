# Git Flow Reference — Azure DevOps

Git Flow is a branching model with well-defined roles for each branch and a strict
release lifecycle. In ADO it pairs naturally with branch policies and PR gates.

---

## Branch Map

```
main          ──────────────────────────────────────────────►  production
               ▲              ▲              ▲
               │ merge+tag    │ hotfix merge │ release merge
               │              │              │
hotfix/1.0.1  ─┤              │              │
               │              │              │
release/1.1.0 ─┼──────────────┼──────────────┤
               │                             ▲
develop       ─┼─────────────────────────────┤──────────────►  integration
               ▲      ▲        ▲
               │      │        │  feature merges (squash/merge)
feature/A     ─┘   feature/B  feature/C
```

## Branch Roles

| Branch | Source | Merges Into | Lifetime | ADO Policy |
|---|---|---|---|---|
| `main` | — | — | permanent | ≥2 reviewers, CI required, squash |
| `develop` | `main` | — | permanent | ≥1 reviewer, CI required |
| `feature/*` | `develop` | `develop` | short-lived | CI required |
| `release/*` | `develop` | `main` + `develop` | medium | ≥2 reviewers, full test suite |
| `hotfix/*` | `main` | `main` + `develop` | short-lived | ≥1 reviewer, expedited CI |
| `bugfix/*` | `develop` | `develop` | short-lived | CI required |

---

## Full Git Flow — Step by Step

### 1. Start a Feature

```bash
# Always branch from develop
git checkout develop
git pull origin develop
git checkout -b feature/TICKET-123-add-oauth

# Work, commit (reference work item)
git commit -m "feat(auth): add OAuth2 provider AB#123"

# Keep up to date with develop (rebase preferred)
git fetch origin
git rebase origin/develop

# Push and open PR → develop
git push -u origin feature/TICKET-123-add-oauth
az repos pr create \
  --source-branch feature/TICKET-123-add-oauth \
  --target-branch develop \
  --title "feat(auth): add OAuth2 provider AB#123" \
  --auto-complete true \
  --squash true \
  --delete-source-branch true
```

### 2. Create a Release

```bash
# Cut release branch from develop
git checkout develop && git pull origin develop
git checkout -b release/1.2.0

# Bump version, update changelog — no new features here
sed -i 's/1.1.0/1.2.0/' version.txt
git commit -m "chore: bump version to 1.2.0"

# Push and open PR → main (and → develop after)
git push -u origin release/1.2.0

az repos pr create \
  --source-branch release/1.2.0 \
  --target-branch main \
  --title "release: v1.2.0" \
  --description "Release 1.2.0 — see CHANGELOG.md"
```

### 3. Finish a Release (after PR merges to main)

```bash
# Tag main
git checkout main && git pull origin main
git tag -a v1.2.0 -m "Release 1.2.0"
git push origin v1.2.0

# Back-merge into develop
git checkout develop && git pull origin develop
git merge --no-ff main -m "chore: back-merge v1.2.0 into develop"
git push origin develop
```

### 4. Hotfix

```bash
# Branch from main (NOT develop)
git checkout main && git pull origin main
git checkout -b hotfix/TICKET-999-critical-null-ref

git commit -m "fix: prevent null ref in payment handler AB#999"
git push -u origin hotfix/TICKET-999-critical-null-ref

# PR → main
az repos pr create \
  --source-branch hotfix/TICKET-999-critical-null-ref \
  --target-branch main \
  --title "hotfix: null ref in payment handler AB#999"

# After merge — tag + back-merge to develop
git checkout main && git pull
git tag -a v1.2.1 -m "Hotfix 1.2.1"
git push origin v1.2.1

git checkout develop
git merge --no-ff main -m "chore: back-merge hotfix v1.2.1 into develop"
git push origin develop
```

---

## ADO Branch Policy Setup for Git Flow

```bash
REPO_ID=$(az repos show -r MY_REPO --query id -o tsv)

# --- main: strict ---
for BRANCH in main develop; do
  # Require build
  az repos policy build create \
    --branch $BRANCH --repository-id $REPO_ID \
    --build-definition-id $CI_PIPELINE_ID \
    --blocking true --valid-duration 720 \
    --display-name "CI must pass"

  # Require reviewer
  az repos policy approver-count create \
    --branch $BRANCH --repository-id $REPO_ID \
    --minimum-approver-count $([ "$BRANCH" = "main" ] && echo 2 || echo 1) \
    --reset-on-source-push true --blocking true

  # Require comment resolution
  az repos policy comment-required create \
    --branch $BRANCH --repository-id $REPO_ID --blocking true

  # Work item link
  az repos policy work-item-linking create \
    --branch $BRANCH --repository-id $REPO_ID --blocking true
done

# main-only: no direct push (force PRs)
az repos policy merge-strategy create \
  --branch main --repository-id $REPO_ID \
  --allow-squash true --allow-no-fast-forward false \
  --allow-rebase false --allow-rebase-merge false \
  --blocking true
```

---

## Git Flow Automation Script

```bash
#!/bin/bash
# gitflow.sh — thin wrapper for common git flow operations
set -euo pipefail

DEVELOP="develop"
MAIN="main"

cmd=$1; shift

case $cmd in
  feature-start)
    TICKET=$1; DESC=$2
    git checkout $DEVELOP && git pull origin $DEVELOP
    git checkout -b "feature/$TICKET-$DESC"
    echo "Branch ready: feature/$TICKET-$DESC"
    ;;

  feature-finish)
    BRANCH=$(git branch --show-current)
    [[ $BRANCH != feature/* ]] && echo "Not on a feature branch" && exit 1
    git push -u origin $BRANCH
    az repos pr create \
      --source-branch $BRANCH --target-branch $DEVELOP \
      --title "$(git log -1 --pretty=%s)" \
      --auto-complete true --squash true --delete-source-branch true
    echo "PR created for $BRANCH → $DEVELOP"
    ;;

  release-start)
    VERSION=$1
    git checkout $DEVELOP && git pull origin $DEVELOP
    git checkout -b "release/$VERSION"
    git push -u origin "release/$VERSION"
    echo "Release branch ready: release/$VERSION"
    ;;

  hotfix-start)
    TICKET=$1; DESC=$2
    git checkout $MAIN && git pull origin $MAIN
    git checkout -b "hotfix/$TICKET-$DESC"
    echo "Hotfix branch ready: hotfix/$TICKET-$DESC"
    ;;

  tag-release)
    VERSION=$1
    git checkout $MAIN && git pull origin $MAIN
    git tag -a "v$VERSION" -m "Release $VERSION"
    git push origin "v$VERSION"
    git checkout $DEVELOP && git pull origin $DEVELOP
    git merge --no-ff $MAIN -m "chore: back-merge v$VERSION into develop"
    git push origin $DEVELOP
    echo "Tagged v$VERSION and back-merged into $DEVELOP"
    ;;

  *)
    echo "Usage: gitflow.sh <feature-start|feature-finish|release-start|hotfix-start|tag-release> [args]"
    exit 1
    ;;
esac
```

---

## Git Flow vs Trunk-Based — When to Use Each

| Criteria | Git Flow | Trunk-Based |
|---|---|---|
| Release cadence | Scheduled releases | Continuous deployment |
| Team size | Medium–large | Any |
| Parallel release versions | ✅ Yes | ❌ No |
| Hot deployment pressure | Medium | High |
| ADO Environments/gates | Works great | Works great |
| Sub-agent parallel work | ✅ Worktrees per feature | ✅ Worktrees per task |
