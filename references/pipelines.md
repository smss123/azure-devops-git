# Azure Pipelines Reference

## YAML Pipeline Skeleton

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include: [main, develop, release/*]
  paths:
    exclude: ['**/*.md', 'docs/**']

pr:
  branches:
    include: [main, develop]
  drafts: false

variables:
  - group: production-secrets          # Key Vault-linked variable group
  - name: buildConfiguration
    value: Release
  - name: imageTag
    value: $(Build.BuildId)

pool:
  vmImage: ubuntu-latest

stages:
  - stage: Build
    displayName: Build & Test
    jobs:
      - job: BuildJob
        steps:
          - checkout: self
            fetchDepth: 0              # full history for SonarQube/semver

          - task: UseDotNet@2
            inputs:
              version: '8.x'

          - script: dotnet build --configuration $(buildConfiguration)
            displayName: Build

          - task: DotNetCoreCLI@2
            displayName: Test
            inputs:
              command: test
              arguments: '--collect:"XPlat Code Coverage"'

          - task: PublishCodeCoverageResults@2
            inputs:
              summaryFileLocation: '**/coverage.cobertura.xml'

          - task: PublishPipelineArtifact@1
            inputs:
              targetPath: $(Build.ArtifactStagingDirectory)
              artifact: drop

  - stage: DeployDev
    displayName: Deploy → Dev
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/develop'))
    jobs:
      - deployment: DeployDev
        environment: development
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: drop
                - script: echo "Deploy to dev"

  - stage: DeployStaging
    displayName: Deploy → Staging
    dependsOn: DeployDev
    condition: succeeded()
    jobs:
      - deployment: DeployStaging
        environment: staging           # approval gate configured in ADO portal
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo "Deploy to staging"
```

---

## Environments & Approval Gates

ADO Environments provide deployment tracking and pre-deployment gates.

```bash
# Create an environment
az devops invoke \
  --area distributedtask --resource environments \
  --route-parameters project=$ADO_PROJECT \
  --http-method POST --api-version 7.1 \
  --in-file - <<EOF
{"name": "production", "description": "Production environment"}
EOF

# List environments
az devops invoke \
  --area distributedtask --resource environments \
  --route-parameters project=$ADO_PROJECT --api-version 7.1
```

### Approval gate configuration (portal)

Navigate to: **Pipelines → Environments → production → Approvals and checks**

Recommended checks for production:
- **Approvals**: 1–2 named approvers required
- **Business hours**: restrict deploys Mon–Fri 09:00–17:00
- **Required template**: only the blessed pipeline template may deploy here
- **Invoke Azure Function**: run a pre-deploy smoke test

---

## Self-Hosted Agent Management

Use self-hosted agents for: private network access, custom tooling, GPU, or Windows licensing.

```bash
# Create an agent pool
az pipelines pool create --name "self-hosted-linux" --authorize

# Register agent on the target machine
mkdir /opt/azagent && cd /opt/azagent
curl -O https://vstsagentpackage.blob.core.windows.net/agent/3.x.x/vsts-agent-linux-x64-3.x.x.tar.gz
tar zxvf *.tar.gz
./config.sh \
  --url $ADO_ORG \
  --auth pat --token $ADO_PAT \
  --pool "self-hosted-linux" \
  --agent $(hostname) \
  --work /opt/azagent/_work \
  --runAsService
sudo ./svc.sh install && sudo ./svc.sh start

# List agents in a pool
az pipelines agent list --pool-id POOL_ID --output table

# Drain agent before maintenance
az pipelines agent update --pool-id POOL_ID --agent-id AGENT_ID --enabled false
```

### Target self-hosted pool in YAML

```yaml
pool:
  name: self-hosted-linux
  demands:
    - docker
    - node -equals 20
```

---

## Artifact Management

```yaml
# Publish
- task: PublishPipelineArtifact@1
  inputs:
    targetPath: $(Build.ArtifactStagingDirectory)
    artifact: drop

# Download in a later stage
- task: DownloadPipelineArtifact@2
  inputs:
    artifact: drop
    path: $(Pipeline.Workspace)/drop
```

```bash
# Download from a completed run
az pipelines runs artifact download \
  --run-id $RUN_ID --artifact-name drop --path ./artifacts
```

---

## Key Pipeline Patterns

### Conditions
```yaml
condition: |
  and(
    succeeded(),
    ne(variables['Build.Reason'], 'PullRequest'),
    eq(variables['Build.SourceBranch'], 'refs/heads/main')
  )
```

### Matrix Strategy
```yaml
strategy:
  matrix:
    Linux_Node20:
      os: ubuntu-latest
      nodeVersion: '20.x'
    Windows_Node20:
      os: windows-latest
      nodeVersion: '20.x'
  maxParallel: 2
```

### Templates
```yaml
# pipelines/templates/deploy-steps.yml
parameters:
  - name: environment
    type: string
  - name: serviceConnection
    type: string
steps:
  - task: AzureWebApp@1
    inputs:
      azureSubscription: ${{ parameters.serviceConnection }}
      appName: myapp-${{ parameters.environment }}

# Reference from main pipeline
- template: templates/deploy-steps.yml
  parameters:
    environment: staging
    serviceConnection: azure-staging
```

### Runtime parameters (manual trigger inputs)
```yaml
parameters:
  - name: targetEnvironment
    displayName: Deploy to
    type: string
    default: staging
    values: [dev, staging, production]
  - name: skipTests
    type: boolean
    default: false

steps:
  - script: echo "Deploying to ${{ parameters.targetEnvironment }}"
  - ${{ if not(parameters.skipTests) }}:
    - script: npm test
```

### Pass variables between jobs
```yaml
# Job A outputs a variable
- script: echo "##vso[task.setvariable variable=myVar;isOutput=true]hello"
  name: setVar

# Job B consumes it
variables:
  importedVar: $[ dependencies.JobA.outputs['setVar.myVar'] ]
```

---

## Environment Promotion Strategy

```
feature/*  → CI → dev (auto)
develop    → CI → dev (auto) → staging (auto)
main       → CI → staging (auto) → prod (manual approval)
hotfix/*   → CI → prod (expedited manual approval)
```

```yaml
stages:
  - stage: DeployProd
    dependsOn: DeployStaging
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: Production
        environment: production        # requires manual approval gate
        strategy:
          runOnce:
            deploy:
              steps: [ template: templates/deploy.yml, parameters: {env: prod} ]
```

---

## Pipeline CLI Operations

```bash
# Run with variable overrides
az pipelines run \
  --name "MyApp-CI" \
  --branch feature/TICKET-123 \
  --variables buildConfig=Debug runIntegrationTests=true

# Wait for run to complete
while true; do
  STATUS=$(az pipelines runs show --id $RUN_ID --query status -o tsv)
  RESULT=$(az pipelines runs show --id $RUN_ID --query result -o tsv)
  echo "$(date +%H:%M:%S): $STATUS / $RESULT"
  [[ "$STATUS" == "completed" ]] && break
  sleep 20
done
[ "$RESULT" = "succeeded" ] || exit 1
```

---

## Common Pitfalls

| Problem | Fix |
|---|---|
| Pipeline hangs waiting for agent | Check self-hosted pool — agent may be offline or drained |
| Secret `$(MyVar)` is empty | Variable group not authorized for this pipeline |
| Cache miss every run | Key must include lock file hash |
| Slow checkout | Add `fetchDepth: 1` for CI; use `filter=blob:none` for large repos |
| Deployment approval stuck | Check Environments → Approvals & Checks → approver list is correct |
| Matrix job not appearing | YAML indentation — `matrix:` must be under `strategy:` |
| Template not found | Path is relative to repo root, not the calling file |
| Variable not visible in next job | Add `isOutput: true` to `setvariable`, reference via `dependencies.JOB.outputs['step.VAR']` |
