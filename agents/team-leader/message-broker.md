# Message Broker

The TL is the only communication channel between agents. Agents never talk to
each other directly — they send requests to the TL and receive answers.
This prevents circular dependencies, race conditions, and context pollution.

---

## Why Agents Don't Talk Directly

```
WRONG — direct agent communication:
  Agent B reads Agent A's worktree directly
  → Agent B is now coupled to Agent A's file layout
  → If A is still writing, B reads partial state
  → No audit trail of what was shared

CORRECT — TL-mediated:
  Agent B sends REQUEST to TL
  → TL waits until A's output is confirmed complete
  → TL extracts only the relevant piece
  → TL delivers it in the exact format B needs
  → Audit log records the exchange
```

---

## Request Types

| Request type | When an agent sends it | TL action |
|---|---|---|
| `knowledge_request` | "I need the schema/API/class from ticket X" | Look up knowledge store, deliver |
| `file_request` | "I need to read a file from ticket X's worktree" | Read + deliver the file content |
| `clarification_request` | "I don't understand what I should do" | Enhance the prompt, re-send |
| `blocker_report` | "I can't proceed until ticket X merges" | Check status, unblock or queue |
| `review_request` | "Can another agent check my code?" | Route to TL review or peer agent |
| `split_request` | "This task is too large for me" | Task manager splits it |
| `human_request` | "I need a human decision" | Escalate via notifications |

---

## Message File Protocol

Every message is a JSON file in the shared TL inbox:

### Directory layout

```
/tmp/tl_inbox/
├── {timestamp}_{from_agent}_{type}.json     ← pending requests
└── processed/
    └── {timestamp}_{from_agent}_{type}.json ← handled requests

/tmp/tl_outbox/
└── {ticket_id}/
    └── {timestamp}_response.json            ← TL reply to agent
```

### Request schema

```json
{
  "request_id":   "req_20250314_102345_abc",
  "from_agent":   "TICKET-102",
  "request_type": "knowledge_request",
  "timestamp":    "2025-03-14T10:23:45Z",
  "payload": {
    "need":        "PaymentGateway class interface",
    "from_ticket": "TICKET-101",
    "reason":      "I need to call .charge() but don't have the signature",
    "urgency":     "blocking"
  }
}
```

### Response schema

```json
{
  "request_id":  "req_20250314_102345_abc",
  "to_agent":    "TICKET-102",
  "status":      "delivered",
  "timestamp":   "2025-03-14T10:23:52Z",
  "payload": {
    "answer": "PaymentGateway(api_key: str, timeout: int = 30)\n.charge(amount: float, currency: str) -> ChargeResult\nChargeResult.transaction_id, .status, .error_message",
    "source": "TICKET-101 RESULT.json produced.new_classes",
    "confidence": "high"
  }
}
```

---

## Message Broker Implementation

```python
# message_broker.py
import json, os, glob, time, datetime, shutil
from pathlib import Path
from audit_log import AuditLog
from knowledge_store import KnowledgeStore

TL_INBOX   = Path("/tmp/tl_inbox")
TL_OUTBOX  = Path("/tmp/tl_outbox")
TL_INBOX.mkdir(exist_ok=True)
TL_OUTBOX.mkdir(exist_ok=True)
(TL_INBOX / "processed").mkdir(exist_ok=True)

audit = AuditLog(agent_id="team-leader/broker")
store = KnowledgeStore()


def send_request(from_agent: str, request_type: str, payload: dict) -> str:
    """Agent calls this to send a request to the TL. Returns request_id."""
    req_id = f"req_{datetime.datetime.utcnow().strftime('%Y%m%d_%H%M%S')}_{from_agent[-6:]}"
    msg = {
        "request_id":   req_id,
        "from_agent":   from_agent,
        "request_type": request_type,
        "timestamp":    datetime.datetime.utcnow().isoformat() + "Z",
        "payload":      payload
    }
    path = TL_INBOX / f"{req_id}_{request_type}.json"
    with open(path, "w") as f:
        json.dump(msg, f, indent=2)
    return req_id


def wait_for_response(request_id: str, timeout_sec: int = 300) -> dict | None:
    """Agent calls this to wait for the TL's answer."""
    # Agent's own outbox dir
    from_agent = request_id.split("_")[2]
    outbox = TL_OUTBOX / from_agent
    outbox.mkdir(exist_ok=True)

    deadline = time.time() + timeout_sec
    while time.time() < deadline:
        for response_file in sorted(outbox.glob("*_response.json")):
            with open(response_file) as f:
                resp = json.load(f)
            if resp.get("request_id") == request_id:
                return resp
        time.sleep(5)
    return None


def process_inbox(once: bool = False):
    """
    TL runs this in a loop to handle incoming requests.
    once=True: process all pending and return (for testing)
    """
    while True:
        pending = sorted(TL_INBOX.glob("req_*.json"))

        for req_path in pending:
            with open(req_path) as f:
                req = json.load(f)

            print(f"[TL/BROKER] Handling {req['request_type']} from {req['from_agent']}")
            response = _route_request(req)
            _deliver_response(req, response)

            # Archive
            shutil.move(str(req_path), str(TL_INBOX / "processed" / req_path.name))
            audit.log("workitem_update",
                      target=req["from_agent"],
                      detail=f"{req['request_type']} → {response['status']}")

        if once:
            break
        time.sleep(5)


def _route_request(req: dict) -> dict:
    """Route a request to the correct handler."""
    handlers = {
        "knowledge_request":    _handle_knowledge,
        "file_request":         _handle_file,
        "clarification_request":_handle_clarification,
        "blocker_report":       _handle_blocker,
        "review_request":       _handle_review,
        "split_request":        _handle_split,
        "human_request":        _handle_human,
    }
    handler = handlers.get(req["request_type"], _handle_unknown)
    return handler(req)


def _deliver_response(req: dict, response: dict):
    """Write response to the requesting agent's outbox."""
    outbox = TL_OUTBOX / req["from_agent"]
    outbox.mkdir(exist_ok=True)
    ts = datetime.datetime.utcnow().strftime("%Y%m%d_%H%M%S")
    path = outbox / f"{ts}_response.json"
    with open(path, "w") as f:
        json.dump(response, f, indent=2)


# ── Handlers ──────────────────────────────────────────────────────────────

def _handle_knowledge(req: dict) -> dict:
    """Retrieve knowledge about another ticket from the store."""
    payload    = req["payload"]
    source_tid = payload.get("from_ticket")
    need       = payload.get("need", "")

    knowledge = store.get(source_tid) if source_tid else {}

    if not knowledge:
        # Check if source ticket is still running — it might not have produced yet
        result_f = f"/tmp/worktrees/{source_tid}/RESULT.json"
        if not os.path.exists(result_f):
            return {
                "request_id": req["request_id"],
                "to_agent":   req["from_agent"],
                "status":     "pending",
                "payload": {
                    "answer":     None,
                    "reason":     f"{source_tid} has not completed yet — retry in 60s",
                    "retry_after": 60
                }
            }

    # Extract the specific piece the agent needs
    answer = _extract_relevant_knowledge(knowledge, need)

    return {
        "request_id": req["request_id"],
        "to_agent":   req["from_agent"],
        "status":     "delivered",
        "payload": {
            "answer":     answer,
            "source":     f"{source_tid} knowledge store",
            "confidence": "high" if answer else "low"
        }
    }


def _handle_file(req: dict) -> dict:
    """Read a specific file from another agent's worktree."""
    payload     = req["payload"]
    source_tid  = payload.get("from_ticket")
    file_path   = payload.get("file_path")

    full_path = f"/tmp/worktrees/{source_tid}/{file_path}"

    if not os.path.exists(full_path):
        return {
            "request_id": req["request_id"],
            "to_agent":   req["from_agent"],
            "status":     "not_found",
            "payload":    {"error": f"{full_path} does not exist"}
        }

    # Safety check: only allow reading, never writing to another agent's worktree
    content = open(full_path).read()

    # Truncate if too large for context
    if len(content) > 8000:
        content = content[:8000] + f"\n\n[TRUNCATED — full file at {full_path}]"

    return {
        "request_id": req["request_id"],
        "to_agent":   req["from_agent"],
        "status":     "delivered",
        "payload": {"content": content, "path": file_path, "source_ticket": source_tid}
    }


def _handle_clarification(req: dict) -> dict:
    """Re-enhance the agent's prompt with more context."""
    from prompt_enhancer import enhance_prompt
    ticket_id = req["from_agent"]
    raw_task  = req["payload"].get("original_task", "")
    confusion = req["payload"].get("confusion", "")

    # Get repo path from worktree
    wt_path = f"/tmp/worktrees/{ticket_id}"

    enhanced = enhance_prompt(
        ticket_id=ticket_id,
        raw_task=f"{raw_task}\n\nSPECIFIC CONFUSION TO RESOLVE:\n{confusion}",
        repo_path=wt_path
    )

    return {
        "request_id": req["request_id"],
        "to_agent":   req["from_agent"],
        "status":     "delivered",
        "payload":    {"enhanced_prompt": enhanced}
    }


def _handle_blocker(req: dict) -> dict:
    """Agent is blocked — check if blocker has resolved."""
    blocking_ticket = req["payload"].get("blocked_by")
    result_f = f"/tmp/worktrees/{blocking_ticket}/RESULT.json"

    if os.path.exists(result_f):
        with open(result_f) as f:
            result = json.load(f)
        return {
            "request_id": req["request_id"],
            "to_agent":   req["from_agent"],
            "status":     "unblocked",
            "payload": {
                "blocking_ticket_status": "done",
                "knowledge":              result.get("produced", {})
            }
        }
    else:
        return {
            "request_id": req["request_id"],
            "to_agent":   req["from_agent"],
            "status":     "still_blocked",
            "payload": {"retry_after": 60}
        }


def _handle_review(req: dict) -> dict:
    """Route code review request — TL does a basic static check."""
    ticket_id = req["from_agent"]
    wt_path   = f"/tmp/worktrees/{ticket_id}"
    file_path = req["payload"].get("file_to_review")

    full_path = f"{wt_path}/{file_path}"
    if not os.path.exists(full_path):
        return {
            "request_id": req["request_id"],
            "to_agent":   req["from_agent"],
            "status":     "error",
            "payload":    {"error": f"File not found: {file_path}"}
        }

    content = open(full_path).read()
    issues  = _static_review(content)

    return {
        "request_id": req["request_id"],
        "to_agent":   req["from_agent"],
        "status":     "delivered",
        "payload": {
            "issues":     issues,
            "approved":   len(issues) == 0,
            "reviewer":   "team-leader/static-check"
        }
    }


def _handle_split(req: dict) -> dict:
    """Notify task manager that an agent needs its work split."""
    # Import and call task manager
    from task_manager import split_task
    ticket_id = req["from_agent"]
    reason    = req["payload"].get("reason", "task too large")
    split_task(ticket_id, reason)

    return {
        "request_id": req["request_id"],
        "to_agent":   req["from_agent"],
        "status":     "acknowledged",
        "payload":    {"message": "Task split queued — you will receive a narrowed prompt"}
    }


def _handle_human(req: dict) -> dict:
    """Escalate to human via notifications."""
    from notification_templates import notify
    ticket_id = req["from_agent"]
    reason    = req["payload"].get("reason", "agent needs human decision")
    question  = req["payload"].get("question", "")

    notify(
        title=f"Human input needed — {ticket_id}",
        message=f"Agent {ticket_id} is blocked and needs a decision:\n\n{question}",
        priority="warning"
    )

    return {
        "request_id": req["request_id"],
        "to_agent":   req["from_agent"],
        "status":     "escalated",
        "payload":    {"message": "Human notified. Agent should pause and wait."}
    }


def _handle_unknown(req: dict) -> dict:
    return {
        "request_id": req["request_id"],
        "to_agent":   req["from_agent"],
        "status":     "error",
        "payload":    {"error": f"Unknown request type: {req['request_type']}"}
    }


# ── Helpers ───────────────────────────────────────────────────────────────

def _extract_relevant_knowledge(knowledge: dict, need: str) -> str:
    """Extract the piece of knowledge most relevant to the need description."""
    need_lower = need.lower()

    if "class" in need_lower or "interface" in need_lower:
        classes = knowledge.get("new_classes", [])
        return f"Classes produced: {json.dumps(classes, indent=2)}" if classes else ""

    if "function" in need_lower or "method" in need_lower:
        funcs = knowledge.get("new_functions", [])
        return f"Functions produced: {json.dumps(funcs, indent=2)}" if funcs else ""

    if "endpoint" in need_lower or "api" in need_lower or "route" in need_lower:
        eps = knowledge.get("new_endpoints", [])
        return f"Endpoints produced: {json.dumps(eps, indent=2)}" if eps else ""

    if "schema" in need_lower or "database" in need_lower or "migration" in need_lower:
        schema = knowledge.get("db_schema_changes", [])
        return f"Schema changes: {json.dumps(schema, indent=2)}" if schema else ""

    # Fallback: return full produced block
    return json.dumps(knowledge, indent=2)


def _static_review(content: str) -> list[str]:
    """Basic static review — catch obvious issues."""
    issues = []
    lines  = content.split("\n")

    for i, line in enumerate(lines, 1):
        if "console.log" in line or "print(" in line:
            issues.append(f"Line {i}: debug output left in code")
        if "TODO(wip)" in line or "FIXME(wip)" in line:
            issues.append(f"Line {i}: WIP marker not resolved")
        if "hardcoded" in line.lower() or ("password" in line.lower() and "=" in line):
            issues.append(f"Line {i}: possible hardcoded credential")
        if line.strip().startswith("except:"):
            issues.append(f"Line {i}: bare except clause — catch specific exceptions")

    return issues
```

---

## Sub-Agent: How to Send a Request

Add this to any sub-agent that needs something from another agent:

```python
# In sub_agent_work.py — when you need something from another agent
import sys
sys.path.append("/tmp")   # or wherever message_broker.py is deployed
from message_broker import send_request, wait_for_response

# Ask TL for knowledge from another ticket
req_id = send_request(
    from_agent   = TICKET_ID,
    request_type = "knowledge_request",
    payload = {
        "need":        "PaymentGateway class interface",
        "from_ticket": "TICKET-101",
        "reason":      "Need to call .charge() in my implementation",
        "urgency":     "blocking"
    }
)

response = wait_for_response(req_id, timeout_sec=300)

if response and response["status"] == "delivered":
    gateway_info = response["payload"]["answer"]
    print(f"[AGENT] Got gateway info from TL: {gateway_info}")
elif response and response["status"] == "pending":
    # Source ticket still running — wait and retry
    import time
    time.sleep(response["payload"]["retry_after"])
    # re-send the request...
else:
    # Write clarification request
    send_request(
        from_agent   = TICKET_ID,
        request_type = "human_request",
        payload = {
            "reason":   "Cannot get PaymentGateway interface from TICKET-101",
            "question": "How should I call the payment service? TICKET-101 didn't produce expected output."
        }
    )
```
