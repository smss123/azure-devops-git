# Secrets & Environment Config Reference

ADO has multiple layers for secrets. Using the wrong one is the most common
security mistake. This reference covers all layers and when to use each.

---

## Secret Storage Hierarchy

```
Most secure ──────────────────────────────────► Least secure

Azure Key Vault → Variable Group (KV-linked) → Variable Group (plain)
→ Pipeline YAML variables → Environment variables → Hardcoded (NEVER)
```

---

## Layer 1 — Azure Key Vault (recommended for production secrets)

```bash
# Link a Key Vault to an ADO Variable Group
az devops invoke \
  --area distributedtask \
  --resource variablegroups \
  --route-parameters project=$ADO_PROJECT \
  --http-method POST \
  --api-version 7.1 \
  --in-file kv_group.json

# kv_group.json
{
  "name": "production-secrets",
  "type": "AzureKeyVault",
  "keyVaultName": "my-keyvault",
  "serviceEndpointId": "SERVICE_CONNECTION_ID",
  "variables": {
    "DatabasePassword":   { "enabled": true },
    "ApiKey":             { "enabled": true },
    "JwtSecret":          { "enabled": true }
  }
}
```

### Use in pipeline YAML

```yaml
variables:
  - group: production-secrets   # pulls from Key Vault at runtime

steps:
  - script: echo "Connecting to DB"
    env:
      DB_PASSWORD: $(DatabasePassword)   # always map via env:, never inline
```

---

## Layer 2 — Variable Groups (plain, for non-production)

```bash
# Create a variable group
az pipelines variable-group create \
  --name "staging-config" \
  --variables \
    AppUrl="https://staging.myapp.com" \
    LogLevel="debug" \
  --authorize true

# Add a SECRET variable (value is masked in logs)
az pipelines variable-group variable create \
  --group-id GROUP_ID \
  --name "StagingApiKey" \
  --value "secret-value-here" \
  --secret true

# List all variable groups
az pipelines variable-group list --output table

# Update a secret
az pipelines variable-group variable update \
  --group-id GROUP_ID \
  --name "StagingApiKey" \
  --value "new-secret-value" \
  --secret true
```

---

## Layer 3 — Pipeline Variables (per-pipeline, per-run)

```yaml
# In pipeline YAML — for non-secret config
variables:
  buildConfiguration: Release
  dockerRegistry: myacr.azurecr.io
  imageTag: $(Build.BuildId)

# NEVER put actual secret values here — use variable groups instead
# The following is WRONG:
variables:
  apiKey: "abc123"   # ← visible to anyone who can read the YAML
```

```bash
# Override at runtime (for feature branch deploys)
az pipelines run \
  --name "MyApp-CD" \
  --variables targetEnv=staging apiVersion=v2
```

---

## Layer 4 — Service Connections (for external system auth)

Service connections are the correct way to authenticate to Azure, Docker, npm, etc.
Never use raw credentials in pipeline steps.

```bash
# Create Azure Resource Manager service connection
az devops service-endpoint azurerm create \
  --azure-rm-service-principal-id $SP_ID \
  --azure-rm-subscription-id $SUB_ID \
  --azure-rm-subscription-name "My Subscription" \
  --azure-rm-tenant-id $TENANT_ID \
  --name "azure-production" \
  --project $ADO_PROJECT

# List service connections
az devops service-endpoint list --output table

# Use in pipeline
- task: AzureWebApp@1
  inputs:
    azureSubscription: "azure-production"   # ← service connection name
    appName: my-app
```

---

## Secret Scanning — Prevent Secrets in Commits

```bash
# Install pre-commit hook to block secret commits
pip install detect-secrets
detect-secrets scan > .secrets.baseline
git add .secrets.baseline

# .pre-commit-config.yaml
repos:
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
```

### ADO Pipeline — secret scanning gate

```yaml
- stage: SecretScan
  jobs:
    - job: ScanSecrets
      steps:
        - script: |
            pip install detect-secrets
            detect-secrets scan --baseline .secrets.baseline
            detect-secrets audit .secrets.baseline
          displayName: Scan for committed secrets
          failOnStderr: false
```

---

## Secret Rotation Pattern (automated)

```python
# rotate_secrets.py — rotate PATs and update variable groups
import subprocess, json, datetime

def rotate_pat_in_variable_group(group_id: str, var_name: str, new_value: str):
    """Update a secret variable with a rotated value."""
    result = subprocess.run([
        "az", "pipelines", "variable-group", "variable", "update",
        "--group-id", group_id,
        "--name", var_name,
        "--value", new_value,
        "--secret", "true",
        "-o", "json"
    ], capture_output=True, text=True)

    if result.returncode != 0:
        raise RuntimeError(f"Rotation failed: {result.stderr}")

    print(f"[ROTATED] {var_name} in group {group_id} at {datetime.datetime.utcnow().isoformat()}")

def audit_secret_expiry(group_id: str) -> list:
    """List variables that haven't been rotated in >90 days."""
    # Note: ADO doesn't expose rotation dates natively.
    # Track in a separate audit log file.
    audit_file = f"/tmp/secret_audit_{group_id}.json"
    if not os.path.exists(audit_file):
        return []

    with open(audit_file) as f:
        audit = json.load(f)

    now = datetime.datetime.utcnow()
    stale = []
    for var, last_rotated in audit.items():
        dt = datetime.datetime.fromisoformat(last_rotated)
        if (now - dt).days > 90:
            stale.append({"variable": var, "days_since_rotation": (now - dt).days})

    return stale
```

---

## Passing Secrets to Sub-Agents (safely)

```python
# WRONG — secret appears in process list, shell history, logs
subprocess.run(["python", "agent.py", "--pat", ADO_PAT])

# WRONG — secret in environment might be inherited by child processes unexpectedly
os.environ["ADO_PAT"] = ADO_PAT

# CORRECT — pass via controlled env dict, not global environment
subprocess.run(
    ["python", "sub_agent.py"],
    env={
        **os.environ,           # pass everything the agent needs
        "ADO_PAT": ADO_PAT,     # inject the secret explicitly
        "TICKET_ID": ticket_id,
        "WT_PATH": wt_path,
    }
)

# CORRECT — write to a temp file, pass path only (for very long secrets)
import tempfile, stat
with tempfile.NamedTemporaryFile(mode='w', suffix='.token', delete=False) as f:
    f.write(ADO_PAT)
    os.chmod(f.name, stat.S_IRUSR)  # owner read-only
    token_file = f.name
subprocess.run(["python", "agent.py", "--token-file", token_file])
# Clean up after agent exits
os.unlink(token_file)
```

---

## Secrets Checklist

Before any pipeline or agent run:

- [ ] No secrets in YAML files (use variable groups)
- [ ] No secrets in commit history (`git log -S "secret-value"` returns nothing)
- [ ] PAT scope is minimum required (not full access)
- [ ] PAT expiry is set (max 90 days for automation accounts)
- [ ] Service connections used for cloud resource auth (not raw credentials)
- [ ] Variable groups marked as `secret: true` for sensitive values
- [ ] Secret scanning pipeline gate is active
- [ ] Rotation schedule documented for each secret
- [ ] Sub-agents receive secrets via env dict, not args or global env
