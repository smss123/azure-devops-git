# Notifications & Alerting Patterns

Agents must notify humans at key moments. This reference covers Teams webhooks,
email, ADO service hooks, and a unified notifier that works across channels.

---

## Notification Events Matrix

| Event | Channel | Priority | Who Sends |
|---|---|---|---|
| PR ready for review | Teams + email | Normal | PR automation agent |
| CI/CD pipeline failed | Teams | High | Pipeline (webhook) |
| Merge conflict needs human | Teams | High | Conflict resolver |
| Agent crashed/escalated | Teams | Critical | Watchdog |
| Deployment succeeded | Teams | Normal | CD pipeline |
| Release tag created | Teams + email | Normal | Orchestrator |
| PR blocked >24h | Teams | Normal | Stale PR scanner |
| Coverage dropped below threshold | Teams | High | Quality gate |

---

## Unified Notifier

```python
# notifier.py — single import for all notification channels
import os, re, requests, json, datetime
from base64 import b64encode

TEAMS_WEBHOOK  = os.environ.get("TEAMS_WEBHOOK_URL")
EMAIL_ENDPOINT = os.environ.get("ADO_EMAIL_ENDPOINT")    # ADO SendMail REST
ADO_ORG        = os.environ.get("ADO_ORG")
ADO_PAT        = os.environ.get("ADO_PAT")

PRIORITY_COLORS = {
    "info":     "0078D4",   # ADO blue
    "success":  "107C10",   # green
    "warning":  "D83B01",   # orange
    "critical": "E24B4A",   # red
}

def notify(
    title: str,
    message: str,
    priority: str = "info",
    link_url: str = None,
    link_text: str = None,
    channels: list = None
):
    """
    Send a notification to configured channels.
    channels = ["teams", "email"] — defaults to all configured
    """
    channels = channels or []
    if not channels:
        if TEAMS_WEBHOOK:  channels.append("teams")
        if EMAIL_ENDPOINT: channels.append("email")

    results = {}
    for ch in channels:
        if ch == "teams":
            results["teams"] = _send_teams(title, message, priority, link_url, link_text)
        elif ch == "email":
            results["email"] = _send_ado_email(title, message, link_url)

    return results

def _send_teams(title, message, priority, link_url, link_text):
    if not TEAMS_WEBHOOK:
        print(f"[NOTIFY] TEAMS_WEBHOOK_URL not set — skipping Teams: {title}")
        return False

    actions = []
    if link_url and link_text:
        actions = [{"@type": "OpenUri", "name": link_text,
                    "targets": [{"os": "default", "uri": link_url}]}]

    payload = {
        "@type": "MessageCard",
        "@context": "http://schema.org/extensions",
        "themeColor": PRIORITY_COLORS.get(priority, "0078D4"),
        "summary": title,
        "sections": [{
            "activityTitle": title,
            "activityText": message,
            "activitySubtitle": datetime.datetime.now(datetime.timezone.utc).strftime("%Y-%m-%d %H:%M UTC"),
        }],
        "potentialAction": actions
    }

    r = requests.post(TEAMS_WEBHOOK, json=payload, timeout=10)
    if r.status_code != 200:
        print(f"[NOTIFY] Teams webhook returned {r.status_code}: {r.text[:200]}")
    return r.status_code == 200

def _send_ado_email(title, message, link_url):
    """Send email via ADO notification service."""
    if not EMAIL_ENDPOINT or not ADO_PAT:
        return False

    headers = {
        "Authorization": f"Basic {b64encode(f':{ADO_PAT}'.encode()).decode()}",
        "Content-Type": "application/json"
    }
    body = message
    if link_url:
        body += f"\n\n{link_url}"

    r = requests.post(EMAIL_ENDPOINT, headers=headers,
                      json={"subject": title, "body": body})
    return r.status_code < 300
```

---

## Pre-built Notification Templates

```python
# notification_templates.py
from notifier import notify

def pr_ready_for_review(ticket_id: str, pr_id: str, pr_url: str, size_label: str):
    notify(
        title=f"PR ready for review — {ticket_id}",
        message=f"PR #{pr_id} is open and ready for review.\nSize: {size_label}",
        priority="info",
        link_url=pr_url,
        link_text="Open PR"
    )

def pipeline_failed(pipeline_name: str, branch: str, run_id: str, run_url: str):
    notify(
        title=f"Pipeline failed — {pipeline_name}",
        message=f"Branch: {branch}\nRun ID: {run_id}",
        priority="critical",
        link_url=run_url,
        link_text="View run"
    )

def conflict_needs_human(ticket_id: str, files: list, pr_url: str):
    file_list = "\n".join(f"  • {f}" for f in files[:5])
    if len(files) > 5:
        file_list += f"\n  • (+{len(files)-5} more)"
    notify(
        title=f"Merge conflict needs human — {ticket_id}",
        message=f"The following files have conflicts that require manual resolution:\n{file_list}",
        priority="warning",
        link_url=pr_url,
        link_text="Open PR"
    )

def agent_escalated(ticket_id: str, reason: str, phase: str):
    notify(
        title=f"Agent escalated — {ticket_id}",
        message=f"Reason: {reason}\nLast phase: {phase}\nWorktree: /tmp/worktrees/{ticket_id}",
        priority="critical"
    )

def deployment_succeeded(env: str, version: str, run_url: str):
    notify(
        title=f"Deployed to {env} — v{version}",
        message=f"Deployment completed successfully.",
        priority="success",
        link_url=run_url,
        link_text="View deployment"
    )

def release_tagged(version: str, changelog_url: str):
    notify(
        title=f"Release tagged — {version}",
        message=f"New release {version} has been tagged and is ready for deployment.",
        priority="info",
        link_url=changelog_url,
        link_text="View changelog"
    )

def coverage_dropped(branch: str, coverage: float, threshold: float):
    notify(
        title=f"Coverage below threshold — {branch}",
        message=f"Coverage: {coverage:.1f}% (threshold: {threshold:.0f}%)\nMerge is blocked.",
        priority="warning"
    )

def pr_stale(pr_id: str, ticket_id: str, days_open: int, pr_url: str):
    notify(
        title=f"Stale PR — {ticket_id} ({days_open} days open)",
        message=f"PR #{pr_id} has had no activity for {days_open} days. Needs attention.",
        priority="info",
        link_url=pr_url,
        link_text="Open PR"
    )
```

---

## ADO Service Hooks (pipeline-native)

Configure these in ADO → Project Settings → Service Hooks:

```bash
# Create a service hook via REST
curl -X POST \
  "$ADO_ORG/_apis/hooks/subscriptions?api-version=7.1" \
  -H "Authorization: Basic $B64_PAT" \
  -H "Content-Type: application/json" \
  -d '{
    "publisherId": "tfs",
    "eventType": "build.complete",
    "resourceVersion": "2.0",
    "consumerId": "webHooks",
    "consumerActionId": "httpRequest",
    "publisherInputs": {
      "buildStatus": "Failed",
      "projectId": "'"$ADO_PROJECT_ID"'"
    },
    "consumerInputs": {
      "url": "'"$TEAMS_WEBHOOK"'"
    }
  }'
```

### Key service hook event types

| Event | Use for |
|---|---|
| `build.complete` | Pipeline pass/fail |
| `git.pullrequest.created` | New PR notification |
| `git.pullrequest.merged` | PR merged → trigger downstream |
| `git.push` | Branch push monitoring |
| `workitem.updated` | Work item state changes |
| `release.deployment.completed` | Deploy done/failed |

---

## Stale PR Scanner (cron job)

```python
# stale_pr_scanner.py — run daily via scheduled pipeline
import re, subprocess, json, datetime

from notification_templates import pr_stale

def scan_stale_prs(stale_threshold_days: int = 3):
    """Find PRs with no activity in N days and notify."""
    prs = json.loads(subprocess.check_output([
        "az", "repos", "pr", "list",
        "--status", "active",
        "-o", "json"
    ]))

    now = datetime.datetime.now(datetime.timezone.utc)
    stale = []

    for pr in prs:
        created = datetime.datetime.fromisoformat(
            pr["creationDate"].replace("Z", "+00:00")
        )
        days_open = (now - created).days

        if days_open >= stale_threshold_days:
            ticket_match = re.search(r"AB#(\d+)", pr["title"])
            ticket_id = f"AB#{ticket_match.group(1)}" if ticket_match else "Unknown"

            stale.append(pr)
            pr_stale(
                pr_id=str(pr["pullRequestId"]),
                ticket_id=ticket_id,
                days_open=days_open,
                pr_url=pr["url"]
            )

    print(f"[STALE] Found {len(stale)} stale PRs")
    return stale
```

### Schedule in pipeline YAML

```yaml
schedules:
  - cron: "0 9 * * 1-5"   # 9am every weekday
    displayName: Daily stale PR scan
    branches:
      include: [main]
    always: true

jobs:
  - job: StalePRScan
    steps:
      - script: python3 stale_pr_scanner.py
        env:
          ADO_PAT: $(ADO_PAT)
          TEAMS_WEBHOOK_URL: $(TEAMS_WEBHOOK_URL)
```
