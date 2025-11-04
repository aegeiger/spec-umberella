# Technical Architecture: LangGraph Workflow Integration

**Author:** Stella (Staff Engineer)
**Date:** 2025-11-04
**Status:** Technical Guidance for RFE

## Executive Summary

This document provides technical architecture guidance for adding LangGraph-based workflow support to the Ambient Agentic Runner (vTeam) platform. The analysis focuses on maintaining architectural integrity while introducing a new execution engine alongside the existing Claude Code + SpecKit workflow.

**Key Finding:** The existing runner-shell abstraction is well-designed for multi-runner support and requires minimal modifications. The primary challenge is workflow lifecycle management and state persistence patterns that differ between CLI-based and graph-based execution.

---

## 1. Integration Pattern: Runner-Shell Abstraction

### Current Architecture Strengths

The runner-shell framework (`/workspace/sessions/agentic-session-1762286246/workspace/vTeam/components/runners/runner-shell/`) provides a solid foundation:

```python
# Current pattern from wrapper.py (lines 25-42)
class ClaudeCodeAdapter:
    async def initialize(self, context: RunnerContext):
        """Initialize with workspace and environment"""

    async def run(self):
        """Main execution loop - returns result dict"""

    async def handle_message(self, message: dict):
        """Process incoming WebSocket messages"""
```

**Protocol abstraction** (`protocol.py`):
- Standardized message types (SYSTEM_MESSAGE, AGENT_MESSAGE, USER_MESSAGE, etc.)
- Session status enum (QUEUED, RUNNING, SUCCEEDED, FAILED)
- WebSocket transport layer completely decoupled from runner logic

### Recommended Pattern: LangGraph Adapter

Create a parallel adapter following the same pattern:

```python
# NEW FILE: /components/runners/langgraph-runner/adapter.py

class LangGraphAdapter:
    """Adapter for LangGraph graph execution engine"""

    def __init__(self):
        self.context = None
        self.shell = None
        self.graph = None
        self.checkpointer = None  # LangGraph's StateGraph persistence

    async def initialize(self, context: RunnerContext):
        """
        Initialize LangGraph runtime:
        - Load graph definition from workspace or config
        - Set up checkpointer (SQLite/Postgres for state)
        - Configure human-in-the-loop callbacks
        """
        self.context = context
        graph_config = self._load_graph_config()
        self.graph = self._build_graph(graph_config)
        self.checkpointer = self._setup_checkpointer()

    async def run(self):
        """
        Execute graph with streaming:
        - If continuing: load checkpoint from parent session
        - Stream node execution via WebSocket
        - Handle interrupts for human-in-the-loop
        - Persist checkpoints to PVC
        """
        thread_id = self.context.session_id

        # Resume from parent if continuing
        parent_session = self.context.get_env('PARENT_SESSION_ID', '')
        if parent_session:
            thread_id = await self._get_parent_thread_id(parent_session)

        # Execute with checkpoint persistence
        async for event in self.graph.astream_events(
            input_data,
            config={"configurable": {"thread_id": thread_id}},
            version="v1"
        ):
            await self._handle_graph_event(event)

    async def handle_message(self, message: dict):
        """
        Handle backend messages:
        - interrupt: pause graph execution
        - user_message: resume with user input (human-in-the-loop)
        - end_session: gracefully terminate
        """
```

**Key Design Decisions:**

1. **No modifications to runner-shell core** - protocol.py, transport_ws.py, shell.py remain unchanged
2. **Adapter implements same interface** - `initialize()`, `run()`, `handle_message()`
3. **State persistence uses PVC** - LangGraph checkpointer writes to `/workspace/sessions/{session-id}/.langgraph/`
4. **WebSocket protocol unchanged** - same message types, just different content

### Technical Risk: Dependency Conflicts

**Challenge:** LangGraph requires LangChain dependencies that may conflict with Claude SDK.

**Mitigation Strategy:**
```dockerfile
# Option 1: Separate container images (RECOMMENDED)
FROM python:3.11 AS langgraph-runner
RUN pip install langgraph langchain-core langchain-anthropic

# Option 2: Conditional imports with virtual environments
# Use venv or pipx for isolated dependencies
```

**Recommendation:** Build separate container images for each runner type. The operator can select the image based on workflow type (see Section 4).

---

## 2. Workflow Extensibility: Registration & Discovery

### Current State: Hardcoded RFE Workflow

The current implementation has a 1:1 relationship between workflow type (RFE) and runner (Claude Code):
- Backend has RFEWorkflow CRD (`/components/manifests/crds/rfeworkflows-crd.yaml`)
- Operator hardcodes runner image in Job spec (`/components/operator/internal/handlers/sessions.go:398`)
- No abstraction for multiple workflow types

### Recommended Pattern: Workflow Type Registry

**Option A: CRD-per-workflow-type (RECOMMENDED for MVP)**

Create a new `LangGraphWorkflow` CRD parallel to `RFEWorkflow`:

```yaml
# NEW FILE: /components/manifests/crds/langgraphworkflows-crd.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: langgraphworkflows.vteam.ambient-code
spec:
  group: vteam.ambient-code
  names:
    kind: LangGraphWorkflow
    plural: langgraphworkflows
    shortNames: [lgw]
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required: [graphDefinition]
            properties:
              graphDefinition:
                type: string
                description: "Path to graph definition file in workspace"
              checkpointerType:
                type: string
                enum: [sqlite, postgres, memory]
                default: sqlite
              humanInTheLoop:
                type: boolean
                description: "Enable human-in-the-loop interrupts"
              repos:
                type: array
                description: "Repositories (same as AgenticSession.repos)"
```

**Backend handler structure:**
```go
// NEW FILE: /components/backend/handlers/langgraph.go

func CreateProjectLangGraphWorkflow(c *gin.Context) {
    // Similar to CreateProjectRFEWorkflow
    // Creates LangGraphWorkflow CR
    // Validates graph definition exists
}

func ListProjectLangGraphWorkflows(c *gin.Context) {
    // List LangGraphWorkflow CRs in project namespace
}
```

**Pros:**
- Clear separation of workflow concerns
- Type-safe validation per workflow type
- Easy to version independently
- Familiar pattern (mirrors RFEWorkflow)

**Cons:**
- Code duplication (each workflow needs handlers, CRDs, etc.)
- Operator needs to watch multiple CRD types

---

**Option B: Unified WorkflowTemplate CRD (FUTURE)**

Create a generic workflow abstraction:

```yaml
# FUTURE: /components/manifests/crds/workflowtemplates-crd.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: workflowtemplates.vteam.ambient-code
spec:
  group: vteam.ambient-code
  names:
    kind: WorkflowTemplate
    plural: workflowtemplates
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
        properties:
          spec:
            type: object
            required: [type, runnerImage]
            properties:
              type:
                type: string
                enum: [rfe, langgraph, custom]
              runnerImage:
                type: string
                description: "Container image for this workflow type"
              configSchema:
                type: object
                description: "JSON Schema for workflow-specific configuration"
                x-kubernetes-preserve-unknown-fields: true
```

**Pros:**
- Single operator watch loop
- Extensible to future workflow types without new CRDs
- Configuration-driven approach

**Cons:**
- Loss of type safety (schema validation moves to runtime)
- More complex operator logic
- Harder to version workflow-specific features

**Recommendation:** Start with **Option A (CRD-per-workflow)** for MVP, refactor to Option B if we need to support 5+ workflow types.

---

## 3. Runner Lifecycle: CLI vs Graph Execution

### Key Differences

| Aspect | Claude Code (CLI) | LangGraph (Graph) |
|--------|-------------------|-------------------|
| **Execution Model** | Interactive CLI process | Declarative graph traversal |
| **State Persistence** | SDK manages `.claude` dir | Checkpointer (DB or file) |
| **Resumption** | SDK's built-in `resume` option | Thread ID + checkpoint lookup |
| **Human-in-the-loop** | Interactive mode (user_message) | Graph interrupt nodes |
| **Streaming** | Line-by-line output | Node execution events |

### State Management Patterns

**Claude Code Pattern (current):**
```python
# From wrapper.py:219-228
if is_continuation and parent_session_id:
    sdk_resume_id = await self._get_sdk_session_id(parent_session_id)
    if sdk_resume_id:
        options.resume = sdk_resume_id
        options.fork_session = False
```

**LangGraph Pattern (recommended):**
```python
class LangGraphAdapter:
    async def _resume_from_checkpoint(self, parent_session_id: str):
        """
        Resume graph execution from parent's checkpoint:
        1. Read parent's thread_id from CR annotation
        2. Load checkpoint from shared PVC path
        3. Resume graph execution from last node
        """
        parent_thread_id = await self._get_thread_id_from_annotation(parent_session_id)

        # Checkpointer reads from /workspace/sessions/{parent}/..langgraph/
        checkpoint = await self.checkpointer.aget(
            config={"configurable": {"thread_id": parent_thread_id}}
        )

        if checkpoint:
            return parent_thread_id  # Use same thread for continuation
        else:
            raise RuntimeError(f"No checkpoint found for {parent_session_id}")
```

**Checkpoint Storage Location:**
```bash
# PVC structure for LangGraph sessions
/workspace/sessions/
  {session-id}/
    workspace/          # Git repos
    .langgraph/         # Checkpoints (SQLite or JSON)
      checkpoints.db
      thread_{id}.json
    .claude/            # Claude SDK state (RFE workflows)
```

### Human-in-the-Loop Implementation

**LangGraph interrupts** are first-class:
```python
from langgraph.graph import StateGraph

graph = StateGraph(AgentState)
graph.add_node("agent", agent_node)
graph.add_node("human", human_node)  # Interrupt here

# Configure interrupt before human node
graph.add_edge("agent", "human")
graph.add_conditional_edges("human", should_continue)

# Execution automatically pauses at interrupt
```

**Integration with runner-shell:**
```python
async def _handle_graph_event(self, event):
    if event["event"] == "on_interrupt":
        # Send WAITING_FOR_INPUT message
        await self.shell._send_message(
            MessageType.WAITING_FOR_INPUT,
            {"node": event["name"], "state": event["data"]}
        )
        # Wait for user_message from handle_message()
        await self._wait_for_user_input()
```

---

## 4. API & CRD Design

### AgenticSession Extension

**Current AgenticSession CRD** (`agenticsessions-crd.yaml`) is workflow-agnostic. This is good design.

**Recommended additions:**
```yaml
# MODIFY: /components/manifests/crds/agenticsessions-crd.yaml
spec:
  properties:
    workflowType:
      type: string
      enum: [rfe, langgraph]  # Extensible
      description: "Workflow type - determines runner image"
    workflowRef:
      type: object
      description: "Reference to workflow definition (RFEWorkflow or LangGraphWorkflow)"
      properties:
        kind:
          type: string
          enum: [RFEWorkflow, LangGraphWorkflow]
        name:
          type: string
    runnerConfig:
      type: object
      description: "Runner-specific configuration"
      x-kubernetes-preserve-unknown-fields: true
```

**Backend session creation:**
```go
// MODIFY: /components/backend/handlers/sessions.go

type CreateAgenticSessionRequest struct {
    // ... existing fields
    WorkflowType string                 `json:"workflowType,omitempty"`  // "rfe" or "langgraph"
    WorkflowRef  *WorkflowReference     `json:"workflowRef,omitempty"`
    RunnerConfig map[string]interface{} `json:"runnerConfig,omitempty"`
}

type WorkflowReference struct {
    Kind string `json:"kind"` // "RFEWorkflow" or "LangGraphWorkflow"
    Name string `json:"name"` // CR name in same namespace
}
```

### Operator Job Creation Logic

**Current pattern** (from `sessions.go:398`):
```go
Image: appConfig.AmbientCodeRunnerImage,  // Hardcoded
```

**Recommended pattern:**
```go
// MODIFY: /components/operator/internal/handlers/sessions.go

func getRunnerImageForWorkflow(spec map[string]interface{}) string {
    workflowType, _, _ := unstructured.NestedString(spec, "workflowType")

    switch workflowType {
    case "langgraph":
        return appConfig.LangGraphRunnerImage
    case "rfe", "":  // Default to RFE for backward compatibility
        return appConfig.AmbientCodeRunnerImage
    default:
        log.Printf("Unknown workflow type %s, defaulting to RFE runner", workflowType)
        return appConfig.AmbientCodeRunnerImage
    }
}

// In job creation (line 398):
Image: getRunnerImageForWorkflow(spec),
```

**Configuration:**
```yaml
# MODIFY: /components/operator/internal/config/config.go
type Config struct {
    // ... existing fields
    AmbientCodeRunnerImage string  // Existing: Claude Code runner
    LangGraphRunnerImage   string  // NEW: LangGraph runner
}

// Environment variables
AMBIENT_CODE_RUNNER_IMAGE=quay.io/ambient-code/claude-runner:latest
LANGGRAPH_RUNNER_IMAGE=quay.io/ambient-code/langgraph-runner:latest
```

---

## 5. Technical Risks & Mitigation

### Risk 1: Dependency Hell

**Problem:** LangGraph requires LangChain, which has 50+ dependencies. Potential conflicts with Claude SDK.

**Mitigation:**
- **Separate container images** (one per runner type)
- Use multi-stage Docker builds to minimize image size
- Pin dependency versions with `requirements.lock`

**Testing strategy:**
```bash
# CI pipeline: test both runners in isolation
docker build -t claude-runner -f Dockerfile.claude .
docker build -t langgraph-runner -f Dockerfile.langgraph .

# Integration test: verify no dependency conflicts
pytest tests/test_claude_runner.py
pytest tests/test_langgraph_runner.py
```

### Risk 2: Checkpoint Persistence

**Problem:** LangGraph checkpoints can be large (graph state + message history). PVC I/O performance critical.

**Mitigation:**
- Use SQLite for checkpointer (better than JSON for large state)
- Implement checkpoint pruning (delete old checkpoints after N days)
- Consider async checkpointer implementation for better performance

**Implementation example:**
```python
from langgraph.checkpoint.sqlite import AsyncSqliteSaver

async def _setup_checkpointer(self):
    checkpoint_path = Path(self.context.workspace_path).parent / ".langgraph" / "checkpoints.db"
    checkpoint_path.parent.mkdir(parents=True, exist_ok=True)

    # Configure with WAL mode for better concurrency
    return await AsyncSqliteSaver.from_conn_string(
        f"sqlite:///{checkpoint_path}?mode=rwc&journal_mode=WAL"
    )
```

### Risk 3: Interactive Mode Complexity

**Problem:** LangGraph's human-in-the-loop is node-based. Claude Code's is message-based. Need to unify UX.

**Mitigation:**
- Abstract interrupt semantics at runner-shell level
- Use `MessageType.WAITING_FOR_INPUT` consistently for both runners
- Document differences in UX for each workflow type

**Unified pattern:**
```python
# Both runners send same message type when waiting for input
await self.shell._send_message(
    MessageType.WAITING_FOR_INPUT,
    {
        "prompt": "Waiting for user approval",
        "context": {...}  # Runner-specific context
    }
)
```

### Risk 4: Migration Path

**Problem:** Existing RFE workflows must continue to work without breaking.

**Mitigation:**
- **Default behavior:** If `workflowType` is not specified, assume RFE
- **Backward compatibility:** Existing AgenticSessions use Claude Code runner
- **Gradual rollout:** Deploy LangGraph runner as opt-in initially

**Version strategy:**
```yaml
# CR annotation for versioning
metadata:
  annotations:
    vteam.ambient-code/workflow-version: "v1"
    vteam.ambient-code/runner-type: "claude-code"  # or "langgraph"
```

---

## 6. Implementation Phases

### Phase 1: Foundation (Week 1-2)

1. Create LangGraph adapter skeleton
   - File: `/components/runners/langgraph-runner/adapter.py`
   - Implement basic `initialize()`, `run()`, `handle_message()`
   - No graph execution yet - just echo messages back

2. Build separate container image
   - File: `/components/runners/langgraph-runner/Dockerfile`
   - Install LangGraph + dependencies
   - Test in isolation (no vTeam integration)

3. Operator configuration changes
   - Add `LANGGRAPH_RUNNER_IMAGE` to config
   - Implement `getRunnerImageForWorkflow()` function
   - Test with mock workflow type

**Success criteria:** Can create AgenticSession with `workflowType: langgraph` and operator launches LangGraph runner container.

### Phase 2: Core Execution (Week 3-4)

1. Implement graph execution
   - Load graph definition from workspace
   - Execute with streaming to WebSocket
   - Handle node events and send via `AGENT_MESSAGE`

2. Checkpoint persistence
   - SQLite checkpointer to PVC
   - Test state persistence across pod restarts
   - Implement checkpoint pruning

3. Session continuation
   - Load checkpoint from parent session
   - Resume graph from interrupt point
   - Verify workspace state preserved

**Success criteria:** LangGraph workflow executes simple graph, persists state, and can resume from checkpoint.

### Phase 3: Human-in-the-Loop (Week 5)

1. Interrupt handling
   - Pause graph at interrupt nodes
   - Send `WAITING_FOR_INPUT` message
   - Resume with user input from WebSocket

2. Interactive mode
   - Multi-turn conversation with graph
   - Handle abort/cancel signals
   - Graceful shutdown

**Success criteria:** Can pause LangGraph execution, get user input via UI, and resume graph.

### Phase 4: API & CRDs (Week 6)

1. LangGraphWorkflow CRD
   - Define schema
   - Backend handlers (create, list, get)
   - Validation logic

2. Frontend integration
   - UI for creating LangGraph workflows
   - Graph definition editor (optional for MVP)
   - Session execution UI (reuse existing)

**Success criteria:** Can create LangGraphWorkflow via API, create session referencing it, and execute graph end-to-end.

---

## 7. Open Questions for Discussion

1. **Graph Definition Format:** Should we support:
   - Python files (`graph.py` in workspace)?
   - JSON/YAML declarative format?
   - Both?

2. **Checkpointer Backend:** SQLite sufficient or need Postgres for production?
   - SQLite: Simple, no extra deps
   - Postgres: Better for large state, concurrent access
   - Recommendation: Start with SQLite, add Postgres as optional upgrade

3. **Workflow Versioning:** How to handle graph definition changes?
   - Immutable workflows (copy-on-edit)?
   - Version tags in CR?
   - Git-based versioning?

4. **Observability:** What metrics/logs do we need?
   - Graph execution timeline (which nodes ran, duration)
   - Checkpoint size (for PVC capacity planning)
   - Interrupt/resume statistics

5. **Security:** Graph definitions are Python code - sandboxing?
   - Run in restricted mode (disable `exec`, `eval`)?
   - Validate graph AST before execution?
   - Only allow pre-approved graph templates?

---

## 8. Testing Strategy

### Unit Tests

```python
# tests/runners/test_langgraph_adapter.py

@pytest.mark.asyncio
async def test_adapter_initialize():
    adapter = LangGraphAdapter()
    context = RunnerContext(session_id="test", workspace_path="/tmp/test")
    await adapter.initialize(context)
    assert adapter.graph is not None

@pytest.mark.asyncio
async def test_checkpoint_persistence():
    adapter = LangGraphAdapter()
    # ... setup
    result = await adapter.run()
    # Verify checkpoint written to PVC
    checkpoint_path = Path("/workspace/sessions/test/.langgraph/checkpoints.db")
    assert checkpoint_path.exists()
```

### Integration Tests

```python
# tests/integration/test_langgraph_workflow.py

async def test_end_to_end_workflow():
    # 1. Create LangGraphWorkflow CR via API
    # 2. Create AgenticSession with workflowType: langgraph
    # 3. Verify operator creates Job with correct image
    # 4. Verify graph executes and sends messages via WebSocket
    # 5. Verify checkpoint persisted
    # 6. Create continuation session
    # 7. Verify resumed from checkpoint
```

### Performance Tests

- **Checkpoint I/O latency:** Measure time to save/load checkpoints of various sizes
- **Concurrent sessions:** Verify multiple LangGraph sessions can run simultaneously
- **PVC capacity:** Test behavior when PVC fills up (checkpoint pruning)

---

## 9. Documentation Requirements

1. **User Guide:**
   - How to create LangGraph workflows
   - Graph definition syntax
   - Human-in-the-loop patterns
   - Migration from RFE workflows

2. **Developer Guide:**
   - How to add new runner types
   - Runner-shell adapter interface
   - Checkpoint patterns
   - Testing workflows locally

3. **Operations Guide:**
   - Deployment configuration
   - PVC sizing recommendations
   - Troubleshooting graph execution
   - Checkpoint backup/restore

---

## 10. Recommendations Summary

### Must Have (MVP)

1. ✅ **Separate container images** - Avoid dependency conflicts
2. ✅ **LangGraphWorkflow CRD** - Type-safe workflow definitions
3. ✅ **Operator workflow type selection** - Dynamic runner image selection
4. ✅ **Checkpoint persistence to PVC** - State management
5. ✅ **Session continuation support** - Resume from parent checkpoint

### Should Have (Post-MVP)

1. ⚠️ **Postgres checkpointer** - Better performance for large state
2. ⚠️ **Graph definition validation** - Security and error prevention
3. ⚠️ **Checkpoint pruning** - PVC management
4. ⚠️ **Observability metrics** - Graph execution telemetry

### Nice to Have (Future)

1. 💡 **Unified WorkflowTemplate CRD** - Generic workflow abstraction
2. 💡 **Visual graph editor** - UI for building LangGraph workflows
3. 💡 **Graph testing framework** - Unit test individual nodes
4. 💡 **Workflow marketplace** - Share pre-built graphs

---

## Appendix A: File Structure

```
vTeam/
├── components/
│   ├── runners/
│   │   ├── runner-shell/          # Unchanged
│   │   │   └── runner_shell/
│   │   │       ├── protocol.py    # Message types (unchanged)
│   │   │       ├── transport_ws.py
│   │   │       └── shell.py
│   │   ├── claude-code-runner/    # Existing
│   │   │   ├── wrapper.py
│   │   │   └── Dockerfile
│   │   └── langgraph-runner/      # NEW
│   │       ├── adapter.py         # LangGraph adapter
│   │       ├── graph_loader.py    # Load graph definitions
│   │       ├── checkpointer.py    # Checkpoint management
│   │       ├── Dockerfile
│   │       └── requirements.txt
│   ├── backend/
│   │   ├── handlers/
│   │   │   ├── rfe.go             # Existing
│   │   │   ├── langgraph.go       # NEW: LangGraph workflow handlers
│   │   │   └── sessions.go        # Modify: workflow type handling
│   │   └── types/
│   │       ├── rfe.go             # Existing
│   │       └── langgraph.go       # NEW: LangGraphWorkflow types
│   ├── operator/
│   │   └── internal/
│   │       ├── config/
│   │       │   └── config.go      # Add: LangGraphRunnerImage
│   │       └── handlers/
│   │           └── sessions.go    # Modify: getRunnerImageForWorkflow()
│   └── manifests/
│       └── crds/
│           ├── agenticsessions-crd.yaml    # Modify: add workflowType
│           ├── rfeworkflows-crd.yaml       # Existing
│           └── langgraphworkflows-crd.yaml # NEW
```

---

## Appendix B: Code Snippets

### B.1: LangGraph Adapter Skeleton

```python
# /components/runners/langgraph-runner/adapter.py

import asyncio
import logging
from pathlib import Path
from typing import Optional, Dict, Any

from langgraph.graph import StateGraph
from langgraph.checkpoint.sqlite import AsyncSqliteSaver
from runner_shell.core.context import RunnerContext
from runner_shell.core.protocol import MessageType

logger = logging.getLogger(__name__)


class LangGraphAdapter:
    """Adapter for executing LangGraph workflows in vTeam platform."""

    def __init__(self):
        self.context: Optional[RunnerContext] = None
        self.shell = None
        self.graph: Optional[StateGraph] = None
        self.checkpointer: Optional[AsyncSqliteSaver] = None
        self.thread_id: Optional[str] = None

    async def initialize(self, context: RunnerContext):
        """Initialize LangGraph runtime with workspace context."""
        self.context = context
        logger.info(f"Initializing LangGraph adapter for session {context.session_id}")

        # Load graph definition from workspace
        graph_path = self._find_graph_definition()
        self.graph = await self._load_graph(graph_path)

        # Set up checkpoint persistence
        self.checkpointer = await self._setup_checkpointer()

        # Determine thread ID (new or resume)
        parent_session = context.get_env('PARENT_SESSION_ID', '')
        if parent_session:
            self.thread_id = await self._get_parent_thread_id(parent_session)
        else:
            self.thread_id = context.session_id

    async def run(self) -> Dict[str, Any]:
        """Execute graph with streaming output to WebSocket."""
        try:
            await self._send_log("Starting LangGraph execution...")

            # Get initial input from prompt
            prompt = self.context.get_env("PROMPT", "")
            if not prompt:
                raise ValueError("PROMPT environment variable is required")

            # Execute graph with streaming
            config = {
                "configurable": {"thread_id": self.thread_id},
                "recursion_limit": 100,
            }

            async for event in self.graph.astream_events(
                {"input": prompt},
                config=config,
                version="v1"
            ):
                await self._handle_graph_event(event)

            await self._send_log("LangGraph execution completed")

            return {
                "success": True,
                "thread_id": self.thread_id,
            }

        except Exception as e:
            logger.error(f"LangGraph execution failed: {e}")
            return {
                "success": False,
                "error": str(e),
            }

    async def handle_message(self, message: Dict[str, Any]):
        """Handle incoming WebSocket messages (user input, interrupts)."""
        msg_type = message.get('type', '')

        if msg_type == 'user_message':
            # Queue user input for resuming graph
            payload = message.get('payload', {})
            user_input = payload.get('content', '')
            # TODO: Resume graph with user input

        elif msg_type == 'interrupt':
            # Interrupt graph execution
            # TODO: Implement interrupt handling
            pass

    async def _handle_graph_event(self, event: Dict[str, Any]):
        """Process graph execution events and send to backend."""
        event_type = event.get("event")

        if event_type == "on_chain_start":
            node_name = event.get("name", "")
            await self._send_log(f"Executing node: {node_name}")

        elif event_type == "on_chain_end":
            output = event.get("data", {}).get("output", {})
            await self.shell._send_message(
                MessageType.AGENT_MESSAGE,
                {"type": "node_output", "output": output}
            )

        elif event_type == "on_interrupt":
            # Graph hit interrupt node - wait for user input
            await self.shell._send_message(
                MessageType.WAITING_FOR_INPUT,
                {"prompt": "Waiting for user input"}
            )

    async def _load_graph(self, graph_path: Path) -> StateGraph:
        """Load graph definition from Python file."""
        # TODO: Implement graph loading from file
        # Security consideration: validate graph definition
        raise NotImplementedError()

    async def _setup_checkpointer(self) -> AsyncSqliteSaver:
        """Set up SQLite checkpointer for state persistence."""
        checkpoint_dir = Path(self.context.workspace_path).parent / ".langgraph"
        checkpoint_dir.mkdir(parents=True, exist_ok=True)

        db_path = checkpoint_dir / "checkpoints.db"
        return await AsyncSqliteSaver.from_conn_string(
            f"sqlite:///{db_path}?journal_mode=WAL"
        )

    def _find_graph_definition(self) -> Path:
        """Locate graph definition file in workspace."""
        # Look for graph.py in workspace root
        workspace = Path(self.context.workspace_path)
        graph_file = workspace / "graph.py"

        if not graph_file.exists():
            raise FileNotFoundError(
                f"Graph definition not found at {graph_file}. "
                "Please create graph.py in your workspace."
            )

        return graph_file

    async def _get_parent_thread_id(self, parent_session: str) -> str:
        """Fetch thread ID from parent session for continuation."""
        # TODO: Query parent session CR for thread_id annotation
        return parent_session  # Fallback: use session ID as thread ID

    async def _send_log(self, message: str):
        """Send system log message via WebSocket."""
        if self.shell:
            await self.shell._send_message(
                MessageType.SYSTEM_MESSAGE,
                {"message": message}
            )
```

### B.2: Operator Workflow Type Selection

```go
// MODIFY: /components/operator/internal/handlers/sessions.go

// Add after line 250 (Load config for this session)

// Determine runner image based on workflow type
runnerImage := getRunnerImageForSession(currentObj, appConfig)

// ...

// Replace line 398:
// Image: appConfig.AmbientCodeRunnerImage,
Image: runnerImage,

// Add helper function at end of file:

func getRunnerImageForSession(obj *unstructured.Unstructured, config *config.Config) string {
	spec, _, _ := unstructured.NestedMap(obj.Object, "spec")

	// Check workflowType field
	workflowType, _, _ := unstructured.NestedString(spec, "workflowType")

	switch strings.ToLower(workflowType) {
	case "langgraph":
		if config.LangGraphRunnerImage == "" {
			log.Printf("⚠️ LangGraph runner image not configured, falling back to Claude runner")
			return config.AmbientCodeRunnerImage
		}
		return config.LangGraphRunnerImage

	case "rfe", "":
		// Default to RFE/Claude runner for backward compatibility
		return config.AmbientCodeRunnerImage

	default:
		log.Printf("⚠️ Unknown workflow type '%s', using Claude runner", workflowType)
		return config.AmbientCodeRunnerImage
	}
}
```

---

## Conclusion

The vTeam architecture is well-positioned for multi-runner support. The runner-shell abstraction provides clean separation between execution engines and platform infrastructure. The main implementation effort is in building the LangGraph adapter, managing state checkpoints, and extending the operator to select runner images based on workflow type.

**Estimated effort:** 6-8 weeks for MVP with core functionality.

**Key success factors:**
1. Keep runner-shell interface stable
2. Use separate container images to avoid dependency conflicts
3. Leverage PVC for all state persistence (checkpoints, workspaces)
4. Start with CRD-per-workflow pattern for type safety
5. Ensure backward compatibility with existing RFE workflows

This design allows incremental rollout - we can deploy LangGraph runner as opt-in while keeping RFE workflows unchanged. The architecture supports adding more runner types in the future (e.g., CrewAI, AutoGPT) without major refactoring.

---

**Next Steps:**
1. Review with architecture team
2. Create detailed RFE document for stakeholders
3. Prototype LangGraph adapter in sandbox environment
4. Performance testing: checkpoint I/O, concurrent sessions
5. Security review: graph definition execution sandboxing
