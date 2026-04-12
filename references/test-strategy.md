# Test Strategy & Quality Gates

"Tests pass" is a finish criterion in the decision engine but this reference
defines *what* tests, *what thresholds*, and *how* to wire them into the pipeline.

---

## Test Pyramid for ADO Projects

```
                    ┌─────────────────┐
                    │   E2E / UI      │  few, slow, high confidence
                    │   Playwright    │  run on: release branch only
                    └────────┬────────┘
               ┌─────────────┴─────────────┐
               │     Integration Tests      │  moderate count, medium speed
               │     real DB, real queue    │  run on: PR + develop
               └──────────────┬────────────┘
          ┌────────────────────┴────────────────────┐
          │              Unit Tests                  │  many, fast, focused
          │  no I/O, mocked deps, covers edge cases  │  run on: every commit
          └─────────────────────────────────────────┘
```

---

## Coverage Thresholds by Branch

| Branch | Minimum Coverage | Gate |
|---|---|---|
| `feature/*` | 70% (lines) | Warning only — don't block |
| `develop` | 80% (lines + branches) | **Block merge if below** |
| `main` | 85% (lines + branches) | **Block merge if below** |
| `release/*` | 85% | **Block release if below** |

---

## Pipeline Quality Gate — YAML

```yaml
- stage: QualityGate
  displayName: Test & Quality Gate
  jobs:
    - job: Tests
      steps:
        # Unit tests + coverage
        - task: DotNetCoreCLI@2
          displayName: Run unit tests
          inputs:
            command: test
            arguments: >
              --configuration $(buildConfiguration)
              --collect:"XPlat Code Coverage"
              --results-directory $(Agent.TempDirectory)/coverage
              --filter "Category!=Integration&Category!=E2E"

        # Publish coverage
        - task: reportgenerator@5
          inputs:
            reports: $(Agent.TempDirectory)/coverage/**/*.xml
            targetdir: $(Build.SourcesDirectory)/coveragereport
            reporttypes: Cobertura;HtmlInline_AzurePipelines

        - task: PublishCodeCoverageResults@2
          inputs:
            summaryFileLocation: coveragereport/Cobertura.xml

        # Coverage threshold enforcement
        - script: |
            COVERAGE=$(python3 -c "
            import xml.etree.ElementTree as ET
            tree = ET.parse('coveragereport/Cobertura.xml')
            root = tree.getroot()
            rate = float(root.attrib.get('line-rate', 0)) * 100
            print(f'{rate:.1f}')
            ")
            echo "Coverage: ${COVERAGE}%"
            THRESHOLD=80
            if (( $(echo "$COVERAGE < $THRESHOLD" | bc -l) )); then
              echo "##vso[task.logissue type=error]Coverage ${COVERAGE}% is below threshold ${THRESHOLD}%"
              exit 1
            fi
          displayName: Enforce coverage threshold

        # Integration tests (on develop/main only)
        - task: DotNetCoreCLI@2
          displayName: Run integration tests
          condition: |
            or(
              eq(variables['Build.SourceBranch'], 'refs/heads/develop'),
              eq(variables['Build.SourceBranch'], 'refs/heads/main'),
              startsWith(variables['Build.SourceBranch'], 'refs/heads/release/')
            )
          inputs:
            command: test
            arguments: --filter "Category=Integration"
```

---

## Quality Gates Matrix

| Gate | Feature PR | Develop | Main | Release |
|---|---|---|---|---|
| Unit tests pass | ✅ Required | ✅ Required | ✅ Required | ✅ Required |
| Coverage ≥ 70% | ⚠️ Warning | ✅ Required (80%) | ✅ Required (85%) | ✅ Required (85%) |
| Integration tests | ❌ Optional | ✅ Required | ✅ Required | ✅ Required |
| E2E tests | ❌ Skip | ❌ Skip | ✅ Required | ✅ Required |
| Static analysis (SonarQube/CodeQL) | ⚠️ Warning | ✅ Required | ✅ Required | ✅ Required |
| Dependency vulnerability scan | ⚠️ Warning | ✅ Required | ✅ Required | ✅ Required |
| No TODO(wip) markers | ✅ Required | ✅ Required | ✅ Required | ✅ Required |
| API contract test | ❌ Optional | ✅ Required | ✅ Required | ✅ Required |

---

## Agent Test Execution (in worktree)

```python
# test_runner.py — run in sub-agent before declaring feature done

import subprocess, json, os

def run_tests(wt_path: str, language: str = "dotnet") -> dict:
    """Run tests in the agent's worktree. Returns pass/fail + coverage."""

    runners = {
        "dotnet": ["dotnet", "test", "--collect:XPlat Code Coverage", "--logger", "trx"],
        "python": ["pytest", "--cov=.", "--cov-report=json", "-q"],
        "node":   ["npm", "test", "--", "--coverage", "--watchAll=false"],
    }

    cmd = runners.get(language)
    if not cmd:
        raise ValueError(f"Unknown language: {language}")

    result = subprocess.run(cmd, cwd=wt_path, capture_output=True, text=True)

    passed = result.returncode == 0
    coverage = extract_coverage(wt_path, language)

    return {
        "passed": passed,
        "coverage": coverage,
        "stdout": result.stdout[-2000:],   # last 2000 chars
        "stderr": result.stderr[-500:] if not passed else ""
    }

def extract_coverage(wt_path: str, language: str) -> float | None:
    """Extract coverage percentage from test output."""
    if language == "python":
        cov_file = f"{wt_path}/coverage.json"
        if os.path.exists(cov_file):
            with open(cov_file) as f:
                data = json.load(f)
            return round(data.get("totals", {}).get("percent_covered", 0), 1)
    # For dotnet and node, parse from stdout or report file
    return None

def assert_quality_gate(test_result: dict, branch_type: str = "feature"):
    """Raise an error if the quality gate fails."""
    thresholds = {
        "feature": 70,
        "develop": 80,
        "main":    85,
        "release": 85,
    }
    threshold = thresholds.get(branch_type, 70)

    issues = []
    if not test_result["passed"]:
        issues.append(f"Tests failed:\n{test_result['stdout'][-500:]}")

    cov = test_result.get("coverage")
    if cov is not None and cov < threshold:
        issues.append(f"Coverage {cov}% is below {threshold}% threshold for {branch_type}")

    if issues:
        raise RuntimeError("Quality gate failed:\n" + "\n".join(issues))

    print(f"[GATE] Quality gate passed — coverage: {cov}%")
```

---

## Static Analysis Gate

```yaml
# Add to pipeline after build
- task: SonarCloudAnalyze@1
  displayName: SonarCloud analysis

- task: SonarCloudPublish@1
  inputs:
    pollingTimeoutSec: 300

# Block on quality gate failure
- script: |
    STATUS=$(curl -s -u "$SONAR_TOKEN:" \
      "https://sonarcloud.io/api/qualitygates/project_status?projectKey=$SONAR_PROJECT_KEY" \
      | python3 -c "import sys,json; print(json.load(sys.stdin)['projectStatus']['status'])")
    echo "SonarCloud quality gate: $STATUS"
    [ "$STATUS" = "OK" ] || exit 1
  displayName: Enforce SonarCloud quality gate
```

---

## Dependency Vulnerability Scan

```yaml
# OWASP / npm audit / pip-audit
- script: |
    # For Node.js
    npm audit --audit-level=high
    # Fails if any HIGH or CRITICAL vulnerabilities found

    # For Python
    pip install pip-audit --break-system-packages
    pip-audit --severity high

    # For .NET
    dotnet list package --vulnerable --include-transitive
  displayName: Dependency vulnerability scan
  continueOnError: false
```
