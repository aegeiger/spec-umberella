# Quick Start: LangGraph Workflow Development

**Audience:** Developers building LangGraph workflows on vTeam
**Prerequisites:** Familiarity with LangGraph basics, vTeam platform

---

## Creating Your First LangGraph Workflow

### 1. Define Your Graph

Create a `graph.py` file in your repository:

```python
# graph.py - Simple approval workflow

from langgraph.graph import StateGraph
from typing import TypedDict, Annotated
from langchain_core.messages import HumanMessage, AIMessage

# Define state schema
class WorkflowState(TypedDict):
    messages: list
    approved: bool


# Define nodes
async def analyze_request(state: WorkflowState) -> WorkflowState:
    """Analyze the request and prepare recommendation."""
    request = state["messages"][-1].content

    # Your analysis logic here
    analysis = f"Analyzed request: {request}"

    return {
        **state,
        "messages": state["messages"] + [AIMessage(content=analysis)]
    }


async def request_approval(state: WorkflowState) -> WorkflowState:
    """Wait for human approval (interrupt point)."""
    # This is a human-in-the-loop node
    # Graph will pause here and wait for user input
    return state


async def execute_action(state: WorkflowState) -> WorkflowState:
    """Execute the approved action."""
    if state.get("approved"):
        result = "Action executed successfully"
    else:
        result = "Action cancelled by user"

    return {
        **state,
        "messages": state["messages"] + [AIMessage(content=result)]
    }


# Build graph
graph = StateGraph(WorkflowState)

# Add nodes
graph.add_node("analyze", analyze_request)
graph.add_node("approval", request_approval)
graph.add_node("execute", execute_action)

# Define edges
graph.set_entry_point("analyze")
graph.add_edge("analyze", "approval")
graph.add_edge("approval", "execute")
graph.set_finish_point("execute")

# Compile with interrupt
graph = graph.compile(
    interrupt_before=["approval"]  # Pause before approval node
)
```

### 2. Add Configuration (Optional)

Create `.langgraph-config.yaml` to customize behavior:

```yaml
# .langgraph-config.yaml

checkpointer:
  type: sqlite
  max_checkpoints_per_thread: 5

execution:
  recursion_limit: 50
  timeout_seconds: 3600

human_in_the_loop:
  enabled: true
  input_timeout_seconds: 1800
```

### 3. Create Workflow via API

```bash
curl -X POST https://vteam.example.com/api/projects/my-project/langgraph-workflows \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Approval Workflow",
    "description": "Request approval before executing actions",
    "graphDefinition": "graph.py",
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

### 4. Create Session and Execute

```bash
curl -X POST https://vteam.example.com/api/projects/my-project/agentic-sessions \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Deploy new feature to production",
    "workflowType": "langgraph",
    "workflowRef": {
      "kind": "LangGraphWorkflow",
      "name": "approval-workflow"
    },
    "interactive": true
  }'
```

### 5. Monitor Execution

Watch the session via WebSocket or UI:

```javascript
// Frontend WebSocket connection
const ws = new WebSocket('wss://vteam.example.com/api/projects/my-project/sessions/SESSION_ID/ws');

ws.onmessage = (event) => {
  const message = JSON.parse(event.data);

  if (message.type === 'WAITING_FOR_INPUT') {
    // Graph is waiting at approval node
    const userApproval = confirm("Approve this action?");

    ws.send(JSON.stringify({
      type: 'user_message',
      payload: {
        content: userApproval ? "approved" : "rejected"
      }
    }));
  }
};
```

---

## Common Patterns

### Pattern 1: Conditional Branching

```python
def should_continue(state: WorkflowState) -> str:
    """Route to different nodes based on state."""
    if state.get("error"):
        return "handle_error"
    elif state.get("needs_approval"):
        return "request_approval"
    else:
        return "complete"

graph.add_conditional_edges(
    "analyze",
    should_continue,
    {
        "handle_error": "error_handler",
        "request_approval": "approval",
        "complete": END
    }
)
```

### Pattern 2: Parallel Execution

```python
# Execute multiple agents in parallel
graph.add_node("agent_1", agent_1_node)
graph.add_node("agent_2", agent_2_node)
graph.add_node("agent_3", agent_3_node)
graph.add_node("aggregate", aggregate_results)

# All agents start from same point
graph.add_edge("start", "agent_1")
graph.add_edge("start", "agent_2")
graph.add_edge("start", "agent_3")

# All agents feed into aggregator
graph.add_edge("agent_1", "aggregate")
graph.add_edge("agent_2", "aggregate")
graph.add_edge("agent_3", "aggregate")
```

### Pattern 3: Retry with Backoff

```python
from implementation_patterns import with_retry

@with_retry(max_attempts=3, backoff_base=2.0)
async def api_call_node(state: WorkflowState) -> WorkflowState:
    """Node with automatic retry on failure."""
    response = await external_api.call(state["request"])
    return {**state, "response": response}
```

### Pattern 4: Session Continuation

Resume a previous session:

```bash
curl -X POST https://vteam.example.com/api/projects/my-project/agentic-sessions \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Continue with user input",
    "workflowType": "langgraph",
    "parent_session_id": "previous-session-id",
    "interactive": true
  }'
```

The graph will automatically resume from the last checkpoint.

---

## Debugging Tips

### View Checkpoint History

```python
# In your workspace, inspect checkpoints
import sqlite3

conn = sqlite3.connect('.langgraph/checkpoints.db')
cursor = conn.cursor()

# List all checkpoints for a thread
cursor.execute("""
    SELECT checkpoint_id, timestamp, step
    FROM checkpoints
    WHERE thread_id = ?
    ORDER BY timestamp DESC
""", ("your-thread-id",))

for row in cursor.fetchall():
    print(f"Checkpoint {row[0]} at {row[1]}: step {row[2]}")
```

### Enable Verbose Logging

```yaml
# .langgraph-config.yaml
observability:
  track_metrics: true
  log_node_outputs: true  # See full node outputs
```

### Test Graph Locally

```python
# test_graph.py - Test without vTeam platform

from graph import graph

# Run graph with test input
result = graph.invoke(
    {"messages": [HumanMessage(content="test input")]},
    config={"configurable": {"thread_id": "test"}}
)

print(result)
```

---

## Best Practices

### 1. Keep Nodes Small and Focused

```python
# GOOD: Single responsibility
async def validate_input(state):
    # Just validation
    return state

async def process_data(state):
    # Just processing
    return state

# BAD: Doing too much
async def validate_and_process_and_send(state):
    # Too many responsibilities
    return state
```

### 2. Use Type Hints

```python
# GOOD: Clear state schema
class WorkflowState(TypedDict):
    request: str
    analysis: dict
    approved: bool

# BAD: Untyped state
state = {}
```

### 3. Handle Errors Gracefully

```python
async def robust_node(state: WorkflowState) -> WorkflowState:
    try:
        result = await risky_operation()
        return {**state, "result": result}
    except Exception as e:
        return {**state, "error": str(e), "retry": True}
```

### 4. Document Interrupt Points

```python
async def approval_node(state: WorkflowState) -> WorkflowState:
    """
    INTERRUPT POINT: Waits for human approval.

    Expected input: "approved" or "rejected"
    """
    # Graph pauses here if interrupt_before=["approval"]
    return state
```

---

## Common Issues

### Issue 1: Graph Definition Not Found

**Error:** `Graph definition not found at /workspace/graph.py`

**Solution:** Ensure `graph.py` is in the repository root and committed.

### Issue 2: Import Error

**Error:** `Module 'xyz' not allowed`

**Solution:** Only whitelisted modules are allowed. Request new modules via platform team or use existing alternatives.

### Issue 3: Checkpoint Database Locked

**Error:** `database is locked`

**Solution:** Another process is accessing checkpoints. Wait and retry, or check for stale processes.

### Issue 4: Timeout Waiting for Input

**Error:** `Input timeout after 3600s`

**Solution:** Adjust timeout in config or ensure user responds within timeout window:

```yaml
human_in_the_loop:
  input_timeout_seconds: 7200  # 2 hours
```

---

## Example Workflows

### Example 1: Code Review Workflow

```python
# Code review with auto-approval for small changes

class ReviewState(TypedDict):
    files_changed: list
    diff_size: int
    approved: bool

async def analyze_changes(state):
    # Calculate change size
    return {**state, "diff_size": calculate_size(state["files_changed"])}

async def auto_approve_small_changes(state):
    if state["diff_size"] < 100:
        return {**state, "approved": True}
    return state

async def request_human_review(state):
    # Interrupt for human review
    return state

# Build graph with conditional approval
graph = StateGraph(ReviewState)
graph.add_node("analyze", analyze_changes)
graph.add_node("auto_approve", auto_approve_small_changes)
graph.add_node("human_review", request_human_review)

graph.set_entry_point("analyze")
graph.add_edge("analyze", "auto_approve")

def needs_human(state):
    return "human_review" if not state.get("approved") else END

graph.add_conditional_edges("auto_approve", needs_human)
graph = graph.compile(interrupt_before=["human_review"])
```

### Example 2: Multi-Agent Research

```python
# Parallel research with synthesis

class ResearchState(TypedDict):
    query: str
    findings: list
    synthesis: str

async def search_academic(state):
    # Research academic sources
    return state

async def search_industry(state):
    # Research industry sources
    return state

async def search_news(state):
    # Research news sources
    return state

async def synthesize(state):
    # Combine all findings
    return state

# Parallel search, then synthesize
graph = StateGraph(ResearchState)
graph.add_node("academic", search_academic)
graph.add_node("industry", search_industry)
graph.add_node("news", search_news)
graph.add_node("synthesize", synthesize)

graph.set_entry_point("academic")
graph.set_entry_point("industry")
graph.set_entry_point("news")

graph.add_edge("academic", "synthesize")
graph.add_edge("industry", "synthesize")
graph.add_edge("news", "synthesize")

graph = graph.compile()
```

---

## Resources

- **LangGraph Documentation**: https://langchain-ai.github.io/langgraph/
- **vTeam Platform Docs**: https://docs.vteam.example.com
- **Implementation Patterns**: See `IMPLEMENTATION-PATTERNS.md` in this repo
- **Technical Architecture**: See `TECHNICAL-ARCHITECTURE.md` in this repo

---

## Getting Help

- **Slack:** #vteam-support
- **Email:** vteam-team@example.com
- **Office Hours:** Tuesdays 2-3pm PT

---

**Last Updated:** 2025-11-04
**Maintained By:** vTeam Platform Team
