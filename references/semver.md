# Semantic Versioning Automation

Auto-calculate the next version from conventional commits. No manual version
bumping. Integrates with git flow release creation and CHANGELOG generation.

---

## Semver Rules

```
Given version MAJOR.MINOR.PATCH:

BREAKING CHANGE  → MAJOR bump  (1.2.3 → 2.0.0)
feat:            → MINOR bump  (1.2.3 → 1.3.0)
fix: / perf:     → PATCH bump  (1.2.3 → 1.2.4)
chore/docs/style → NO bump     (no release needed)
```

---

## Core Version Calculator

```python
# semver.py
import subprocess, re, os

def get_latest_tag() -> str:
    """Get the most recent semver tag, or v0.0.0 if none."""
    result = subprocess.run(
        ["git", "describe", "--tags", "--abbrev=0", "--match", "v[0-9]*"],
        capture_output=True, text=True
    )
    return result.stdout.strip() if result.returncode == 0 else "v0.0.0"

def get_commits_since(tag: str) -> list[str]:
    """Get commit subjects AND bodies since a tag (needed for BREAKING CHANGE in footer)."""
    result = subprocess.run(
        ["git", "log", f"{tag}..HEAD", "--pretty=format:%B%x00", "--no-merges"],
        capture_output=True, text=True
    )
    # Split on null delimiter, one entry per commit (subject + body)
    raw = result.stdout.strip().split("\x00") if result.stdout.strip() else []
    return [c.strip() for c in raw if c.strip()]

def determine_bump(commits: list[str]) -> str:
    """Read commit messages (subject + body) and determine the correct semver bump."""
    bump = None   # None = no release needed

    for commit in commits:
        lines = commit.splitlines()
        subject = lines[0].strip() if lines else ""
        body    = "\n".join(lines[1:]) if len(lines) > 1 else ""

        # BREAKING CHANGE in subject — any type with ! suffix is allowed per Conventional Commits spec
        if re.search(r"^(\w+)(\([^)]+\))?!:", subject):
            return "major"
        # BREAKING CHANGE footer/body (conventional commits spec §8)
        if re.search(r"^BREAKING CHANGE:", body, re.MULTILINE):
            return "major"
        # Feature
        if re.match(r"^feat(\([^)]+\))?:", subject):
            if bump != "major":
                bump = "minor"
        # Fix or perf
        elif re.match(r"^(fix|perf)(\([^)]+\))?:", subject):
            if bump not in ("major", "minor"):
                bump = "patch"

    return bump   # None if no release-worthy commits

def bump_version(current: str, bump: str) -> str:
    """Apply a bump to a version string."""
    current = current.lstrip("v")
    parts = current.split(".")
    major, minor, patch = int(parts[0]), int(parts[1]), int(parts[2] if len(parts) > 2 else 0)

    if bump == "major":
        major += 1
        minor = 0
        patch = 0
    elif bump == "minor": minor += 1; patch = 0
    elif bump == "patch": patch += 1
    else:
        return f"v{major}.{minor}.{patch}"   # no bump

    return f"v{major}.{minor}.{patch}"

def compute_next_version(repo_path: str = ".") -> dict:
    """
    Full calculation: current tag → commits → next version.
    Returns {"current": "v1.2.3", "bump": "minor", "next": "v1.3.0", "commits": [...]}
    """
    os.chdir(repo_path)
    current = get_latest_tag()
    commits = get_commits_since(current)
    bump = determine_bump(commits)

    if bump is None:
        return {
            "current": current,
            "bump": None,
            "next": current,
            "commits": [c.splitlines()[0] for c in commits],
            "release_needed": False
        }

    next_ver = bump_version(current, bump)
    return {
        "current": current,
        "bump": bump,
        "next": next_ver,
        "commits": [c.splitlines()[0] for c in commits],
        "release_needed": True
    }
```

---

## Version Bump Script (used during release)

```bash
#!/bin/bash
# bump_version.sh — run at start of release branch creation
set -euo pipefail

cd "${1:-.}"

python3 - <<'EOF'
from semver import compute_next_version
import json, sys

result = compute_next_version()
print(json.dumps(result, indent=2))

if not result["release_needed"]:
    print("\nNo releaseable commits found — nothing to release", file=sys.stderr)
    sys.exit(0)

print(f"\nBumping {result['current']} → {result['next']} ({result['bump']})")

# Write to VERSION file
with open("VERSION", "w") as f:
    f.write(result["next"].lstrip("v") + "\n")

print(f"VERSION file updated to: {result['next'].lstrip('v')}")
EOF
```

---

## Integration with Git Flow Release

```bash
#!/bin/bash
# gitflow_release_with_semver.sh
set -euo pipefail

# 1. Compute next version from commits
RESULT=$(python3 -c "
from semver import compute_next_version
import json
r = compute_next_version()
print(json.dumps(r))
")

NEXT=$(echo $RESULT | python3 -c "import sys,json; print(json.load(sys.stdin)['next'])")
BUMP=$(echo $RESULT | python3 -c "import sys,json; print(json.load(sys.stdin)['bump'])")

echo "Next version: $NEXT (bump: $BUMP)"

if [ "$BUMP" = "null" ] || [ -z "$BUMP" ]; then
  echo "No releaseable commits — skipping release"
  exit 0
fi

# 2. Create release branch
git checkout develop && git pull origin develop
git checkout -b "release/$NEXT"

# 3. Bump VERSION + generate CHANGELOG
echo "${NEXT#v}" > VERSION
python3 generate_changelog.py $(git describe --tags --abbrev=0 develop) HEAD

git add VERSION CHANGELOG.md
git commit -m "chore: release $NEXT"
git push -u origin "release/$NEXT"

echo "Release branch ready: release/$NEXT"
echo "Next: open PR from release/$NEXT → main"
```

---

## CI Pipeline: Auto-detect Version

```yaml
# In build pipeline — set BUILD_VERSION variable for use downstream
- script: |
    python3 - <<'EOF'
    from semver import compute_next_version
    r = compute_next_version()
    print(f"##vso[task.setvariable variable=buildVersion]{r['next']}")
    print(f"##vso[task.setvariable variable=semverBump]{r['bump'] or 'none'}")
    print(f"Current: {r['current']} → Next: {r['next']} ({r['bump']})")
    EOF
  displayName: Compute semver

- script: echo "Building version $(buildVersion)"
  displayName: Use computed version
```

---

## Prerelease & Build Metadata

```python
def make_prerelease_version(base: str, branch: str, build_id: str) -> str:
    """
    Generate a prerelease version for non-release builds.
    Examples:
      develop → v1.3.0-dev.20250314.42
      feature/TICKET-123 → v1.3.0-TICKET-123.42
    """
    base = base.lstrip("v")
    from datetime import date

    if branch == "develop":
        pre = f"dev.{date.today().strftime('%Y%m%d')}.{build_id}"
    elif branch.startswith("feature/"):
        ticket = branch.split("/")[-1][:20]    # cap length
        pre = f"{ticket}.{build_id}"
    elif branch.startswith("release/"):
        pre = f"rc.{build_id}"
    else:
        pre = f"ci.{build_id}"

    return f"v{base}-{pre}"

# Examples:
# develop  → v1.3.0-dev.20250314.42
# release  → v1.3.0-rc.42
# feature  → v1.3.0-TICKET-123.42
```
