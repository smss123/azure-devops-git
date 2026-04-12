# Work Items & Boards Reference

## Create Work Items

```bash
# Create a User Story
az boards work-item create \
  --type "User Story" \
  --title "As a user, I can reset my password" \
  --description "Acceptance criteria: ..." \
  --area "$ADO_PROJECT\\MyTeam" \
  --iteration "$ADO_PROJECT\\Sprint 5"

# Create a Task linked to a Story
az boards work-item create \
  --type Task \
  --title "Implement password reset endpoint" \
  --project $ADO_PROJECT

# Link work items
az boards work-item relation add \
  --id TASK_ID \
  --relation-type "parent" \
  --target-id STORY_ID
```

## Query Work Items

```bash
# All active items assigned to me
az boards query \
  --wiql "SELECT [Id],[Title],[State],[AssignedTo] FROM WorkItems WHERE [AssignedTo] = @Me AND [State] <> 'Closed'"

# Sprint backlog
az boards query \
  --wiql "SELECT [Id],[Title],[State],[StoryPoints] FROM WorkItems WHERE [System.IterationPath] = @CurrentIteration AND [System.TeamProject] = '$ADO_PROJECT' ORDER BY [System.StackRank]"

# Save query as table
az boards query \
  --wiql "SELECT [Id],[Title],[State] FROM WorkItems WHERE [System.TeamProject] = '$ADO_PROJECT'" \
  --output table
```

## Update Work Items

```bash
# Move to Active
az boards work-item update --id 1234 --state Active

# Assign to someone
az boards work-item update --id 1234 --assigned-to "user@company.com"

# Set story points
az boards work-item update --id 1234 --fields "Microsoft.VSTS.Scheduling.StoryPoints=5"

# Add a comment
az boards work-item update --id 1234 --discussion "Reviewed and approved for Sprint 5"
```

## Bulk Import from CSV

```python
# bulk_import.py — import work items from CSV
import csv, subprocess, json

def create_wi(wi_type, title, description, area, iteration):
    result = subprocess.run([
        "az", "boards", "work-item", "create",
        "--type", wi_type,
        "--title", title,
        "--description", description,
        "--area", area,
        "--iteration", iteration,
        "-o", "json"
    ], capture_output=True, text=True)
    return json.loads(result.stdout)["id"]

with open("work_items.csv") as f:
    for row in csv.DictReader(f):
        wi_id = create_wi(
            row["Type"], row["Title"], row["Description"],
            row["Area"], row["Iteration"]
        )
        print(f"Created #{wi_id}: {row['Title']}")
```

## Sprint Management

```bash
# List iterations (sprints)
az boards iteration project list --depth 2

# Create sprint
az boards iteration project create \
  --name "Sprint 6" \
  --start-date "2025-04-01" \
  --finish-date "2025-04-14"

# Add team to iteration
az boards iteration team add \
  --team "MyTeam" \
  --id ITERATION_ID
```
