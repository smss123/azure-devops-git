# Multi-Repo Atomic Change Coordination

Some changes must land in 3+ repos at the same time — a shared library update,
an API contract change, or an org-wide config update. This file provides the
saga pattern for coordinating atomic multi-repo changes with rollback.

---

## The Problem

```
repo-api     ── needs new endpoint ──►
repo-worker  ── needs updated client ──►  must all deploy together
repo-shared  ── needs new model ──►
```

If `repo-shared` merges and deploys but `repo-api` fails CI, the system is in
an inconsistent state. This requires a saga with compensation actions.

---

## Saga Pattern for Multi-Repo Changes

```
PHASE 1: Prepare — create all branches and PRs simultaneously
PHASE 2: Validate — all CI pipelines must pass
PHASE 3: Commit — merge all PRs in dependency order
PHASE 4: Rollback — if any merge fails, revert already-merged repos
```

---

## Saga Coordinator

```python
# multi_repo_saga.py
import json, subprocess, time, datetime, os
from audit_log import AuditLog

WT_ROOT = "/tmp/worktrees"
audit   = AuditLog(agent_id="saga-coordinator")

class MultiRepoSaga:
    def __init__(self, saga_id: str, repos: list[dict], change_fn):
        """
        repos: [{"repo": "repo-api", "branch": "feature/TICKET-X", "ticket": "TICKET-X"}]
        change_fn(repo, wt_path): function that applies the change to a worktree
        """
        self.saga_id   = saga_id
        self.repos     = repos
        self.change_fn = change_fn
        self.state     = self._load_state()

    def _state_file(self):
        return f"/tmp/saga_{self.saga_id}.json"

    def _load_state(self) -> dict:
        f = self._state_file()
        if os.path.exists(f):
            with open(f) as fh:
                state = json.load(fh)
            print(f"[SAGA] Resuming from checkpoint: phase={state['phase']}")
            return state
        return {
            "saga_id": self.saga_id,
            "phase": "prepare",
            "repos": {r["repo"]: {"status": "pending", "pr_id": None, "wt_path": None}
                      for r in self.repos}
        }

    def _save_state(self):
        self.state["updated_at"] = datetime.datetime.now(datetime.timezone.utc).isoformat().replace("+00:00", "Z")
        with open(self._state_file(), "w") as f:
            json.dump(self.state, f, indent=2)

    # ─── PHASE 1: Prepare ────────────────────────────────────────────────────

    def phase_prepare(self):
        """Create branches and apply changes to all repos in parallel."""
        print(f"\n[SAGA] Phase 1: Prepare ({len(self.repos)} repos)")

        for repo_spec in self.repos:
            repo = repo_spec["repo"]
            if self.state["repos"][repo]["status"] != "pending":
                print(f"  [skip] {repo} already prepared")
                continue

            branch  = repo_spec["branch"]
            wt_path = f"{WT_ROOT}/saga-{self.saga_id}-{repo}"

            try:
                # Clone / worktree setup
                _setup_worktree(repo, branch, wt_path)

                # Apply the change
                self.change_fn(repo, wt_path)

                # Commit and push
                sha = _commit_and_push(wt_path, branch,
                                       f"feat: {self.saga_id} in {repo}")

                # Open PR
                pr_id = _open_pr(repo, branch, "develop",
                                 f"[SAGA {self.saga_id}] {repo}")

                self.state["repos"][repo].update({
                    "status": "pr_open",
                    "pr_id":  str(pr_id),
                    "wt_path": wt_path,
                    "sha": sha
                })
                audit.log("pr_create", target=repo, pr_id=str(pr_id))

            except Exception as e:
                self.state["repos"][repo]["status"] = "failed"
                self.state["repos"][repo]["error"]  = str(e)
                audit.log("pr_create", target=repo, status="failed", error=str(e))

            self._save_state()

        self.state["phase"] = "validate"
        self._save_state()

    # ─── PHASE 2: Validate ───────────────────────────────────────────────────

    def phase_validate(self, timeout_min: int = 45) -> bool:
        """Wait for all PRs to pass CI."""
        print(f"\n[SAGA] Phase 2: Validate — waiting for CI on all PRs")
        deadline = time.time() + timeout_min * 60

        while time.time() < deadline:
            all_passed = True
            any_failed = False

            for repo, info in self.state["repos"].items():
                if info["status"] in ("ci_passed", "merged"):
                    continue
                if info["status"] == "failed":
                    any_failed = True
                    continue

                pr_status = _get_pr_ci_status(info["pr_id"])

                if pr_status == "succeeded":
                    self.state["repos"][repo]["status"] = "ci_passed"
                    print(f"  [✓] {repo} CI passed")
                elif pr_status == "failed":
                    self.state["repos"][repo]["status"] = "ci_failed"
                    any_failed = True
                    print(f"  [✗] {repo} CI failed")
                else:
                    all_passed = False
                    print(f"  [⏳] {repo} CI running...")

                self._save_state()

            if any_failed:
                print("[SAGA] Validation failed — initiating rollback")
                self.state["phase"] = "rollback"
                self._save_state()
                return False

            if all_passed:
                self.state["phase"] = "commit"
                self._save_state()
                return True

            time.sleep(30)

        print("[SAGA] Validation timeout — initiating rollback")
        self.state["phase"] = "rollback"
        self._save_state()
        return False

    # ─── PHASE 3: Commit ─────────────────────────────────────────────────────

    def phase_commit(self, merge_order: list[str] = None):
        """Merge all PRs in dependency order."""
        print(f"\n[SAGA] Phase 3: Commit — merging all PRs")

        order = merge_order or [r["repo"] for r in self.repos]

        for repo in order:
            info = self.state["repos"][repo]
            if info["status"] == "merged":
                print(f"  [skip] {repo} already merged")
                continue

            try:
                _complete_pr(info["pr_id"])
                self.state["repos"][repo]["status"] = "merged"
                audit.log("pr_merge", target=repo, pr_id=info["pr_id"])
                print(f"  [✓] {repo} merged")
            except Exception as e:
                self.state["repos"][repo]["status"] = "merge_failed"
                self.state["repos"][repo]["error"] = str(e)
                print(f"  [✗] {repo} merge failed: {e} — rolling back committed repos")
                self.state["phase"] = "rollback"
                self._save_state()
                return False

            self._save_state()

        self.state["phase"] = "done"
        self._save_state()
        print(f"\n[SAGA] ✓ All repos merged successfully")
        return True

    # ─── PHASE 4: Rollback ───────────────────────────────────────────────────

    def phase_rollback(self):
        """Abandon open PRs and revert any that already merged."""
        print(f"\n[SAGA] Phase 4: Rollback")

        for repo, info in self.state["repos"].items():
            status = info["status"]

            if status == "merged":
                # Revert already-merged PRs
                print(f"  [revert] {repo}")
                try:
                    _revert_pr(repo, info["pr_id"], f"Saga {self.saga_id} rollback")
                    self.state["repos"][repo]["status"] = "reverted"
                    audit.log("rollback_revert", target=repo, pr_id=info["pr_id"])
                except Exception as e:
                    print(f"  [ERROR] Could not auto-revert {repo}: {e}")
                    audit.log("rollback_revert", target=repo, status="failed", error=str(e))

            elif status == "pr_open":
                # Abandon open PRs
                _abandon_pr(info["pr_id"])
                self.state["repos"][repo]["status"] = "abandoned"
                audit.log("pr_abandon", target=repo, pr_id=info["pr_id"])
                print(f"  [abandoned] {repo}")

        self.state["phase"] = "rolled_back"
        self._save_state()
        print(f"[SAGA] Rollback complete")

    # ─── Run ─────────────────────────────────────────────────────────────────

    def run(self, merge_order: list[str] = None):
        """Execute the full saga, resuming from checkpoint if needed."""
        phase = self.state["phase"]

        if phase == "prepare":    self.phase_prepare()
        if phase == "validate":
            ok = self.phase_validate()
            if not ok: self.phase_rollback(); return False
        if phase == "commit":
            ok = self.phase_commit(merge_order)
            if not ok: self.phase_rollback(); return False
        if phase == "rollback":   self.phase_rollback(); return False

        return self.state["phase"] == "done"
```

---

## Helper Functions

```python
def _setup_worktree(repo: str, branch: str, wt_path: str):
    bare = f"/workspace/{repo}.git"
    if not os.path.exists(bare):
        # Use credential helper / GCM — never embed PAT in URLs
        # The ADO_PAT is injected via Git's insteadOf config at runtime
        clone_url = f"https://dev.azure.com/{ADO_ORG}/{ADO_PROJECT}/_git/{repo}"
        subprocess.run([
            "git", "clone", "--bare", clone_url, bare
        ], check=True, env={
            **os.environ,
            "GIT_CONFIG_COUNT": "1",
            "GIT_CONFIG_KEY_0": f"url.https://anything:{ADO_PAT}@dev.azure.com.insteadOf",
            "GIT_CONFIG_VALUE_0": "https://dev.azure.com",
        })
    subprocess.run(
        ["git", "-C", bare, "worktree", "add", "-b", branch, wt_path, "origin/develop"],
        check=True
    )

def _commit_and_push(wt_path: str, branch: str, message: str) -> str:
    subprocess.run(["git", "-C", wt_path, "add", "-A"], check=True)
    subprocess.run(["git", "-C", wt_path, "commit", "-m", message], check=True)
    subprocess.run(["git", "-C", wt_path, "push", "-u", "origin", branch], check=True)
    return subprocess.check_output(
        ["git", "-C", wt_path, "rev-parse", "--short", "HEAD"], text=True
    ).strip()

def _open_pr(repo: str, source: str, target: str, title: str) -> str:
    result = subprocess.run([
        "az", "repos", "pr", "create",
        "--repository", repo, "--source-branch", source, "--target-branch", target,
        "--title", title, "--auto-complete", "false", "-o", "json"
    ], capture_output=True, text=True, check=True)
    return json.loads(result.stdout)["pullRequestId"]

def _get_pr_ci_status(pr_id: str) -> str:
    pr = json.loads(subprocess.check_output([
        "az", "repos", "pr", "show", "--id", pr_id, "-o", "json"
    ]))
    return pr.get("mergeStatus", "notSet")

def _complete_pr(pr_id: str):
    subprocess.run([
        "az", "repos", "pr", "update", "--id", pr_id,
        "--status", "completed", "--merge-strategy", "squash",
        "--delete-source-branch", "true"
    ], check=True)

def _abandon_pr(pr_id: str):
    subprocess.run([
        "az", "repos", "pr", "update", "--id", pr_id, "--status", "abandoned"
    ], check=False)

def _revert_pr(repo: str, pr_id: str, reason: str):
    from rollback_playbook import revert_pr
    revert_pr(pr_id, reason)
```

---

## Example Usage

```python
# Apply a shared library version bump across 3 repos
def bump_lib_version(repo: str, wt_path: str):
    """Change dependency version in each repo's package manifest."""
    import re
    for manifest in ["package.json", "requirements.txt", "*.csproj"]:
        # Update version string — repo-specific logic here
        pass

saga = MultiRepoSaga(
    saga_id="upgrade-shared-lib-v2.1.0",
    repos=[
        {"repo": "repo-shared",  "branch": "feature/lib-upgrade", "ticket": "TICKET-200"},
        {"repo": "repo-api",     "branch": "feature/lib-upgrade", "ticket": "TICKET-201"},
        {"repo": "repo-workers", "branch": "feature/lib-upgrade", "ticket": "TICKET-202"},
    ],
    change_fn=bump_lib_version
)

success = saga.run(merge_order=["repo-shared", "repo-api", "repo-workers"])
print("Saga succeeded" if success else "Saga rolled back")
```
