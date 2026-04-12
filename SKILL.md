---
name: azure-devops-git
description: >
  Expert Azure DevOps + Git skill for managing repositories, branches, pull requests, pipelines,
  work items, and long-running tasks via sub-agents. Use this skill whenever the user mentions:
  Azure DevOps, ADO, Azure Repos, Azure Pipelines, YAML pipelines, git in ADO context, PRs in
  DevOps, branch policies, work items, sprints, backlogs, boards, service connections, release
  pipelines, build agents, PAT tokens, secrets, variable groups, Key Vault, CODEOWNERS, coverage
  gates, changelogs, semver, policy drift, rollbacks, conflict resolution, or multi-repo changes.
  Also trigger when the user wants to: clone/push/pull from ADO repos, manage branches at scale,
  run parallel tasks, coordinate sub-agents for long jobs, automate CI/CD, manage ADO via REST API,
  implement git flow, use git worktrees, handle merge conflicts, revert bad releases, monitor agent
  health, order feature dependencies, auto-assign PR reviewers, generate release notes, send Teams
  or email alerts, detect policy drift, automate versioning, audit agent actions, coordinate
  atomic changes across multiple repos, kick off a full sprint with one command, use a Team Leader
  agent to manage sub-agents, route requests between agents, enhance prompts before agents run,
  share knowledge between agents, dynamically reassign failed tasks, or split oversized agent work.
  Do NOT skip this skill even if the request seems simple — it contains critical patterns and pitfalls.
---
# author : samer abd allah (xprema systems) 
# Azure DevOps Git & Sub-Agent Orchestration Skill

## Quick Reference — Read First

| Task Category | Jump To |
|---|---|
| Auth & Setup | [Authentication](#authentication) |
| Git Operations | [references/git-ops.md] |
| Repos & Branches | [references/repos-branches.md] |
| Pull Requests | [references/pull-requests.md] |
| Pipelines (YAML/Classic) | [references/pipelines.md] |
| Work Items & Boards | [references/work-items.md] |
| REST API Patterns | [references/rest-api.md] |
| Git Flow Strategy | [references/git-flow.md] |
| Worktrees for Sub-Agents | [references/git-worktree.md] |
| Secrets & Env Config | [references/secrets-config.md] |
| Test Strategy & Quality Gates | [references/test-strategy.md] |
| Changelog & Release Notes | [references/changelog-release-notes.md] |
| Notifications & Alerting | [references/notifications.md] |
| Policy Drift Detection | [references/policy-drift.md] |
| Semantic Versioning | [references/semver.md] |
| **⚡ When to commit / open / close / merge** | **[agents/decision-engine.md] ← read when unsure** |
| Sub-Agent Orchestration | [agents/orchestrator.md] |
| Long Task Management | [agents/long-tasks.md] |
| **Conflict Resolution** | **[agents/conflict-resolver.md] ← when PR hits conflicts** |
| **Rollback & Recovery** | **[agents/rollback-playbook.md] ← when things go wrong** |
| **Agent Health Monitor** | **[agents/health-monitor.md] ← watchdog for sub-agents** |
| **Feature Dependency Ordering** | **[agents/dependency-ordering.md] ← chained features** |
| PR Review Automation | [agents/pr-review-automation.md] |
| Agent Audit Log | [agents/audit-log.md] |
| Multi-Repo Atomic Changes | [agents/multi-repo-saga.md] |
| **Sprint Kickoff Automation** | **[agents/sprint-planner.md] ← full sprint in one command** |
| **Team Leader Agent** | **[agents/team-leader/SKILL.md] ← manages sub-agents + inter-agent comms** |

---

## Authentication

**Always establish auth before any ADO operation.**

### Personal Access Token (PAT)
```bash
# Set env vars — never hardcode
export ADO_ORG="https://dev.azure.com/YOUR_ORG"
export ADO_PAT="your-pat-token"
export ADO_PROJECT="YourProject"

# Encode for basic auth header
B64_PAT=$(echo -n ":$ADO_PAT" | base64)
```

### az CLI (preferred for most tasks)
```bash
# Install & login
az extension add --name azure-devops
az devops configure --defaults organization=$ADO_ORG project=$ADO_PROJECT
az devops login --organization $ADO_ORG   # paste PAT when prompted
```

### Git credential setup for ADO
```bash
# Preferred: use the Git Credential Manager (GCM) — no PAT in URLs
git config --global credential.helper manager
az repos clone --repository REPO   # az CLI handles auth automatically

# For CI/CD automation without GCM, pass PAT via Git's askpass mechanism
# rather than embedding it in the URL (URLs appear in logs and .git/config)
export GIT_ASKPASS=/usr/lib/git-core/git-askpass
git -c credential.helper= \
    -c "url.https://anything:$ADO_PAT@dev.azure.com.insteadOf=https://dev.azure.com" \
    clone https://dev.azure.com/ORG/PROJECT/_git/REPO
# ⚠️  NEVER embed PAT directly in clone URLs — they are stored in .git/config,
#     shell history, and CI/CD logs.
```

---

## Core Principles

1. **Idempotency** — All scripts must be safe to re-run. Check existence before creating.
2. **Atomicity** — For multi-step tasks, checkpoint state so sub-agents can resume.
3. **Least privilege** — Scope PATs to minimum required permissions.
4. **Audit trail** — Log all write operations with timestamps.
5. **Rate limits** — ADO REST API: 200 req/min per user. Add `sleep 0.3` between bulk calls.

---

## Workflow Decision Tree

```
User Request
    │
    ├─ Unsure WHEN to commit, open feature, close worktree, or merge?
    │       └─ agents/decision-engine.md  ← START HERE
    │
    ├─ PR hit a conflict / rebase failed?
    │       └─ agents/conflict-resolver.md  ← triage + recovery steps
    │
    ├─ Something went wrong — need to undo / revert / recover?
    │       └─ agents/rollback-playbook.md  ← branch/PR/release/deploy rollback
    │
    ├─ Sub-agent crashed, stalled, or hung?
    │       └─ agents/health-monitor.md  ← watchdog + restart logic
    │
    ├─ Feature B depends on Feature A (not merged yet)?
    │       └─ agents/dependency-ordering.md  ← chained branches + batch order
    │
    ├─ Single repo/branch/PR operation?
    │       └─ references/git-ops.md
    │
    ├─ Need a branching strategy / release workflow?
    │       └─ references/git-flow.md
    │
    ├─ Sub-agents need isolated git workspaces?
    │       └─ references/git-worktree.md
    │
    ├─ Secrets, variable groups, Key Vault, env config?
    │       └─ references/secrets-config.md
    │
    ├─ Test coverage / quality gates / static analysis?
    │       └─ references/test-strategy.md
    │
    ├─ Generate CHANGELOG or release notes?
    │       └─ references/changelog-release-notes.md
    │
    ├─ Send Teams/email alerts for PR/pipeline/deploy events?
    │       └─ references/notifications.md
    │
    ├─ Branch policies out of sync across repos?
    │       └─ references/policy-drift.md
    │
    ├─ Auto-calculate next version from commits?
    │       └─ references/semver.md
    │
    ├─ Auto-assign reviewers, label PR by size, post diff summary?
    │       └─ agents/pr-review-automation.md
    │
    ├─ Audit trail of all agent actions?
    │       └─ agents/audit-log.md
    │
    ├─ Change must land in 3+ repos simultaneously?
    │       └─ agents/multi-repo-saga.md  ← saga with rollback
    │
    ├─ Kick off a full sprint from ADO backlog?
    │       └─ agents/sprint-planner.md  ← tickets → worktrees → agents in one command
    │
    ├─ Need a Team Leader to manage agents + route inter-agent requests + enhance prompts?
    │       └─ agents/team-leader/SKILL.md  ← start here, then its sub-files
    │
    ├─ Pipeline create/run/monitor?
    │       └─ references/pipelines.md
    │
    ├─ Work items / sprints / boards?
    │       └─ references/work-items.md
    │
    ├─ Bulk operation (many repos, branches, PRs)?
    │       └─ agents/orchestrator.md
    │
    └─ Long-running task (>2 min, many steps)?
            └─ agents/long-tasks.md
```

---

## Common Quick Commands

```bash
# List all repos in project
az repos list --output table

# Create branch
az repos ref create --name "refs/heads/feature/my-branch" \
  --object-id $(az repos ref list --filter heads/main --query "[0].objectId" -o tsv) \
  --repository MY_REPO

# List open PRs
az repos pr list --status active --output table

# Trigger a pipeline
az pipelines run --name "MyPipeline" --branch main

# List work items in current sprint
az boards query --wiql "SELECT [Id],[Title],[State] FROM WorkItems WHERE [System.IterationPath] = @CurrentIteration AND [System.TeamProject] = '$ADO_PROJECT'"
```

---

## Reference Files

Load these on demand — only read what's needed for the task:

- **references/git-ops.md** — clone, push, merge, cherry-pick, rebase, conflict resolution, history rewrite
- **references/repos-branches.md** — repo creation, branch policies, protection rules, naming conventions
- **references/pull-requests.md** — PR creation, reviewers, auto-complete, merge strategies, PR templates
- **references/pipelines.md** — YAML pipeline authoring, stages/jobs/steps, triggers, variables, secrets, agents
- **references/work-items.md** — create/update/link work items, sprints, queries, bulk import
- **references/rest-api.md** — raw REST patterns, pagination, retry logic, batch endpoints
- **references/git-flow.md** — feature/release/hotfix branching model, ADO policy setup, automation scripts
- **references/git-worktree.md** — parallel isolated workspaces per sub-agent, bare clone setup, full orchestration
- **references/secrets-config.md** — PAT, variable groups, Key Vault linking, service connections, secret scanning, rotation
- **references/test-strategy.md** — test pyramid, coverage thresholds by branch, quality gates, agent test runner
- **references/changelog-release-notes.md** — auto-generate CHANGELOG.md and release notes from conventional commits
- **references/notifications.md** — Teams webhooks, ADO service hooks, stale PR scanner, pre-built alert templates
- **references/policy-drift.md** — baseline definition, scanner, auto-remediation, scheduled pipeline
- **references/semver.md** — compute next version from commits, prerelease versions, pipeline integration

## Agent Files

- **agents/decision-engine.md** — when to commit, open/close features, create/destroy worktrees, merge strategy
- **agents/conflict-resolver.md** — rebase conflicts, PR conflicts, parallel agent conflicts, escalation
- **agents/rollback-playbook.md** — branch/PR/release/deployment/ADO-data rollback with decision matrix
- **agents/health-monitor.md** — heartbeat protocol, watchdog loop, restart logic, health dashboard
- **agents/dependency-ordering.md** — topological sort, chained branches, contract stubs, batch execution
- **agents/pr-review-automation.md** — CODEOWNERS reviewer assignment, size labelling, diff summary comment
- **agents/audit-log.md** — structured event log, AuditLog class, query tool, compliance CSV export
- **agents/multi-repo-saga.md** — prepare/validate/commit/rollback saga for atomic multi-repo changes
- **agents/team-leader/SKILL.md** — Team Leader entry point: architecture, role map, quick decision guide
- **agents/team-leader/prompt-enhancer.md** — rewrite vague tasks into precise instructions before agents start; inject prior agent knowledge; set quality bar; anticipate blockers
- **agents/team-leader/message-broker.md** — inter-agent request/response protocol; agents never talk directly; TL routes knowledge_request, file_request, clarification_request, blocker_report, review_request, split_request, human_request
- **agents/team-leader/knowledge-store.md** — shared memory of what every agent produced; auto-ingested from RESULT.json; `find_owner()`, `get_context_for_agent()`, conflict detection
- **agents/team-leader/task-manager.md** — dynamic reassignment, task splitting, duplicate merging, escalation; full TL main event loop
- **agents/orchestrator.md** — central coordination hub: integrates health-monitor, audit-log, dependency-ordering, canonical sub-agent prompt template, full filesystem layout
- **agents/long-tasks.md** — checkpoint/resume pattern for tasks that exceed context or time limits
