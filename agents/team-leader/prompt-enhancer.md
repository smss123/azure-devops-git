# Prompt Enhancer

The TL rewrites every sub-agent task description before the agent sees it.
A vague prompt produces vague work. A precise prompt produces precise work.

---

## The Enhancement Pipeline

```
Raw task (from ADO / user)
        │
        ▼
1. ENRICH     — add repo context, file structure, related tickets
2. CONSTRAIN  — add explicit boundaries (what NOT to touch)
3. CALIBRATE  — set output format, quality bar, commit style
4. CONNECT    — inject knowledge from agents that already ran
5. ANTICIPATE — pre-answer likely questions the agent will hit
        │
        ▼
Enhanced prompt → sub-agent
```

---

## Enhancement Rules

### Rule 1 — Never leave a vague verb
Replace every ambiguous word with a specific action.

| Vague | Enhanced |
|---|---|
| "Update the auth module" | "Add JWT expiry validation to `src/auth/token_validator.py::validate()`. Token should reject if `exp` claim is missing or past." |
| "Fix the bug" | "Fix the KeyError on line 47 of `src/api/users.py` when `profile` key is absent in the response dict. Add a `.get()` with default `{}`." |
| "Add tests" | "Add 3 pytest unit tests to `tests/test_payments.py` covering: (1) successful charge, (2) declined card returns 402, (3) missing amount raises ValueError." |
| "Clean up" | "Remove dead code: the `legacy_format()` function in `src/utils.py` has no callers. Delete it and its import in `main.py`." |

### Rule 2 — Always include the acceptance boundary
Tell the agent exactly where its responsibility ends.

```
YOUR SCOPE (implement these):
  - src/auth/token_validator.py — add expiry check
  - tests/test_token_validator.py — add 2 tests

OUT OF SCOPE (do not touch):
  - src/auth/oauth_provider.py — owned by TICKET-102
  - Any database migration files
  - package.json / requirements.txt (no new dependencies)
```

### Rule 3 — Inject prior agent knowledge
If agent A produced something that agent B needs, the TL injects it.

```
CONTEXT FROM PREVIOUS AGENTS:
  TICKET-101 (merged): Added `PaymentGateway` class to src/payments/gateway.py
    - Constructor: PaymentGateway(api_key: str, timeout: int = 30)
    - Method: .charge(amount: float, currency: str) -> ChargeResult
    - ChargeResult has fields: transaction_id, status, error_message

  Your task uses this class. Import it as:
    from payments.gateway import PaymentGateway, ChargeResult
```

### Rule 4 — Set the quality bar explicitly

```
QUALITY REQUIREMENTS:
  - All new functions must have docstrings
  - New code must follow existing style: snake_case, type hints required
  - No print() statements (use logger = logging.getLogger(__name__))
  - Test coverage for new code must be ≥ 80%
  - Run: pytest tests/ --tb=short before committing
  - Commit message format: feat(payments): <description> AB#TICKET_ID
```

### Rule 5 — Anticipate blockers
Answer the questions the agent is likely to ask before it asks them.

```
LIKELY QUESTIONS (pre-answered):
  Q: Where is the database connection configured?
  A: src/config/database.py — use get_db_connection() from there

  Q: How are errors returned from the API?
  A: All API endpoints return {"error": "message", "code": N} on failure
     See src/api/base.py::APIError for the pattern

  Q: What test fixtures exist?
  A: tests/fixtures/payments.py has mock_gateway() and sample_charge_response()
```

---

## Enhancer Implementation

```python
# prompt_enhancer.py
import json, os
from knowledge_store import KnowledgeStore

store = KnowledgeStore()

def enhance_prompt(
    ticket_id: str,
    raw_task: str,
    repo_path: str,
    related_tickets: list[str] = None,
    agent_role: str = "implementer"
) -> str:
    """
    Take a raw task description and produce a precise, contextual agent prompt.
    """
    sections = []

    # ── Header ────────────────────────────────────────────────────────────
    sections.append(f"# Task Brief — {ticket_id}\n")
    sections.append(f"**Your role:** {agent_role}")
    sections.append(f"**Work item:** AB#{ticket_id.replace('TICKET-', '')}\n")

    # ── Enhanced task ─────────────────────────────────────────────────────
    enhanced_task = _enhance_task_description(raw_task, repo_path)
    sections.append(f"## What to implement\n\n{enhanced_task}\n")

    # ── Scope boundary ────────────────────────────────────────────────────
    scope = _compute_scope(ticket_id, repo_path, related_tickets or [])
    sections.append(f"## Your scope\n\n{scope}\n")

    # ── Prior agent knowledge ─────────────────────────────────────────────
    context = _build_context(related_tickets or [])
    if context:
        sections.append(f"## Context from other agents\n\n{context}\n")

    # ── Quality bar ───────────────────────────────────────────────────────
    quality = _quality_bar(repo_path)
    sections.append(f"## Quality requirements\n\n{quality}\n")

    # ── Anticipated blockers ──────────────────────────────────────────────
    blockers = _anticipate_blockers(repo_path, raw_task)
    if blockers:
        sections.append(f"## Pre-answered questions\n\n{blockers}\n")

    # ── Output contract ───────────────────────────────────────────────────
    sections.append(_output_contract(ticket_id))

    return "\n".join(sections)


def _enhance_task_description(raw: str, repo_path: str) -> str:
    """
    Replace vague verbs with specific file+function targets.
    In production: call LLM with repo context. Here: rule-based enhancement.
    """
    # Detect vague patterns and flag them
    vague_patterns = [
        ("update the", "Specify: which file, which function, what change"),
        ("fix the bug", "Specify: file path, line number, expected vs actual behaviour"),
        ("add tests", "Specify: test file, function names, scenarios to cover"),
        ("clean up", "Specify: which file, what exactly to remove/rename"),
        ("improve", "Specify: what metric improves, by how much, in which file"),
        ("refactor", "Specify: which class/function, target pattern, why"),
    ]

    flags = []
    for pattern, advice in vague_patterns:
        if pattern.lower() in raw.lower():
            flags.append(f"⚠️  '{pattern}' is vague — {advice}")

    result = raw
    if flags:
        result += "\n\n**TL note — clarify these before starting:**\n"
        result += "\n".join(flags)

    return result


def _compute_scope(ticket_id: str, repo_path: str, related_tickets: list) -> str:
    """Build an explicit in-scope / out-of-scope boundary."""
    # Read DEPS.json if present
    deps_file = f"{os.environ.get('WT_ROOT', '/tmp/worktrees')}/{ticket_id}/DEPS.json"
    deps = []
    if os.path.exists(deps_file):
        with open(deps_file) as f:
            deps = json.load(f).get("dependency_files", [])

    lines = ["**In scope (you own these):**"]
    lines.append("  — (derived from ticket description and repo structure)")
    lines.append("")
    lines.append("**Out of scope (do not modify):**")
    for dep_file in deps:
        lines.append(f"  - {dep_file}  ← owned by dependency ticket")
    for related in related_tickets:
        lines.append(f"  - Files claimed by {related}")
    lines.append("  - Any DB migration files (require human review)")
    lines.append("  - CI/CD pipeline YAML (unless your ticket specifically covers it)")

    return "\n".join(lines)


def _build_context(related_tickets: list) -> str:
    """Inject knowledge produced by agents that already completed."""
    if not related_tickets:
        return ""

    lines = []
    for ticket in related_tickets:
        knowledge = store.get(ticket)
        if not knowledge:
            continue
        lines.append(f"**{ticket}** produced:")
        for k, v in knowledge.items():
            if k in ("new_classes", "new_functions", "new_endpoints", "db_schema_changes"):
                lines.append(f"  {k}: {json.dumps(v, indent=4)}")
        lines.append("")

    return "\n".join(lines) if lines else ""


def _quality_bar(repo_path: str) -> str:
    """Detect project language and return appropriate quality requirements."""
    is_python  = os.path.exists(f"{repo_path}/requirements.txt") or \
                 os.path.exists(f"{repo_path}/pyproject.toml")
    is_dotnet  = bool(list(__import__('glob').glob(f"{repo_path}/**/*.csproj", recursive=True)))
    is_node    = os.path.exists(f"{repo_path}/package.json")

    if is_python:
        return """- Type hints on all new functions
- Docstrings on all new public functions/classes
- No bare `except:` — always catch specific exception types
- No `print()` — use `logging.getLogger(__name__)`
- Run `pytest --tb=short` before committing — all tests must pass
- New code coverage ≥ 80%"""

    if is_dotnet:
        return """- XML doc comments on all public methods
- No magic strings — use constants or enums
- Async methods must be awaited — no `.Result` or `.Wait()`
- Run `dotnet test` before committing — all tests must pass
- New code coverage ≥ 80%"""

    if is_node:
        return """- JSDoc on all exported functions
- No `var` — use `const` / `let`
- No `console.log` — use the project logger
- Run `npm test` before committing — all tests must pass
- New code coverage ≥ 80%"""

    return "- Follow existing code style\n- All tests must pass before committing"


def _anticipate_blockers(repo_path: str, task: str) -> str:
    """
    Scan repo structure and task description to pre-answer likely questions.
    This is the most valuable part of prompt enhancement.
    """
    lines = []

    # Database config
    db_configs = [
        "src/config/database.py", "config/db.py", "app/db.py",
        "src/data/connection.py", "database.py"
    ]
    for f in db_configs:
        if os.path.exists(f"{repo_path}/{f}"):
            lines.append(f"**Database connection:** `{f}` — use the factory function there")
            break

    # Test fixtures
    fixture_dirs = ["tests/fixtures", "tests/conftest.py", "test/helpers"]
    for d in fixture_dirs:
        if os.path.exists(f"{repo_path}/{d}"):
            lines.append(f"**Test fixtures:** available in `{d}`")
            break

    # Error handling pattern
    error_files = ["src/api/base.py", "src/core/errors.py", "app/exceptions.py"]
    for f in error_files:
        if os.path.exists(f"{repo_path}/{f}"):
            lines.append(f"**Error pattern:** see `{f}` for the project error convention")
            break

    return "\n".join(lines)


def _output_contract(ticket_id: str) -> str:
    return f"""## Output contract

When done, write `/tmp/worktrees/{ticket_id}/RESULT.json`:
```json
{{
  "ticket": "{ticket_id}",
  "status": "done",
  "branch": "feature/{ticket_id}",
  "commit": "<short sha>",
  "pr_id": "<pr number>",
  "produced": {{
    "new_functions": ["module.function_name"],
    "new_classes":   ["module.ClassName"],
    "new_endpoints": ["POST /api/path"],
    "files_changed": ["src/file.py", "tests/test_file.py"]
  }},
  "notes_for_team_leader": "<anything the TL should know for other agents>"
}}
```

The `produced` block is critical — the TL uses it to brief dependent agents."""
```

---

## Enhancement Quality Checklist

Before sending any prompt to a sub-agent, the TL verifies:

- [ ] No vague verbs (update, fix, improve, clean, add) without a specific target
- [ ] File paths are explicit (not "the auth module" but `src/auth/token_validator.py`)
- [ ] Scope boundary is clear — what NOT to touch is listed
- [ ] Context from already-completed agents is injected (if any)
- [ ] Quality requirements match the project language
- [ ] Output contract (RESULT.json `produced` block) is specified
- [ ] At least 2 pre-answered questions anticipate likely blockers
- [ ] Commit message format is specified with the AB# link
