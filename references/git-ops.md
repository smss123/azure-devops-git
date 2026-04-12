# Git Operations Reference — Azure DevOps

## Clone & Setup

```bash
# Standard clone (uses Git Credential Manager or existing credential helper)
git clone https://dev.azure.com/ORG/PROJECT/_git/REPO

# CI/automation: inject PAT via environment — never embed in URL
# (URLs are stored in .git/config, shell history, and pipeline logs)
git -c "url.https://anything:$ADO_PAT@dev.azure.com.insteadOf=https://dev.azure.com" \
    clone https://dev.azure.com/ORG/PROJECT/_git/REPO

# Sparse clone (large monorepos — only checkout specific paths)
git -c "url.https://anything:$ADO_PAT@dev.azure.com.insteadOf=https://dev.azure.com" \
    clone --no-checkout https://dev.azure.com/ORG/PROJECT/_git/REPO
cd $REPO
git sparse-checkout init --cone
git sparse-checkout set src/ tests/
git checkout main

# Shallow clone (CI — fast, no history)
git clone --depth=1 --single-branch --branch main https://.../$REPO

# Partial clone (fetch blobs on demand — best for large repos)
git clone --filter=blob:none https://.../$REPO
```

---

## Branch Operations

```bash
# Create and switch
git checkout -b feature/TICKET-123-description

# Push with upstream tracking
git push -u origin feature/TICKET-123-description

# Delete remote branch
git push origin --delete feature/old-branch

# Sync local with remote (prune deleted remote branches)
git fetch --prune origin
git branch -vv | grep ': gone]' | awk '{print $1}' | xargs git branch -d

# Bulk delete merged branches (CAUTION — review the list before deleting)
git branch --merged main \
  | grep -v "^\*\|main\|develop" \
  | while read branch; do git branch -d "$branch" || true; done
```

---

## Commit Patterns

```bash
# Conventional commits (required for ADO PR auto-linking)
git commit -m "feat(api): add rate limiting middleware AB#1234"
# AB#NNNN syntax auto-links to work item NNNN

# Stage only specific changes within a file
git add -p filename.py

# Amend last commit (before push)
git commit --amend --no-edit

# Interactive rebase — clean up before PR
git rebase -i HEAD~5
# Commands: pick / squash(s) / fixup(f) / reword(r) / drop(d)
```

---

## Merge Strategies

| Strategy | ADO Option | When to Use |
|---|---|---|
| Merge commit | Merge | Preserve full history |
| Squash | Squash merge | Clean main branch, many WIP commits |
| Rebase | Rebase and fast-forward | Linear history |
| Semi-linear | Rebase + merge commit | Default ADO recommendation |

```bash
# Local rebase onto main before PR
git fetch origin
git rebase origin/main

# Resolve conflicts during rebase
git status                    # see conflicted files
# edit files...
git add resolved-file.txt
git rebase --continue
# or abort: git rebase --abort
```

---

## Tag Management

```bash
# Create annotated tag (use for releases — includes tagger + message)
git tag -a v1.2.0 -m "Release 1.2.0"
git push origin v1.2.0

# Push all tags at once
git push origin --tags

# List tags (sorted by version)
git tag --sort=-v:refname | head -10

# Delete a tag (only if not yet a release anchor — see rollback-playbook.md)
git tag -d v1.2.0-bad
git push origin :refs/tags/v1.2.0-bad

# Check out code at a specific tag (for investigation — detached HEAD)
git checkout v1.1.0
# IMPORTANT: never commit in detached HEAD — create a branch first:
git checkout -b investigate/v1.1.0-issue

# Find which tag contains a commit
git tag --contains abc1234
```

---

## Cherry-Pick

Use cherry-pick to apply a specific commit to a different branch without a full merge.
Typical use: apply a critical fix from develop to a release branch already cut.

```bash
# Cherry-pick a single commit
git checkout release/1.2.0
git cherry-pick abc1234

# Cherry-pick a range of commits (abc1234~1..def5678 includes abc1234)
git cherry-pick abc1234~1..def5678

# Cherry-pick without committing (apply changes, review first)
git cherry-pick --no-commit abc1234
git diff --staged   # review what you're about to commit
git commit -m "cherry-pick: fix null ref from ABC#999"

# If cherry-pick hits a conflict
git status
# resolve conflicts...
git add resolved-file.py
git cherry-pick --continue

# Abort a messy cherry-pick
git cherry-pick --abort

# Cherry-pick from another repo (add as remote first)
git remote add upstream https://...other-repo.git
git fetch upstream
git cherry-pick upstream/main~3..upstream/main
```

**When to use cherry-pick vs revert vs merge:**

| Situation | Use |
|---|---|
| Apply one fix commit to release branch | Cherry-pick |
| Apply multiple related commits | Cherry-pick range OR merge |
| Undo a specific commit on shared branch | `git revert` (see rollback-playbook.md) |
| Bring all of develop into release | Merge |

---

## Git Bisect — Find the Commit That Introduced a Bug

```bash
# Start bisect
git bisect start

# Mark current state as bad
git bisect bad HEAD

# Mark last known good state
git bisect good v1.1.0

# Git checks out a midpoint — test it, then tell git:
./run_tests.sh && git bisect good || git bisect bad

# Repeat until git reports: "abc1234 is the first bad commit"

# Automate bisect with a test script (exits 0 = good, non-zero = bad)
git bisect run python3 test_regression.py

# End bisect and return to original branch
git bisect reset

# Find the commit in ADO work items
git show abc1234 --format="%s %b"   # look for AB#NNNN in the message
```

---

## Submodules

ADO supports submodules. Use when a repo depends on a shared internal library.

```bash
# Add a submodule — use SSH or credential helper; never embed PAT in URLs
# (submodule URLs are committed to .gitmodules and visible to everyone)
git submodule add https://dev.azure.com/ORG/PROJ/_git/shared-lib libs/shared

# Clone a repo that has submodules
git clone --recurse-submodules https://.../$REPO
# Or after cloning:
git submodule update --init --recursive

# Update all submodules to their latest tracked commit
git submodule update --remote --merge

# Pin a submodule to a specific commit
cd libs/shared
git checkout v2.1.0
cd ..
git add libs/shared
git commit -m "chore: pin shared-lib to v2.1.0"

# Check submodule status
git submodule status
```

### Pipeline YAML — checkout submodules

```yaml
steps:
  - checkout: self
    submodules: recursive        # or: true (non-recursive)
    persistCredentials: true     # needed for private ADO submodules
```

---

## Stash & Recovery

```bash
git stash push -m "WIP: feature work"
git stash list
git stash pop stash@{0}
git stash drop stash@{0}   # discard without applying

# Recover lost commits (git keeps them for 30 days)
git reflog | head -20
git checkout -b recovery/lost-work abc1234

# Find a deleted branch's last commit
git reflog | grep "branch-name"
```

---

## History & Blame

```bash
# Find when a line was introduced
git log -S "search string" --oneline

# Find all commits that touched a file
git log --follow --oneline -- src/api/payments.py

# Blame with ignore whitespace and detect moved code
git blame -w -C -C filename.py

# Log with ADO work item links
git log --oneline --grep="AB#" main..feature/branch

# Show diff for a specific commit
git show abc1234 --stat

# Compare two branches
git diff main...feature/TICKET-123 --stat
```

---

## Large Repos — Performance Tips

```bash
# Git LFS for large files (binaries, models, media)
git lfs track "*.psd" "*.zip" "*.dll" "*.model"
git add .gitattributes
git lfs pull     # download LFS objects

# Maintenance (speed up local operations)
git gc --aggressive
git repack -a -d --depth=250 --window=250

# Diagnose slow operations
GIT_TRACE_PERFORMANCE=1 git status
```

---

## Hooks for ADO

Place in `.git/hooks/` or configure `core.hooksPath` for team-wide hooks:

```bash
# commit-msg: enforce work item reference
# Allowed prefixes: conventional commit types, Merge, Revert, Initial commits
# Note: 'wip:' is allowed for in-progress checkpoints but must be squashed before PR
#!/bin/bash
MSG=$(cat "$1")
if ! echo "$MSG" | grep -qE "AB#[0-9]+|^Merge |^Revert |^Initial commit"; then
  echo "ERROR: Commit must reference a work item (AB#NNNN) or start with Merge/Revert/Initial commit"
  exit 1
fi

# pre-push: block direct push to protected branches
#!/bin/bash
while read local_ref local_sha remote_ref remote_sha; do
  BRANCH=$(echo "$remote_ref" | sed 's|refs/heads/||')
  if [[ "$BRANCH" =~ ^(main|develop|release/.*)$ ]]; then
    echo "ERROR: Direct push to $BRANCH is blocked. Open a PR."
    exit 1
  fi
done
```
