# Azure DevOps Git & Sub-Agent Orchestration

> **Author:** Samer Abd Allah — [Xprema Systems](https://xprema.com)

A comprehensive **Copilot Skill** that provides expert-level guidance and automation for managing Azure DevOps repositories, branches, pull requests, pipelines, work items, and multi-agent orchestration — all from a conversational AI interface.

---

## Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Repository Structure](#repository-structure)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Authentication Setup](#authentication-setup)
- [How to Use](#how-to-use)
  - [Using as a Copilot Skill](#using-as-a-copilot-skill)
  - [Common Tasks](#common-tasks)
- [Reference Guides](#reference-guides)
- [Agent Guides](#agent-guides)
- [Team Leader Agent](#team-leader-agent)
- [Workflow Decision Tree](#workflow-decision-tree)
- [Core Principles](#core-principles)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

This repository is an **AI Copilot Skill** — a structured knowledge base that equips GitHub Copilot (or any compatible AI agent) with deep expertise in Azure DevOps and Git operations. Rather than a traditional code library, it is a collection of reference documents, decision engines, and agent playbooks that the AI reads on demand to provide accurate, production-ready guidance.

**What it solves:**

- Eliminates the need to memorize Azure DevOps CLI syntax, REST API patterns, and Git workflows.
- Provides battle-tested scripts and patterns for CI/CD pipelines, branching strategies, secrets management, and more.
- Enables **sub-agent orchestration** — coordinating multiple AI agents working in parallel on large-scale tasks like multi-repo changes, sprint kickoffs, and feature dependency chains.

---

## Key Features

| Category | Capabilities |
|---|---|
| **Git Operations** | Clone, push, merge, rebase, cherry-pick, conflict resolution, history rewrite |
| **Repository & Branch Management** | Branch policies, protection rules, naming conventions, bulk operations |
| **Pull Requests** | Creation, auto-complete, reviewer assignment, merge strategies, PR templates |
| **YAML Pipelines** | Authoring, triggers, variables, stages/jobs/steps, agent pool config |
| **Work Items & Boards** | Create/update/link work items, sprint queries, bulk import |
| **REST API** | Authenticated Python/Bash patterns, pagination, retry logic, batch endpoints |
| **Git Flow** | Feature/release/hotfix branching model with ADO policy automation |
| **Git Worktrees** | Parallel isolated workspaces for sub-agents without checkout conflicts |
| **Secrets & Config** | PAT management, variable groups, Azure Key Vault linking, secret rotation |
| **Testing & Quality** | Test pyramid, coverage thresholds, quality gates, static analysis |
| **Changelog & Releases** | Auto-generated CHANGELOG from conventional commits, release notes |
| **Notifications** | Microsoft Teams webhooks, ADO service hooks, stale PR scanning |
| **Policy Drift Detection** | Baseline definition, automated scanning, auto-remediation |
| **Semantic Versioning** | Auto-compute next version from commit history, prerelease support |
| **Sub-Agent Orchestration** | Spawn, coordinate, health-check, and recover parallel AI agents |
| **Multi-Repo Coordination** | Atomic changes across 3+ repos with saga pattern and rollback |
| **Sprint Automation** | One-command sprint kickoff from ADO backlog to running agents |

---

## Repository Structure

```
azure-devops-git/
├── SKILL.md                          # Skill manifest & entry point
├── README.md                         # This file
│
├── references/                       # How-to guides for Azure DevOps & Git
│   ├── git-ops.md                    # Clone, push, merge, rebase, cherry-pick
│   ├── repos-branches.md             # Repo creation, branch policies, naming
│   ├── pull-requests.md              # PR workflows, auto-complete, reviewers
│   ├── pipelines.md                  # YAML pipeline authoring & triggers
│   ├── work-items.md                 # Work items, sprints, board queries
│   ├── rest-api.md                   # ADO REST API patterns (Python/Bash)
│   ├── git-flow.md                   # Git Flow strategy & ADO policy setup
│   ├── git-worktree.md               # Parallel workspaces for sub-agents
│   ├── secrets-config.md             # Secrets, Key Vault, variable groups
│   ├── test-strategy.md              # Test pyramid & quality gates
│   ├── changelog-release-notes.md    # Auto-generate CHANGELOG & release notes
│   ├── notifications.md              # Teams/email alerts & service hooks
│   ├── policy-drift.md               # Branch policy drift detection & fix
│   └── semver.md                     # Semantic versioning from commits
│
└── agents/                           # AI agent coordination playbooks
    ├── decision-engine.md            # When to commit, merge, create worktrees
    ├── conflict-resolver.md          # Conflict triage & resolution
    ├── rollback-playbook.md          # Recovery from bad commits/releases
    ├── health-monitor.md             # Watchdog for sub-agent health
    ├── dependency-ordering.md        # Feature dependency graph & ordering
    ├── pr-review-automation.md       # Auto-assign reviewers, label PRs
    ├── audit-log.md                  # Structured logging for agent actions
    ├── multi-repo-saga.md            # Atomic multi-repo changes with rollback
    ├── orchestrator.md               # Central sub-agent coordination hub
    ├── long-tasks.md                 # Checkpoint/resume for long tasks
    ├── sprint-planner.md             # Sprint kickoff automation
    └── team-leader/                  # Team Leader agent subsystem
        ├── SKILL.md                  # TL entry point & architecture
        ├── prompt-enhancer.md        # Enrich prompts before agent execution
        ├── message-broker.md         # Inter-agent message routing
        ├── knowledge-store.md        # Shared memory across agents
        └── task-manager.md           # Dynamic task reassignment & splitting
```

---

## Getting Started

### Prerequisites

- **Azure DevOps Organization** with at least one project
- **Personal Access Token (PAT)** with appropriate scopes (Code, Build, Work Items, etc.)
- **Azure CLI** with the `azure-devops` extension installed
- **Git** 2.20+ (for worktree support)

### Authentication Setup

**Option 1 — Azure CLI (Recommended)**

```bash
# Install the Azure DevOps extension
az extension add --name azure-devops

# Configure defaults
az devops configure --defaults \
  organization=https://dev.azure.com/YOUR_ORG \
  project=YOUR_PROJECT

# Login (paste PAT when prompted)
az devops login --organization https://dev.azure.com/YOUR_ORG
```

**Option 2 — Environment Variables**

```bash
# Set these in your shell profile or CI/CD pipeline
export ADO_ORG="https://dev.azure.com/YOUR_ORG"
export ADO_PAT="your-pat-token"
export ADO_PROJECT="YourProject"

# Encode PAT for REST API calls
B64_PAT=$(echo -n ":$ADO_PAT" | base64)
```

**Option 3 — Git Credential Manager (for Git operations)**

```bash
# Use GCM for seamless Git auth — no PAT in URLs
git config --global credential.helper manager

# Clone via Azure CLI (handles auth automatically)
az repos clone --repository REPO_NAME
```

> ⚠️ **Security:** Never embed PATs directly in clone URLs — they leak into `.git/config`, shell history, and CI/CD logs.

---

## How to Use

### Using as a Copilot Skill

This repository is designed to be used as a **Copilot Skill**. When integrated with your AI assistant:

1. **Reference the skill** — mention Azure DevOps, ADO, pipelines, PRs, or any related topic in your conversation.
2. **Ask naturally** — describe what you want to do in plain language.
3. **Get production-ready output** — the AI uses the reference docs and agent playbooks to provide accurate commands, scripts, and strategies.

**Example prompts:**

```
"Create a feature branch from main and set up branch policies"

"Set up a YAML pipeline with build, test, and deploy stages"

"Generate a CHANGELOG from the last 20 commits"

"Kick off a sprint — create worktrees and agents for all backlog items"

"This PR has merge conflicts — help me resolve them"

"Roll back the last release to the previous version"

"Set up Teams notifications for failed pipeline runs"

"Check if branch policies have drifted from our baseline"
```

### Common Tasks

#### List Repositories

```bash
az repos list --output table
```

#### Create a Feature Branch

```bash
az repos ref create \
  --name "refs/heads/feature/my-feature" \
  --object-id $(az repos ref list --filter heads/main \
    --query "[0].objectId" -o tsv) \
  --repository MY_REPO
```

#### Create a Pull Request

```bash
az repos pr create \
  --title "Add login feature" \
  --description "Implements OAuth2 login flow" \
  --source-branch feature/my-feature \
  --target-branch main \
  --repository MY_REPO
```

#### List Open Pull Requests

```bash
az repos pr list --status active --output table
```

#### Trigger a Pipeline

```bash
az pipelines run --name "MyPipeline" --branch main
```

#### Query Work Items (Current Sprint)

```bash
az boards query --wiql \
  "SELECT [Id],[Title],[State] FROM WorkItems
   WHERE [System.IterationPath] = @CurrentIteration
   AND [System.TeamProject] = 'YourProject'"
```

#### Auto-Generate Semantic Version

```bash
# Uses conventional commits to calculate next version
# See references/semver.md for full implementation
git log --oneline v1.2.0..HEAD | grep -c "^.*feat:" && echo "→ minor bump"
git log --oneline v1.2.0..HEAD | grep -c "^.*fix:"  && echo "→ patch bump"
git log --oneline v1.2.0..HEAD | grep    "BREAKING CHANGE" && echo "→ MAJOR bump"
```

---

## Reference Guides

Each guide covers a specific Azure DevOps or Git topic in depth. Load them on demand as needed:

| Guide | Description |
|---|---|
| [references/git-ops.md](references/git-ops.md) | Clone, push, merge, cherry-pick, rebase, conflict resolution, history rewrite |
| [references/repos-branches.md](references/repos-branches.md) | Repository creation, branch policies, protection rules, naming conventions |
| [references/pull-requests.md](references/pull-requests.md) | PR creation, reviewers, auto-complete, merge strategies, templates |
| [references/pipelines.md](references/pipelines.md) | YAML pipeline authoring, stages, jobs, steps, triggers, variables, agents |
| [references/work-items.md](references/work-items.md) | Create/update/link work items, sprints, queries, bulk import |
| [references/rest-api.md](references/rest-api.md) | Raw REST patterns, pagination, retry logic, batch endpoints |
| [references/git-flow.md](references/git-flow.md) | Feature/release/hotfix branching model, ADO policy setup |
| [references/git-worktree.md](references/git-worktree.md) | Parallel isolated workspaces for sub-agents, bare clone setup |
| [references/secrets-config.md](references/secrets-config.md) | PAT, variable groups, Key Vault linking, service connections, secret rotation |
| [references/test-strategy.md](references/test-strategy.md) | Test pyramid, coverage thresholds, quality gates, agent test runner |
| [references/changelog-release-notes.md](references/changelog-release-notes.md) | Auto-generate CHANGELOG and release notes from conventional commits |
| [references/notifications.md](references/notifications.md) | Teams webhooks, ADO service hooks, stale PR scanner, alert templates |
| [references/policy-drift.md](references/policy-drift.md) | Branch policy baseline, scanner, auto-remediation, scheduled pipeline |
| [references/semver.md](references/semver.md) | Compute next version from commits, prerelease versions, pipeline integration |

---

## Agent Guides

These playbooks define how AI sub-agents coordinate, make decisions, and recover from failures:

| Agent | Description |
|---|---|
| [agents/decision-engine.md](agents/decision-engine.md) | Deterministic flowcharts: when to commit, merge, open/close features, create/destroy worktrees |
| [agents/conflict-resolver.md](agents/conflict-resolver.md) | Triage and resolve rebase, merge, PR, and parallel agent conflicts |
| [agents/rollback-playbook.md](agents/rollback-playbook.md) | Recovery patterns for bad commits, PRs, releases, pipeline failures, and deployments |
| [agents/health-monitor.md](agents/health-monitor.md) | Watchdog pattern: heartbeat protocol, stall detection, auto-restart for sub-agents |
| [agents/dependency-ordering.md](agents/dependency-ordering.md) | Topological sort for feature dependencies, chained branches, contract stubs |
| [agents/pr-review-automation.md](agents/pr-review-automation.md) | Auto-assign reviewers via CODEOWNERS, label PRs by size, post diff summaries |
| [agents/audit-log.md](agents/audit-log.md) | Structured event log for all agent actions, compliance CSV export |
| [agents/multi-repo-saga.md](agents/multi-repo-saga.md) | Saga pattern for atomic changes across 3+ repos with coordinated rollback |
| [agents/orchestrator.md](agents/orchestrator.md) | Central coordination hub: spawns sub-agents, orders by dependency, monitors health |
| [agents/long-tasks.md](agents/long-tasks.md) | Checkpoint/resume pattern for tasks exceeding context or time limits |
| [agents/sprint-planner.md](agents/sprint-planner.md) | One-command sprint kickoff: fetch backlog → create worktrees → spawn agents → dashboard |

---

## Team Leader Agent

The **Team Leader** is a meta-agent that manages other agents. It acts as a coordinator, router, and quality gatekeeper:

| Component | Purpose |
|---|---|
| [agents/team-leader/SKILL.md](agents/team-leader/SKILL.md) | Entry point — architecture overview, role map, quick decision guide |
| [agents/team-leader/prompt-enhancer.md](agents/team-leader/prompt-enhancer.md) | Rewrites vague tasks into precise agent instructions with repo context |
| [agents/team-leader/message-broker.md](agents/team-leader/message-broker.md) | Routes inter-agent requests (knowledge, files, reviews) — agents never talk directly |
| [agents/team-leader/knowledge-store.md](agents/team-leader/knowledge-store.md) | Shared memory of what every agent produced; enables cross-agent context |
| [agents/team-leader/task-manager.md](agents/team-leader/task-manager.md) | Dynamic reassignment, task splitting, duplicate merging, escalation to humans |

---

## Workflow Decision Tree

Use this to quickly find the right guide for your situation:

```
Your Task
    │
    ├── Unsure when to commit / merge / open feature?
    │       → agents/decision-engine.md
    │
    ├── PR has merge conflicts?
    │       → agents/conflict-resolver.md
    │
    ├── Something went wrong — need to undo/revert?
    │       → agents/rollback-playbook.md
    │
    ├── Sub-agent crashed or stalled?
    │       → agents/health-monitor.md
    │
    ├── Feature B depends on Feature A (not yet merged)?
    │       → agents/dependency-ordering.md
    │
    ├── Single repo / branch / PR operation?
    │       → references/git-ops.md
    │
    ├── Need a branching strategy?
    │       → references/git-flow.md
    │
    ├── Sub-agents need isolated Git workspaces?
    │       → references/git-worktree.md
    │
    ├── Secrets, variable groups, Key Vault?
    │       → references/secrets-config.md
    │
    ├── Test coverage / quality gates?
    │       → references/test-strategy.md
    │
    ├── Generate CHANGELOG or release notes?
    │       → references/changelog-release-notes.md
    │
    ├── Send Teams / email alerts?
    │       → references/notifications.md
    │
    ├── Branch policies out of sync?
    │       → references/policy-drift.md
    │
    ├── Auto-calculate next version?
    │       → references/semver.md
    │
    ├── Auto-assign reviewers / label PRs?
    │       → agents/pr-review-automation.md
    │
    ├── Need an audit trail of agent actions?
    │       → agents/audit-log.md
    │
    ├── Changes must land in 3+ repos simultaneously?
    │       → agents/multi-repo-saga.md
    │
    ├── Kick off a full sprint from backlog?
    │       → agents/sprint-planner.md
    │
    ├── Need a Team Leader to manage agents?
    │       → agents/team-leader/SKILL.md
    │
    ├── Pipeline create / run / monitor?
    │       → references/pipelines.md
    │
    ├── Work items / sprints / boards?
    │       → references/work-items.md
    │
    ├── Bulk operation (many repos / branches / PRs)?
    │       → agents/orchestrator.md
    │
    └── Long-running task (>2 min, many steps)?
            → agents/long-tasks.md
```

---

## Core Principles

1. **Idempotency** — All scripts are safe to re-run. Always check existence before creating resources.
2. **Atomicity** — Multi-step tasks use checkpoints so sub-agents can resume on failure.
3. **Least Privilege** — Scope PATs to the minimum required permissions for each task.
4. **Audit Trail** — All write operations are logged with timestamps for traceability.
5. **Rate Limits** — ADO REST API allows ~200 requests/minute per user. Bulk scripts include `sleep 0.3` between calls.

---

## Contributing

1. Fork this repository.
2. Create a feature branch: `git checkout -b feature/my-improvement`.
3. Follow [conventional commit](https://www.conventionalcommits.org/) messages (e.g., `feat:`, `fix:`, `docs:`).
4. Submit a pull request with a clear description of the changes.

When adding new reference or agent guides:
- Place operational guides in `references/`.
- Place agent coordination playbooks in `agents/`.
- Update the tables in [SKILL.md](SKILL.md) and this README.
- Follow the existing Markdown formatting and structure.

---

## License

This project is provided as-is for use with Azure DevOps and AI agent workflows. See the repository for any specific license terms.
