# Knowledge Store

The TL maintains a shared knowledge store: a structured record of what every
completed agent has produced. This is how agent B knows what agent A built,
without reading agent A's worktree directly.

---

## What Gets Stored

Every time an agent completes, the TL reads its `RESULT.json` and ingests
the `produced` block into the store. The store is the single source of truth
for inter-agent knowledge sharing.

```
Agent A finishes → writes RESULT.json with produced block
        │
        ▼
TL reads RESULT.json
        │
        ▼
TL stores: {
  "TICKET-101": {
    "new_classes":        ["payments.gateway.PaymentGateway"],
    "new_functions":      ["payments.gateway.PaymentGateway.charge"],
    "new_endpoints":      ["POST /api/payments/charge"],
    "db_schema_changes":  ["payments table: added transaction_id column"],
    "files_changed":      ["src/payments/gateway.py", "tests/test_gateway.py"],
    "notes":              "Uses stripe API key from env var STRIPE_KEY",
    "commit":             "abc1234",
    "pr_id":              "42"
  }
}
        │
        ▼
Agent B asks TL: "what did TICKET-101 produce?"
TL returns the relevant slice
```

---

## Knowledge Store Implementation

```python
# knowledge_store.py
import json, os, datetime, threading
from pathlib import Path

STORE_FILE = Path("/tmp/tl_knowledge_store.json")
_lock      = threading.Lock()


class KnowledgeStore:
    def __init__(self):
        self.data = self._load()

    def _load(self) -> dict:
        if STORE_FILE.exists():
            with open(STORE_FILE) as f:
                return json.load(f)
        return {}

    def _save(self):
        with _lock:
            with open(STORE_FILE, "w") as f:
                json.dump(self.data, f, indent=2)

    # ── Write ──────────────────────────────────────────────────────────────

    def ingest_result(self, ticket_id: str, result_path: str):
        """
        Called by TL when an agent completes.
        Reads RESULT.json and stores the 'produced' block.
        """
        if not os.path.exists(result_path):
            return

        with open(result_path) as f:
            result = json.load(f)

        if result.get("status") != "done":
            return

        entry = {
            "ticket":           ticket_id,
            "ingested_at":      datetime.datetime.now(datetime.timezone.utc).isoformat().replace("+00:00", "Z"),
            "commit":           result.get("commit"),
            "pr_id":            result.get("pr_id"),
            "branch":           result.get("branch"),
            "notes":            result.get("notes_for_team_leader", ""),
        }
        # Merge the produced block
        entry.update(result.get("produced", {}))

        self.data[ticket_id] = entry
        self._save()
        print(f"[KS] Ingested knowledge from {ticket_id}: "
              f"{list(result.get('produced', {}).keys())}")

    def record_fact(self, ticket_id: str, key: str, value):
        """
        TL records an ad-hoc fact about a ticket.
        E.g.: TL discovers a shared constant that agents should all use.
        """
        if ticket_id not in self.data:
            self.data[ticket_id] = {}
        self.data[ticket_id][key] = value
        self._save()

    # ── Read ───────────────────────────────────────────────────────────────

    def get(self, ticket_id: str) -> dict | None:
        return self.data.get(ticket_id)

    def get_all_produced(self, category: str) -> dict[str, list]:
        """
        Get everything produced across all agents in a category.
        category: "new_classes" | "new_functions" | "new_endpoints" | "db_schema_changes"
        """
        result = {}
        for ticket, data in self.data.items():
            items = data.get(category, [])
            if items:
                result[ticket] = items
        return result

    def find_owner(self, artifact: str) -> str | None:
        """
        Find which ticket produced a given artifact.
        Useful when agent B references a class and TL needs to find the source.
        artifact: e.g. "PaymentGateway", "POST /api/payments", "payments table"
        """
        artifact_lower = artifact.lower()
        for ticket, data in self.data.items():
            for category in ("new_classes", "new_functions", "new_endpoints",
                             "db_schema_changes", "files_changed"):
                for item in data.get(category, []):
                    if artifact_lower in item.lower():
                        return ticket
        return None

    def get_team_summary(self) -> str:
        """
        Generate a human-readable summary of everything the team has produced.
        Used by TL for sprint end report and for briefing new agents.
        """
        if not self.data:
            return "No agents have completed yet."

        lines = ["## Team Production Summary\n"]
        for ticket, data in sorted(self.data.items()):
            lines.append(f"### {ticket} (PR #{data.get('pr_id', '?')})")
            for cat in ("new_classes", "new_functions", "new_endpoints",
                        "db_schema_changes", "files_changed"):
                items = data.get(cat, [])
                if items:
                    label = cat.replace("_", " ").title()
                    lines.append(f"  **{label}:** {', '.join(items)}")
            notes = data.get("notes", "")
            if notes:
                lines.append(f"  **Notes:** {notes}")
            lines.append("")

        return "\n".join(lines)

    def get_context_for_agent(self, ticket_id: str,
                               depends_on: list[str]) -> str:
        """
        Build the 'context from other agents' section for a prompt.
        Used by prompt-enhancer.md to inject prior knowledge.
        """
        sections = []
        for dep_ticket in depends_on:
            data = self.get(dep_ticket)
            if not data:
                sections.append(
                    f"**{dep_ticket}:** Not yet completed — "
                    f"your task may need to wait or use a stub."
                )
                continue

            lines = [f"**{dep_ticket}** (commit `{data.get('commit', '?')}`) produced:"]

            for cat in ("new_classes", "new_functions", "new_endpoints", "db_schema_changes"):
                items = data.get(cat, [])
                if items:
                    lines.append(f"  - {cat}: {', '.join(items)}")

            notes = data.get("notes", "")
            if notes:
                lines.append(f"  - TL note: {notes}")

            sections.append("\n".join(lines))

        return "\n\n".join(sections) if sections else ""

    # ── Consistency checks ─────────────────────────────────────────────────

    def check_conflicts(self) -> list[str]:
        """
        Detect agents that both claim to produce the same artifact.
        E.g. two agents both added an endpoint POST /api/users.
        """
        conflicts = []
        seen = {}   # artifact → ticket

        for ticket, data in self.data.items():
            for cat in ("new_classes", "new_functions", "new_endpoints"):
                for item in data.get(cat, []):
                    key = f"{cat}:{item}"
                    if key in seen:
                        conflicts.append(
                            f"CONFLICT: '{item}' produced by both "
                            f"{seen[key]} and {ticket}"
                        )
                    else:
                        seen[key] = ticket

        return conflicts
```

---

## TL: Auto-Ingest on Agent Completion

Wire this into the orchestrator's result collection loop:

```python
# In orchestrator.py — after each agent completes:
from knowledge_store import KnowledgeStore

store = KnowledgeStore()

def on_agent_done(ticket_id: str):
    """Called by orchestrator when an agent's RESULT.json appears."""
    result_path = f"/tmp/worktrees/{ticket_id}/RESULT.json"

    # 1. Ingest into knowledge store
    store.ingest_result(ticket_id, result_path)

    # 2. Check for conflicts with previous agents
    conflicts = store.check_conflicts()
    if conflicts:
        from notification_templates import notify
        notify(
            title=f"Knowledge conflict detected after {ticket_id}",
            message="\n".join(conflicts),
            priority="warning"
        )

    # 3. Find any agents that were blocked waiting for this ticket
    _unblock_waiting_agents(ticket_id)


def _unblock_waiting_agents(completed_ticket: str):
    """
    Scan the TL inbox for blocker_reports waiting on this ticket.
    Re-route them as 'unblocked'.
    """
    import glob
    for req_path in glob.glob("/tmp/tl_inbox/req_*blocker_report.json"):
        with open(req_path) as f:
            req = json.load(f)
        if req["payload"].get("blocked_by") == completed_ticket:
            print(f"[KS] Unblocking {req['from_agent']} — {completed_ticket} is done")
            # Re-queue as a knowledge_request so broker delivers the result
            from message_broker import _handle_blocker, _deliver_response
            response = _handle_blocker(req)
            _deliver_response(req, response)
```

---

## Store Content Example (after 3 agents complete)

```json
{
  "TICKET-101": {
    "ticket": "TICKET-101",
    "ingested_at": "2025-03-14T10:15:00Z",
    "commit": "abc1234",
    "pr_id": "42",
    "branch": "feature/TICKET-101",
    "new_classes": ["payments.gateway.PaymentGateway"],
    "new_functions": [
      "payments.gateway.PaymentGateway.__init__",
      "payments.gateway.PaymentGateway.charge"
    ],
    "new_endpoints": [],
    "db_schema_changes": [],
    "files_changed": ["src/payments/gateway.py", "tests/test_gateway.py"],
    "notes": "Uses STRIPE_KEY env var. ChargeResult.status can be 'ok'|'declined'|'error'"
  },
  "TICKET-102": {
    "ticket": "TICKET-102",
    "commit": "def5678",
    "pr_id": "43",
    "new_endpoints": ["POST /api/payments/charge"],
    "files_changed": ["src/api/payments.py", "tests/test_api_payments.py"],
    "notes": "Calls PaymentGateway from TICKET-101. Returns 402 on declined."
  }
}
```
