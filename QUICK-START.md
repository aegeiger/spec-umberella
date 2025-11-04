# Quick Start: RFE Workflow with Runner Selection

**Audience:** Users creating RFE workflows on vTeam
**Prerequisites:** Familiarity with vTeam platform, RFE workflow basics

---

## Creating RFE Workflow with Runner Selection

### Option 1: Claude Code Runner (Default)

The Claude Code runner is the default, stable option that has been running RFE workflows in production.

```bash
curl -X POST https://vteam.example.com/api/projects/my-project/rfe-workflows \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Add User Authentication",
    "description": "Implement OAuth2 authentication with GitHub provider",
    "repos": [
      {
        "input": {
          "url": "https://github.com/my-org/my-repo",
          "branch": "main"
        }
      }
    ]
  }'
```

**Note:** No `runner` field specified = defaults to `claude-code`.

### Option 2: LangGraph Runner (New)

The LangGraph runner provides checkpointed execution with observable graph stages.

```bash
curl -X POST https://vteam.example.com/api/projects/my-project/rfe-workflows \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Add User Authentication",
    "description": "Implement OAuth2 authentication with GitHub provider",
    "runner": "langgraph",
    "repos": [
      {
        "input": {
          "url": "https://github.com/my-org/my-repo",
          "branch": "main"
        }
      }
    ]
  }'
```

**Note:** Explicit `runner: langgraph` selects LangGraph runner.

---

## When to Use Each Runner

### Use Claude Code Runner When:

- **Default choice**: You want the proven, stable option
- **Interactive execution**: You're comfortable with CLI-based interaction
- **Production workloads**: You need maximum reliability
- **No checkpoint needs**: You don't need pause/resume functionality

### Use LangGraph Runner When:

- **Checkpointed execution**: You want ability to resume from any stage (specify, plan, tasks)
- **Enhanced observability**: You need structured execution events for debugging
- **Future features**: You want foundation for human-in-the-loop review gates (post-MVP)
- **Experimentation**: You're validating graph-based execution model

---

## Complete Workflow Example

### Step 1: Create RFE Workflow

```bash
# Create workflow with LangGraph runner
curl -X POST https://vteam.example.com/api/projects/my-project/rfe-workflows \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Add User Authentication",
    "description": "OAuth2 with GitHub provider",
    "runner": "langgraph",
    "repos": [
      {
        "input": {
          "url": "https://github.com/my-org/backend-api",
          "branch": "main"
        }
      },
      {
        "input": {
          "url": "https://github.com/my-org/spec-kit",
          "branch": "main",
          "tags": ["spec-kit"]
        }
      }
    ]
  }'

# Response includes workflow ID
{
  "id": "rfe-workflow-123",
  "title": "Add User Authentication",
  "runner": "langgraph",
  "status": "pending"
}
```

### Step 2: Seed Repositories

```bash
# This step is the same for both runners
curl -X POST https://vteam.example.com/api/projects/my-project/rfe-workflows/rfe-workflow-123/seed

# Wait for seeding to complete
# SpecKit templates will be loaded to .specify/templates/
```

### Step 3: Create Agentic Session

```bash
curl -X POST https://vteam.example.com/api/projects/my-project/agentic-sessions \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Implement OAuth2 authentication with GitHub provider. Include user profile sync and token refresh.",
    "workflowRef": {
      "kind": "RFEWorkflow",
      "name": "rfe-workflow-123"
    },
    "interactive": true
  }'

# Response includes session ID
{
  "id": "session-456",
  "status": "running"
}
```

### Step 4: Monitor Execution

Connect via WebSocket to watch progress:

```javascript
const ws = new WebSocket('wss://vteam.example.com/api/projects/my-project/sessions/session-456/ws');

ws.onmessage = (event) => {
  const message = JSON.parse(event.data);

  switch (message.type) {
    case 'SYSTEM_MESSAGE':
      console.log(`[System] ${message.payload.message}`);
      break;

    case 'AGENT_MESSAGE':
      if (message.payload.type === 'file_written') {
        console.log(`[File] ${message.payload.file_path} written`);
      }
      break;

    case 'SESSION_STATUS':
      console.log(`[Status] ${message.payload.status}`);
      break;
  }
};
```

**Example output (LangGraph runner):**
```
[System] Starting RFE workflow execution...
[System] Executing node: specify
[File] spec.md written (12.5 KB)
[System] Checkpoint saved (specify stage complete)
[System] Executing node: plan
[File] plan.md written (18.3 KB)
[System] Checkpoint saved (plan stage complete)
[System] Executing node: tasks
[File] tasks.md written (9.7 KB)
[System] Checkpoint saved (tasks stage complete)
[Status] completed
```

### Step 5: View Outputs

```bash
# Download spec.md
curl https://vteam.example.com/api/projects/my-project/sessions/session-456/files/spec.md

# Download plan.md
curl https://vteam.example.com/api/projects/my-project/sessions/session-456/files/plan.md

# Download tasks.md
curl https://vteam.example.com/api/projects/my-project/sessions/session-456/files/tasks.md
```

**Note:** Outputs are identical regardless of runner choice (claude-code or langgraph).

---

## Session Continuation (LangGraph Only)

If a LangGraph runner session fails or is interrupted, you can resume from the last checkpoint:

```bash
curl -X POST https://vteam.example.com/api/projects/my-project/agentic-sessions \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Continue from where we left off",
    "workflowRef": {
      "kind": "RFEWorkflow",
      "name": "rfe-workflow-123"
    },
    "parent_session_id": "session-456",
    "interactive": true
  }'
```

The LangGraph runner will:
1. Load checkpoint from parent session
2. Resume from last completed node (e.g., if plan.md was written, start at tasks node)
3. Reuse spec.md and plan.md from previous session

---

## Comparing Outputs

### Side-by-Side Comparison

Create two RFE workflows with identical prompts but different runners:

```bash
# Workflow 1: Claude Code runner
curl -X POST .../rfe-workflows -d '{
  "title": "Test OAuth2",
  "runner": "claude-code",
  ...
}'

# Workflow 2: LangGraph runner
curl -X POST .../rfe-workflows -d '{
  "title": "Test OAuth2",
  "runner": "langgraph",
  ...
}'

# Run sessions for both
# Compare outputs: spec.md, plan.md, tasks.md
```

**Expected results:**
- Both runners produce spec.md with same sections (Overview, Goals, Requirements, etc.)
- Both runners produce plan.md with same structure (Architecture, Dependencies, Implementation)
- Both runners produce tasks.md with actionable task breakdown
- Content quality is equivalent (validated by manual review)

---

## Configuration Options

### Environment Variables (Operator)

```yaml
# Operator deployment manifest
env:
  - name: AMBIENT_CODE_RUNNER_IMAGE
    value: quay.io/ambient-code/claude-runner:latest
  - name: LANGGRAPH_RUNNER_IMAGE
    value: quay.io/ambient-code/langgraph-runner:latest
```

### Environment Variables (LangGraph Runner)

```yaml
# LangGraph runner pod
env:
  - name: MAX_CHECKPOINTS_PER_THREAD
    value: "10"  # Keep last 10 checkpoints
  - name: PRUNE_INTERVAL_HOURS
    value: "24"  # Prune daily
  - name: RETRY_ATTEMPTS
    value: "3"   # Retry failed nodes 3 times
  - name: TIMEOUT_SECONDS
    value: "3600"  # 1 hour timeout
  - name: TRACK_METRICS
    value: "true"  # Enable metrics tracking
```

---

## Debugging Tips

### View Checkpoint History (LangGraph Only)

```bash
# SSH into runner pod
kubectl exec -it <langgraph-runner-pod> -- /bin/bash

# Inspect checkpoint database
sqlite3 /workspace/sessions/<session-id>/.langgraph/checkpoints.db

# List all checkpoints
SELECT checkpoint_id, timestamp, step FROM checkpoints ORDER BY timestamp DESC;

# View checkpoint count
SELECT COUNT(*) FROM checkpoints;
```

### Check Runner Selection

```bash
# View which runner image was selected for a session
kubectl describe job <session-job-name> | grep Image

# Should show either:
# - quay.io/ambient-code/claude-runner:latest
# - quay.io/ambient-code/langgraph-runner:latest
```

### Compare Execution Times

```bash
# Get execution metrics for both runners
curl https://vteam.example.com/api/projects/my-project/sessions/session-456/metrics

# LangGraph runner includes per-node timing:
{
  "total_duration_ms": 45230,
  "nodes": [
    {"name": "specify", "duration_ms": 12340},
    {"name": "plan", "duration_ms": 18920},
    {"name": "tasks", "duration_ms": 13970}
  ],
  "checkpoints_saved": 3
}
```

---

## Best Practices

### 1. Start with Claude Code Runner

For production workloads, start with the default Claude Code runner:

```json
{
  "title": "Production Feature",
  "runner": "claude-code"  // or omit for default
}
```

Switch to LangGraph runner once validated in development.

### 2. Use LangGraph for Long-Running Workflows

If your RFE workflow might be interrupted (network issues, pod restarts):

```json
{
  "title": "Complex Feature",
  "runner": "langgraph"  // Checkpoint resume capability
}
```

### 3. Monitor Checkpoint Storage

For LangGraph workflows, monitor PVC usage:

```bash
# Check checkpoint database size
kubectl exec -it <pod> -- du -h /workspace/sessions/*/. langgraph/checkpoints.db

# Should be < 10 MB per session (with pruning)
```

### 4. Validate Output Equivalence

Before adopting LangGraph runner broadly, validate outputs:

```bash
# Run same prompt with both runners
# Compare spec.md, plan.md, tasks.md
diff <(curl .../session-claude/files/spec.md) \
     <(curl .../session-langgraph/files/spec.md)

# Should show minimal differences (timestamps, minor formatting)
```

---

## Common Issues

### Issue 1: Runner Field Not Recognized

**Error:** `Invalid runner value: langgraph`

**Solution:** Ensure RFEWorkflow CRD has been updated with `runner` field:

```bash
kubectl get crd rfeworkflows.vteam.ambient-code -o yaml | grep runner
```

If missing, apply updated CRD manifest.

### Issue 2: LangGraph Runner Image Not Found

**Error:** `Failed to pull image: langgraph-runner:latest`

**Solution:** Verify operator has `LANGGRAPH_RUNNER_IMAGE` environment variable:

```bash
kubectl get deployment operator -o yaml | grep LANGGRAPH_RUNNER_IMAGE
```

### Issue 3: SpecKit Templates Missing

**Error:** `SpecKit template not found: spec-template.md`

**Solution:** Ensure repositories are seeded before creating session:

```bash
# Check seeding status
curl https://vteam.example.com/api/projects/my-project/rfe-workflows/rfe-workflow-123

# Should show: "seeded": true

# If not seeded, trigger seeding
curl -X POST .../rfe-workflows/rfe-workflow-123/seed
```

### Issue 4: Checkpoint Database Locked

**Error:** `database is locked`

**Solution:** Another process is accessing checkpoints. This is rare with SQLite WAL mode. Check for:
- Stale runner pods
- Multiple sessions using same thread_id (shouldn't happen)

```bash
# List all pods for this session
kubectl get pods -l session-id=<session-id>

# Should be only one running pod
```

---

## Example Outputs

### spec.md (Both Runners)

```markdown
# Feature Specification: OAuth2 Authentication

## Overview
This feature adds OAuth2 authentication with GitHub as the identity provider...

## Goals
- Enable users to log in using GitHub accounts
- Securely store and manage access tokens
...

## Requirements
### MVP Requirements
- [MVP-1] GitHub OAuth2 integration
- [MVP-2] User profile synchronization
...
```

### plan.md (Both Runners)

```markdown
# Implementation Plan: OAuth2 Authentication

## Architecture
- Frontend: React components for login flow
- Backend: OAuth2 callback handlers in Go
...

## Dependencies
- GitHub OAuth App registration
- OAuth2 client library (golang.org/x/oauth2)
...

## Implementation
### Phase 1: OAuth2 Flow
1. Register OAuth app with GitHub
2. Implement /auth/github/login endpoint
...
```

### tasks.md (Both Runners)

```markdown
# Tasks: OAuth2 Authentication

## Backend Tasks
- [ ] Register GitHub OAuth application
- [ ] Implement /auth/github/login handler
- [ ] Implement /auth/github/callback handler
...

## Frontend Tasks
- [ ] Create Login button component
- [ ] Handle OAuth redirect flow
...

## Testing Tasks
- [ ] Unit tests for OAuth handlers
- [ ] Integration test for full login flow
...
```

---

## Resources

- **vTeam Platform Docs**: https://docs.vteam.example.com
- **RFEWorkflow CRD Reference**: See `/components/manifests/crds/rfeworkflows-crd.yaml`
- **Technical Architecture**: See `TECHNICAL-ARCHITECTURE.md` in this repo
- **Implementation Patterns**: See `IMPLEMENTATION-PATTERNS.md` in this repo

---

## Getting Help

- **Slack:** #vteam-support
- **Email:** vteam-team@example.com
- **Office Hours:** Tuesdays 2-3pm PT

---

**Last Updated:** 2025-11-04
**Maintained By:** vTeam Platform Team
