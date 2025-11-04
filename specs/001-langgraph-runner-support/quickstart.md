# Quickstart Guide: LangGraph Runner Support

**Feature**: LangGraph Runner Support
**Audience**: Developers, QA Engineers, Platform Users
**Reading Time**: 15 minutes

## Overview

This quickstart guide provides everything you need to understand, use, and troubleshoot LangGraph runner support in the vTeam platform. After reading this guide, you will:

- Understand when to choose LangGraph vs Claude Code runner
- Know how to create sessions with LangGraph runner via API and UI
- Be able to test LangGraph runner locally
- Troubleshoot common issues

---

## When to Use LangGraph Runner

### Choose LangGraph Runner When:

✅ **Structured Workflow Execution** - You need predictable, phase-based workflow orchestration (RFE: Ideate → Specify → Plan → Tasks)

✅ **Reproducible Automation** - You want deterministic execution with minimal interactive decision-making

✅ **Multi-Phase Projects** - You're working through complete RFE Workflow phases sequentially

✅ **Batch Processing** - You're running multiple similar tasks that benefit from workflow templates

### Choose Claude Code Runner When:

✅ **Interactive Development** - You need back-and-forth collaboration with the agent during implementation

✅ **Exploratory Work** - You're investigating, debugging, or refactoring with uncertain scope

✅ **Custom Tasks** - You need flexible tool usage beyond RFE Workflow structure

✅ **Continuation Support** - You want to pause and resume sessions (currently LangGraph doesn't support continuation)

### Key Differences

| Feature | Claude Code | LangGraph |
|---------|-------------|-----------|
| **Execution Model** | Interactive REPL | State machine workflow |
| **Session Continuation** | ✅ Supported | ❌ Not supported (MVP) |
| **RFE Workflow** | ✅ Via slash commands | ✅ Native phase support |
| **Custom Tools** | ✅ Full SDK tooling | ⚠️ Predefined tool set |
| **Determinism** | Variable (interactive) | High (workflow-driven) |
| **Streaming Output** | ✅ Token streaming | ✅ Token streaming |
| **Multi-Repo** | ✅ Supported | ✅ Supported |

---

## Creating Sessions with LangGraph Runner

### Via API (cURL)

**Example 1: Ideate Phase (Interactive)**

```bash
curl -X POST "http://backend:8080/api/projects/my-project/sessions" \
  -H "Authorization: Bearer ${JWT_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "runnerType": "langgraph",
    "prompt": "Create an RFE for adding OAuth2 authentication to the platform",
    "interactive": true,
    "llmSettings": {
      "model": "claude-3-7-sonnet-latest",
      "temperature": 0.8,
      "maxTokens": 4000
    },
    "repos": [
      {
        "input": {
          "url": "https://github.com/example/my-app.git",
          "branch": "main"
        },
        "output": {
          "url": "https://github.com/example/my-app.git",
          "branch": "001-oauth2-auth"
        }
      }
    ],
    "autoPushOnComplete": true
  }'
```

**Example 2: Plan Phase (Automated)**

```bash
curl -X POST "http://backend:8080/api/projects/my-project/sessions" \
  -H "Authorization: Bearer ${JWT_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "runnerType": "langgraph",
    "prompt": "Generate implementation plan based on the specification",
    "interactive": false,
    "llmSettings": {
      "model": "claude-3-7-sonnet-latest",
      "temperature": 0.3,
      "maxTokens": 4000
    },
    "repos": [
      {
        "input": {
          "url": "https://github.com/example/my-app.git",
          "branch": "001-oauth2-auth"
        },
        "output": {
          "url": "https://github.com/example/my-app.git",
          "branch": "001-oauth2-auth"
        }
      }
    ],
    "autoPushOnComplete": true,
    "environmentVariables": {
      "RFE_WORKFLOW_PHASE": "plan"
    }
  }'
```

**Example 3: Multi-Repo Session**

```bash
curl -X POST "http://backend:8080/api/projects/my-project/sessions" \
  -H "Authorization: Bearer ${JWT_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "runnerType": "langgraph",
    "prompt": "Create specification for API service updates",
    "repos": [
      {
        "input": {
          "url": "https://github.com/example/backend.git",
          "branch": "main"
        },
        "output": {
          "url": "https://github.com/example/backend.git",
          "branch": "002-api-updates"
        }
      },
      {
        "input": {
          "url": "https://github.com/example/frontend.git",
          "branch": "main"
        }
      }
    ],
    "mainRepoIndex": 0
  }'
```

### Via Frontend UI

1. **Navigate to Session Creation**
   - Go to Project → New Session

2. **Select Runner Type**
   - In the "Runner Type" dropdown, select **LangGraph**
   - (Claude Code is the default)

3. **Configure Session**
   - **Prompt**: Enter your task description
   - **Model Settings**: Choose model, temperature, max tokens
   - **Timeout**: Set session timeout (default: 600s)
   - **Interactive Mode**: Toggle if you need interactive prompts (MVP: queued but not processed)

4. **Configure Repositories**
   - Click "Add Repository"
   - Enter input repository URL and branch
   - (Optional) Configure output repository and branch
   - For multi-repo: Add additional repositories and set main repo index

5. **Advanced Options** (Optional)
   - Auto-push on complete
   - Custom environment variables
   - Resource overrides (CPU, memory)

6. **Create Session**
   - Click "Create Session"
   - Monitor execution in real-time via WebSocket stream

---

## Testing LangGraph Runner Locally

### Prerequisites

- Docker or Podman
- Kubernetes cluster (Kind for local testing)
- vTeam backend running
- Git repository with test content

### Setup Local Environment

**1. Build LangGraph Runner Image**

```bash
cd components/runners/langgraph-runner
docker build -t vteam-langgraph-runner:local .
```

**2. Load Image into Kind Cluster**

```bash
kind load docker-image vteam-langgraph-runner:local --name vteam-cluster
```

**3. Update Operator Configuration**

```bash
kubectl set env deployment/vteam-operator \
  AMBIENT_LANGGRAPH_RUNNER_IMAGE=vteam-langgraph-runner:local \
  -n vteam-system
```

### Run Test Session

**1. Create Test Session via kubectl**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: vteam.ambient-code/v1alpha1
kind: AgenticSession
metadata:
  name: test-langgraph-session
  namespace: test-project
spec:
  runnerType: langgraph
  prompt: "Create a simple RFE for adding a health check endpoint"
  llmSettings:
    model: claude-3-7-sonnet-latest
    temperature: 0.7
    maxTokens: 4000
  timeout: 600
  interactive: false
  repos:
    - input:
        url: https://github.com/example/test-repo.git
        branch: main
      output:
        url: https://github.com/example/test-repo.git
        branch: test-health-check
  autoPushOnComplete: false
EOF
```

**2. Monitor Job Creation**

```bash
# Watch for Job creation
kubectl get jobs -n test-project -w

# Check pod status
kubectl get pods -n test-project -l job-name=test-langgraph-session-job

# View logs
kubectl logs -n test-project -l job-name=test-langgraph-session-job -f
```

**3. Verify Session Status**

```bash
# Check AgenticSession CR status
kubectl get agenticsession test-langgraph-session -n test-project -o yaml

# Check artifacts in status field
kubectl get agenticsession test-langgraph-session -n test-project \
  -o jsonpath='{.status.artifacts}'
```

### Manual Runner Testing (Without Kubernetes)

**1. Set Environment Variables**

```bash
export SESSION_ID="test-session-local"
export WORKSPACE_PATH="/tmp/test-workspace"
export WEBSOCKET_URL="ws://localhost:8080/api/projects/test/sessions/test-session-local/ws"
export INPUT_REPO_URL="https://github.com/example/test-repo.git"
export INPUT_BRANCH="main"
export OUTPUT_REPO_URL="https://github.com/example/test-repo.git"
export OUTPUT_BRANCH="test-langgraph-local"
export PROMPT="Create a simple RFE for adding a health check endpoint"
export LLM_MODEL="claude-3-7-sonnet-latest"
export LLM_TEMPERATURE="0.7"
export LLM_MAX_TOKENS="4000"
export INTERACTIVE="false"
export ANTHROPIC_API_KEY="sk-ant-..."
export GITHUB_TOKEN="ghp_..."
export BOT_TOKEN="test-token"
export PROJECT_NAME="test-project"
export AGENTIC_SESSION_NAMESPACE="test-project"
```

**2. Run LangGraph Runner**

```bash
cd components/runners/langgraph-runner
python3 -m src.main
```

**3. Monitor Output**

- Watch console for system messages
- Check `/tmp/test-workspace` for cloned repositories
- Verify artifacts in `specs/{branchName}/` directory

---

## Environment Variables Reference

### Required Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `SESSION_ID` | Session identifier | `agentic-session-1699564832` |
| `WORKSPACE_PATH` | Workspace directory | `/workspace/sessions/{session_id}/workspace` |
| `WEBSOCKET_URL` | Backend WebSocket URL | `ws://backend:8080/api/projects/{project}/sessions/{session}/ws` |
| `PROMPT` | User's initial prompt | `Create an RFE for OAuth2 auth` |
| `ANTHROPIC_API_KEY` | Anthropic API key | `sk-ant-...` |
| `BOT_TOKEN` | Kubernetes service account token | (injected by operator) |

### Repository Configuration

| Variable | Description | Example |
|----------|-------------|---------|
| `INPUT_REPO_URL` | Input repository URL (legacy) | `https://github.com/example/repo.git` |
| `INPUT_BRANCH` | Input branch (legacy) | `main` |
| `OUTPUT_REPO_URL` | Output repository URL (legacy) | `https://github.com/example/repo.git` |
| `OUTPUT_BRANCH` | Output branch (legacy) | `001-feature` |
| `REPOS_JSON` | Multi-repo config (JSON array) | `[{input: {url, branch}, output: {url, branch}}]` |
| `MAIN_REPO_INDEX` | Main repository index | `0` |
| `GITHUB_TOKEN` | Git authentication token | `ghp_...` |

### LLM Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `LLM_MODEL` | Claude model identifier | `claude-3-7-sonnet-latest` |
| `LLM_TEMPERATURE` | LLM temperature | `0.7` |
| `LLM_MAX_TOKENS` | Max tokens for response | `4000` |

### Optional Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `INTERACTIVE` | Enable interactive mode | `false` |
| `TIMEOUT` | Session timeout (seconds) | `600` |
| `AUTO_PUSH_ON_COMPLETE` | Auto-push artifacts | `false` |
| `PARENT_SESSION_ID` | Parent session for continuation | (none) |
| `RFE_WORKFLOW_PHASE` | Explicit phase override | (auto-detect) |

---

## Troubleshooting

### Common Issues

#### 1. Session Fails with "Unsupported runner type"

**Symptoms**: API returns 400 Bad Request with error message

**Cause**: Invalid `runnerType` value in request

**Solution**:
```bash
# Valid values: "claude-code", "langgraph"
# Check for typos (case-sensitive, lowercase only)
curl ... -d '{"runnerType": "langgraph", ...}'  # ✅ Correct
curl ... -d '{"runnerType": "LangGraph", ...}'  # ❌ Wrong (case)
curl ... -d '{"runnerType": "langraph", ...}'   # ❌ Wrong (typo)
```

#### 2. Job Pods Stuck in ImagePullBackOff

**Symptoms**: `kubectl get pods` shows ImagePullBackOff status

**Cause**: LangGraph runner image not available in registry or Kind cluster

**Solution**:
```bash
# For local testing with Kind:
docker build -t vteam-langgraph-runner:local .
kind load docker-image vteam-langgraph-runner:local --name vteam-cluster

# For production:
# Ensure CI/CD pipeline published image to registry
# Check operator environment variable AMBIENT_LANGGRAPH_RUNNER_IMAGE
kubectl get deployment vteam-operator -n vteam-system -o yaml | grep AMBIENT_LANGGRAPH
```

#### 3. Session Fails with "Failed to authenticate with Anthropic API"

**Symptoms**: Session status shows Failed with auth error

**Cause**: Invalid or missing ANTHROPIC_API_KEY

**Solution**:
```bash
# Check if secret exists
kubectl get secret anthropic-api-key -n test-project

# Verify secret content (base64 encoded)
kubectl get secret anthropic-api-key -n test-project -o jsonpath='{.data.api-key}' | base64 -d

# Update secret if invalid
kubectl create secret generic anthropic-api-key \
  --from-literal=api-key=sk-ant-... \
  -n test-project \
  --dry-run=client -o yaml | kubectl apply -f -
```

#### 4. Artifacts Not Committed to Git

**Symptoms**: Session completes but no artifacts in repository

**Cause**: Missing GITHUB_TOKEN or insufficient permissions

**Solution**:
```bash
# Check if git credentials are configured
kubectl logs -n test-project -l job-name={session}-job | grep "git push"

# Verify GITHUB_TOKEN secret
kubectl get secret github-token -n test-project

# Check repository permissions (token needs write access)
# Test manually:
git clone https://${GITHUB_TOKEN}@github.com/example/repo.git
cd repo
git checkout -b test-branch
echo "test" > test.txt
git add test.txt
git commit -m "test"
git push origin test-branch  # Should succeed
```

#### 5. WebSocket Connection Failures

**Symptoms**: Session starts but no real-time updates in UI

**Cause**: WebSocket connection failed, backend not reachable

**Solution**:
```bash
# Check backend service is running
kubectl get svc backend-service -n vteam-system

# Test WebSocket connection manually (using wscat)
wscat -c "ws://backend:8080/api/projects/test/sessions/test-session/ws" \
  -H "Authorization: Bearer ${BOT_TOKEN}"

# Check runner logs for connection errors
kubectl logs -n test-project -l job-name={session}-job | grep -i websocket
```

#### 6. Session Times Out Before Completion

**Symptoms**: Session status shows Failed with timeout error

**Cause**: Default timeout (600s) insufficient for complex tasks

**Solution**:
```bash
# Increase timeout in session request
curl ... -d '{
  "runnerType": "langgraph",
  "timeout": 1800,  # 30 minutes
  ...
}'

# Or via kubectl
kubectl patch agenticsession {session-name} -n test-project \
  --type=merge -p '{"spec":{"timeout":1800}}'
```

#### 7. Prerequisite Validation Errors

**Symptoms**: Session fails immediately with "spec.md required for plan phase"

**Cause**: Phase requires prerequisite artifact that doesn't exist

**Solution**:
```bash
# Check what artifacts exist in workspace
kubectl exec -it -n test-project {pod-name} -- ls -la /workspace/sessions/{session}/workspace/specs/{branch}/

# Ensure correct phase sequencing:
# 1. Ideate → produces rfe.md
# 2. Specify (requires rfe.md) → produces spec.md
# 3. Plan (requires spec.md) → produces plan.md
# 4. Tasks (requires plan.md) → produces tasks.md

# Run phases in order, or manually create prerequisite files
```

### Debugging Tips

**View Detailed Logs**:
```bash
# Runner logs (most useful)
kubectl logs -n {namespace} -l job-name={session}-job --tail=100 -f

# Operator logs (for Job provisioning issues)
kubectl logs -n vteam-system deployment/vteam-operator --tail=100 -f

# Backend logs (for API validation issues)
kubectl logs -n vteam-system deployment/backend-service --tail=100 -f
```

**Inspect AgenticSession CR**:
```bash
# Full CR details
kubectl get agenticsession {session-name} -n {namespace} -o yaml

# Just status field
kubectl get agenticsession {session-name} -n {namespace} -o jsonpath='{.status}'

# Watch for status changes
kubectl get agenticsession {session-name} -n {namespace} -w
```

**Check PVC and Workspace**:
```bash
# List PVCs
kubectl get pvc -n {namespace}

# Access workspace directly (if pod is running)
kubectl exec -it -n {namespace} {pod-name} -- bash
cd /workspace/sessions/{session-id}/workspace
ls -la
```

### Getting Help

- **Documentation**: Check `/docs/langgraph-runner.md` for detailed architecture
- **Slack**: #vteam-support channel
- **GitHub Issues**: https://github.com/ambient-code/vteam/issues
- **Operator Logs**: Most issues are visible in operator logs

---

## Best Practices

### Session Configuration

✅ **DO**:
- Use `autoPushOnComplete: true` for automated workflows
- Set appropriate timeouts based on task complexity (ideate: 5min, plan: 10min)
- Use `interactive: false` for MVP (interactive not yet implemented)
- Provide clear, specific prompts for better results

❌ **DON'T**:
- Use LangGraph runner for exploratory/debugging tasks (use Claude Code)
- Attempt session continuation with LangGraph (not supported in MVP)
- Mix runner types within same project scope (consistency required)
- Omit output repository if you want artifacts committed

### Repository Management

✅ **DO**:
- Use separate feature branches for each session
- Configure output repository for automatic commits
- Use `mainRepoIndex` to select primary working directory for multi-repo
- Test git credentials before running sessions

❌ **DON'T**:
- Work directly on main branch
- Forget to configure git user identity (GIT_USER_NAME/GIT_USER_EMAIL)
- Use SSH URLs without configuring SSH keys
- Share workspace PVCs across incompatible runner types

### Monitoring and Observability

✅ **DO**:
- Monitor WebSocket stream for real-time progress
- Check CR status after session completion
- Review runner logs if session fails
- Use structured logging for debugging

❌ **DON'T**:
- Ignore timeout warnings
- Delete PVCs immediately after session (may need for debugging)
- Overlook authentication errors (fail fast, don't retry)

---

## Next Steps

- **Architecture Deep Dive**: Read `/specs/001-langgraph-runner-support/research.md`
- **API Reference**: Review `/specs/001-langgraph-runner-support/contracts/api-create-session.yaml`
- **Data Model**: Understand entities in `/specs/001-langgraph-runner-support/data-model.md`
- **Testing**: Explore testing strategy in testing documents
- **Implementation**: See tasks breakdown in `/specs/001-langgraph-runner-support/tasks.md` (Phase 2 output)

---

## Feedback

Found an issue or have a suggestion? Create an issue in the vTeam repository with the label `langgraph-runner`.
