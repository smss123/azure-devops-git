# Azure DevOps REST API Patterns

## Base Setup

```python
import os, requests, time, json
from base64 import b64encode

ORG  = os.environ["ADO_ORG"]      # https://dev.azure.com/ORG
PAT  = os.environ["ADO_PAT"]
PROJ = os.environ["ADO_PROJECT"]

def ado_headers():
    token = b64encode(f":{PAT}".encode()).decode()
    return {
        "Authorization": f"Basic {token}",
        "Content-Type": "application/json",
    }

def ado_get(path, params=None, api="7.1"):
    url = f"{ORG}/{PROJ}/_apis/{path}?api-version={api}"
    r = requests.get(url, headers=ado_headers(), params=params)
    r.raise_for_status()
    return r.json()

def ado_post(path, body, api="7.1"):
    url = f"{ORG}/{PROJ}/_apis/{path}?api-version={api}"
    r = requests.post(url, headers=ado_headers(), json=body)
    r.raise_for_status()
    return r.json()

def ado_patch(path, body, api="7.1", content_type=None):
    headers = ado_headers()
    if content_type:
        headers["Content-Type"] = content_type
    url = f"{ORG}/{PROJ}/_apis/{path}?api-version={api}"
    r = requests.patch(url, headers=headers, json=body)
    r.raise_for_status()
    return r.json()
```

## Pagination Pattern (critical for bulk ops)

```python
def ado_get_all(path, params=None, api="7.1"):
    """Fetch all pages from a paginated ADO endpoint."""
    params = params or {}
    params["$top"] = 100
    params["$skip"] = 0
    results = []
    while True:
        data = ado_get(path, params=params, api=api)
        items = data.get("value", [data])
        results.extend(items)
        if len(items) < params["$top"]:
            break
        params["$skip"] += params["$top"]
        time.sleep(0.3)   # respect rate limits
    return results
```

## Retry with Backoff

```python
def ado_get_retry(path, params=None, max_retries=5):
    for attempt in range(max_retries):
        try:
            return ado_get(path, params)
        except requests.HTTPError as e:
            if e.response.status_code == 429:   # rate limited
                wait = int(e.response.headers.get("Retry-After", 2 ** attempt))
                time.sleep(wait)
            elif attempt == max_retries - 1:
                raise
            else:
                time.sleep(2 ** attempt)
```

## Common Endpoints

```python
# List repositories
repos = ado_get("git/repositories")["value"]

# Get repo by name
repo = ado_get(f"git/repositories/{REPO_NAME}")

# List branches
branches = ado_get(f"git/repositories/{REPO_ID}/refs", {"filter": "heads/"})["value"]

# Create branch
new_branch = ado_post(f"git/repositories/{REPO_ID}/refs", [{
    "name": "refs/heads/feature/new-feature",
    "newObjectId": BASE_COMMIT_SHA,
    "oldObjectId": "0000000000000000000000000000000000000000"
}])

# Create work item
wi = ado_post("wit/workitems/$Task", [
    {"op": "add", "path": "/fields/System.Title", "value": "My Task"},
    {"op": "add", "path": "/fields/System.Description", "value": "Details"},
    {"op": "add", "path": "/fields/System.AreaPath", "value": PROJ},
], api="7.1", content_type="application/json-patch+json")

# Update work item state
ado_patch(f"wit/workitems/{WI_ID}", [
    {"op": "replace", "path": "/fields/System.State", "value": "Active"}
], content_type="application/json-patch+json")

# Run WIQL query
results = ado_post("wit/wiql", {
    "query": f"SELECT [Id] FROM WorkItems WHERE [System.State] = 'New' AND [System.TeamProject] = '{PROJ}'"
})
```

## Batch Work Item Update

```python
def bulk_update_work_items(wi_ids: list, field: str, value: str):
    """Update a field on many work items, respecting rate limits."""
    updated = []
    errors = []
    for wi_id in wi_ids:
        try:
            ado_patch(f"wit/workitems/{wi_id}", [
                {"op": "replace", "path": f"/fields/{field}", "value": value}
            ], content_type="application/json-patch+json")
            updated.append(wi_id)
            time.sleep(0.3)
        except Exception as e:
            errors.append({"id": wi_id, "error": str(e)})
    return {"updated": updated, "errors": errors}
```
