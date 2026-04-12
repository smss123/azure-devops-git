# Changelog & Release Notes Automation

Auto-generate `CHANGELOG.md` and ADO release notes from conventional commits
between tags. No manual writing required.

---

## Convention → Changelog Mapping

| Commit prefix | Changelog section | Semver bump |
|---|---|---|
| `feat:` | ✨ Features | minor |
| `fix:` | 🐛 Bug Fixes | patch |
| `feat!:` or `BREAKING CHANGE:` | 💥 Breaking Changes | major |
| `perf:` | ⚡ Performance | patch |
| `refactor:` | 🔧 Internal | — (no release note) |
| `docs:` | 📝 Documentation | — |
| `ci:` | — | — (excluded) |
| `chore:` | — | — (excluded) |
| `wip:` | — | — (excluded) |

---

## Changelog Generator

```python
# generate_changelog.py
import subprocess, re, json, datetime, os

def get_commits_since_tag(from_tag: str, to_ref: str = "HEAD") -> list[dict]:
    """Get all commits between from_tag and to_ref."""
    raw = subprocess.check_output([
        "git", "log", f"{from_tag}..{to_ref}",
        "--pretty=format:%H|%s|%b|%an",
        "--no-merges"
    ], text=True).strip()

    if not raw:
        return []

    commits = []
    for line in raw.split("\n"):
        parts = line.split("|", 3)
        if len(parts) < 3:
            continue
        sha, subject, body, author = parts[0], parts[1], parts[2], parts[3] if len(parts) > 3 else ""
        commits.append({
            "sha": sha[:8],
            "subject": subject,
            "body": body,
            "author": author,
            "breaking": "BREAKING CHANGE" in body or subject.endswith("!") or "!:" in subject
        })
    return commits

def parse_conventional(subject: str) -> tuple[str, str, str]:
    """Parse 'feat(scope): description AB#123' → (type, scope, description)"""
    m = re.match(r"^(\w+)(?:\(([^)]+)\))?!?:\s*(.+?)(?:\s+AB#\d+)?$", subject)
    if m:
        return m.group(1), m.group(2) or "", m.group(3)
    return "other", "", subject

def compute_next_version(current_tag: str, commits: list[dict]) -> str:
    """Compute next semver from commit types."""
    # Strip 'v' prefix
    version = current_tag.lstrip("v")
    parts = version.split(".")
    major, minor, patch = int(parts[0]), int(parts[1]), int(parts[2] if len(parts) > 2 else 0)

    bump = "patch"
    for c in commits:
        ctype, _, _ = parse_conventional(c["subject"])
        if c["breaking"]:
            bump = "major"
            break
        if ctype == "feat" and bump != "major":
            bump = "minor"

    if bump == "major":   major += 1; minor = 0; patch = 0
    elif bump == "minor": minor += 1; patch = 0
    else:                 patch += 1

    return f"v{major}.{minor}.{patch}"

def generate_changelog_entry(version: str, commits: list[dict], date: str = None) -> str:
    """Generate a CHANGELOG.md entry for one release."""
    date = date or datetime.date.today().isoformat()

    sections = {
        "feat":    ("✨ Features",       []),
        "fix":     ("🐛 Bug Fixes",      []),
        "perf":    ("⚡ Performance",    []),
        "docs":    ("📝 Documentation",  []),
        "breaking":("💥 Breaking Changes",[]),
    }

    for c in commits:
        ctype, scope, desc = parse_conventional(c["subject"])
        # Extract work item link
        wi_match = re.search(r"AB#(\d+)", c["subject"])
        wi_link = f" ([#{wi_match.group(1)}])" if wi_match else ""
        scope_str = f"**{scope}**: " if scope else ""
        entry = f"- {scope_str}{desc}{wi_link} (`{c['sha']}`)"

        if c["breaking"]:
            sections["breaking"][1].append(entry)
        elif ctype in sections:
            sections[ctype][1].append(entry)

    lines = [f"## [{version}] — {date}", ""]
    for key in ["breaking", "feat", "fix", "perf", "docs"]:
        title, entries = sections[key]
        if entries:
            lines += [f"### {title}", ""] + entries + [""]

    return "\n".join(lines)

def update_changelog_file(new_entry: str, changelog_path: str = "CHANGELOG.md"):
    """Prepend new entry to existing CHANGELOG.md."""
    header = "# Changelog\n\nAll notable changes to this project.\n\n"

    if os.path.exists(changelog_path):
        existing = open(changelog_path).read()
        # Remove header if present
        existing = re.sub(r"^# Changelog.*?\n\n", "", existing, flags=re.DOTALL)
    else:
        existing = ""

    with open(changelog_path, "w") as f:
        f.write(header + new_entry + "\n" + existing)

    print(f"[CHANGELOG] Updated {changelog_path}")
```

---

## Full Release Notes Generation Script

```bash
#!/bin/bash
# generate_release_notes.sh — run during release branch creation
# Usage: ./generate_release_notes.sh v1.1.0 v1.2.0

set -euo pipefail

FROM_TAG=$1   # previous release tag
TO_REF=${2:-HEAD}

python3 - <<EOF
from generate_changelog import *

commits = get_commits_since_tag("$FROM_TAG", "$TO_REF")
print(f"Found {len(commits)} commits since $FROM_TAG")

current_tag = subprocess.check_output(["git", "describe", "--tags", "--abbrev=0", "$FROM_TAG"],
                                       text=True).strip()
next_ver = compute_next_version(current_tag, commits)
print(f"Next version: {next_ver}")

entry = generate_changelog_entry(next_ver, commits)
print("\n--- CHANGELOG ENTRY ---")
print(entry)

# Write to file
update_changelog_file(entry)

# Also write a separate RELEASE_NOTES.md for this release
with open("RELEASE_NOTES.md", "w") as f:
    f.write(entry)

print(f"\n✓ CHANGELOG.md updated")
print(f"✓ RELEASE_NOTES.md written")
print(f"✓ Suggested tag: {next_ver}")

# Write version to file
with open("VERSION", "w") as f:
    f.write(next_ver.lstrip("v") + "\n")
EOF
```

---

## Post Release Notes to ADO Wiki

```python
def post_to_ado_wiki(version: str, content: str, wiki_name: str = "ProjectWiki"):
    """Create or update a wiki page with release notes."""
    import requests
    from base64 import b64encode

    headers = {
        "Authorization": f"Basic {b64encode(f':{ADO_PAT}'.encode()).decode()}",
        "Content-Type": "application/json"
    }

    page_path = f"/Releases/{version}"

    # Try to create (PUT upserts)
    r = requests.put(
        f"{ADO_ORG}/{ADO_PROJECT}/_apis/wiki/wikis/{wiki_name}/pages"
        f"?path={page_path}&api-version=7.1",
        headers=headers,
        json={"content": content}
    )
    r.raise_for_status()
    print(f"[WIKI] Release notes posted: {page_path}")
```

---

## Pipeline Integration

```yaml
# In release branch pipeline
- stage: ReleaseNotes
  condition: startsWith(variables['Build.SourceBranch'], 'refs/heads/release/')
  jobs:
    - job: GenerateNotes
      steps:
        - checkout: self
          fetchDepth: 0    # full history needed for git log

        - script: |
            PREV_TAG=$(git describe --tags --abbrev=0 HEAD^)
            echo "Generating notes from $PREV_TAG to HEAD"
            python3 generate_release_notes.py $PREV_TAG HEAD
          displayName: Generate CHANGELOG and release notes

        - script: |
            git config user.email "devops-bot@company.com"
            git config user.name "DevOps Bot"
            git add CHANGELOG.md VERSION RELEASE_NOTES.md
            git diff --cached --quiet || git commit -m "chore: update changelog for $(cat VERSION)"
            git push
          displayName: Commit changelog update
```

---

## ADO Release Notes (native)

```bash
# Attach RELEASE_NOTES.md as a pipeline artifact
- task: PublishPipelineArtifact@1
  inputs:
    targetPath: RELEASE_NOTES.md
    artifact: release-notes

# Create an ADO Release (classic) with the notes
az devops invoke \
  --area release \
  --resource releases \
  --http-method POST \
  --api-version 7.1 \
  --in-file release_def.json
```
