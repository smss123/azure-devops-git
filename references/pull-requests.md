# Pull Requests Reference — Azure DevOps

## Create a PR

```bash
# Basic PR via CLI
az repos pr create \
  --repository MY_REPO \
  --source-branch feature/TICKET-123 \
  --target-branch main \
  --title "feat: Add payment gateway AB#1234" \
  --description "$(cat PR_TEMPLATE.md)" \
  --reviewers "user@company.com" "team-name" \
  --auto-complete true \   # Merges automatically once ALL branch policies pass
  --squash true \
  --delete-source-branch true
# ⚠️  --auto-complete will merge without human intervention once policies are satisfied.
#     Only use when CI, reviewer-count, and comment-resolution policies are all enforced.

# Draft PR (not ready for review)
az repos pr create ... --draft true
```

## PR Template (place at repo root)

```markdown
<!-- Place at repo root: pull_request_template.md -->
<!-- ADO also supports .azuredevops/pull_request_template.md -->
## Summary
<!-- What does this PR do? -->

## Related Work Items
- AB#XXXX

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Refactor / cleanup

## Testing
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] Manual testing done

## Checklist
- [ ] Code follows style guide
- [ ] Self-reviewed
- [ ] No debug code / commented-out code
- [ ] Docs updated if needed
```

## Branch Policies (set via REST or portal)

```bash
# Require minimum reviewers
az repos policy approver-count create \
  --branch main \
  --repository-id $(az repos show -r MY_REPO --query id -o tsv) \
  --minimum-approver-count 2 \
  --creator-vote-counts false \
  --allow-downvotes false \
  --reset-on-source-push true \
  --blocking true

# Require linked work item
az repos policy work-item-linking create \
  --branch main \
  --repository-id REPO_ID \
  --blocking true

# Require comment resolution
az repos policy comment-required create \
  --branch main \
  --repository-id REPO_ID \
  --blocking true
```

## Manage PRs

```bash
# List open PRs assigned to me
az repos pr list --reviewer me --status active

# Approve a PR
az repos pr update --id PR_ID --status active
az repos pr reviewer add --id PR_ID --reviewer "user@company.com"

# Vote on PR
az repos pr reviewer add --id PR_ID --reviewer "me" --vote approve
# votes: approve, approve-with-suggestions, wait-for-author, reject, no-vote

# Complete (merge) a PR
az repos pr update --id PR_ID \
  --status completed \
  --merge-strategy squash \
  --delete-source-branch true

# Abandon a PR
az repos pr update --id PR_ID --status abandoned

# Add comment
az repos pr comment add --id PR_ID --comment "LGTM! Nice work."
```

## Bulk PR Operations (use with orchestrator)

```bash
# Get all active PRs across all repos
az repos pr list --status active --query "[].{id:pullRequestId,repo:repository.name,title:title,author:createdBy.displayName}" -o table

# Find PRs older than 7 days (UTC-safe)
CUTOFF=$(date -u -d "7 days ago" +"%Y-%m-%dT%H:%M:%SZ" 2>/dev/null || date -u -v-7d +"%Y-%m-%dT%H:%M:%SZ")
az repos pr list --status active --query "[?creationDate<'$CUTOFF'].{id:pullRequestId,repo:repository.name,age:creationDate}" -o table
```

## Auto-Complete Strategy

| Merge Strategy | History | When to Use |
|---|---|---|
| `merge` | Full history | Preserve all commits |
| `squash` | 1 commit | Clean main, throwaway branches |
| `rebase` | Linear | Open source style |
| `rebaseMerge` | Semi-linear | ADO default recommendation |
