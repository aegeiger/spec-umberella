# Data Model: LangGraph Runner Support

**Feature**: LangGraph Runner Support
**Date**: 2025-11-04
**Status**: Design Phase

## Overview

This document defines the data entities and their relationships for adding LangGraph runner support to the vTeam platform. All entities align with the existing vTeam architecture patterns and maintain backward compatibility.

---

## Entity Definitions

### 1. AgenticSession CR (Modified)

**Location**: Kubernetes Custom Resource Definition
**Scope**: Cluster-wide, namespaced

**Purpose**: Represents a single agentic session execution. Extended to include runner type selection.

#### Schema

```yaml
apiVersion: vteam.ambient-code/v1alpha1
kind: AgenticSession
metadata:
  name: agentic-session-{timestamp}
  namespace: project-namespace
  labels:
    project: project-name
    rfe-workflow: workflow-id  # optional
  annotations:
    sdk-session-id: session-id-for-continuation  # optional

spec:
  # NEW FIELD: Runner type selection
  runnerType: string  # "claude-code" | "langgraph" | defaults to "claude-code"

  # Existing fields (unchanged)
  prompt: string
  displayName: string
  project: string
  llmSettings:
    model: string
    temperature: float
    maxTokens: int
  timeout: int  # seconds
  interactive: bool

  # Multi-repo configuration
  repos:
    - input:
        url: string
        branch: string
      output:
        url: string
        branch: string
  mainRepoIndex: int

  # Behavior
  autoPushOnComplete: bool

  # Context
  userContext:
    userId: string
    displayName: string
    groups: []string
  botAccount:
    name: string

  # Custom configuration
  environmentVariables:
    key: value
  resourceOverrides:
    cpu: string
    memory: string
    storageClass: string
    priorityClass: string

status:
  phase: string  # "Pending" | "Creating" | "Running" | "Completed" | "Failed"
  startTime: string
  completionTime: string
  error: string  # if phase == "Failed"
  artifacts:
    - type: string  # "rfe.md" | "spec.md" | "plan.md" | "tasks.md"
      path: string
      commitSHA: string
```

#### Field Validation

- **runnerType**:
  - Type: `string`
  - Allowed values: `"claude-code"`, `"langgraph"`
  - Default: `"claude-code"` (if not specified)
  - Immutable: Cannot change after session creation
  - Validation: Backend returns 400 Bad Request if invalid value provided

#### Relationships

- **Owns**: Kubernetes Job (via owner references)
- **References**: PVC (workspace persistence)
- **Referenced by**: Frontend UI (session detail view), Backend API (status updates)

---

### 2. RunnerAdapter Interface (Abstract)

**Location**: `/workspace/sessions/{session}/workspace/vTeam/components/runners/runner-shell/runner_shell/core/adapter.py` (new file)

**Purpose**: Abstract interface contract that all runner implementations must implement. Enables polymorphic runner selection.

#### Python Type Definition

```python
from abc import ABC, abstractmethod
from typing import Dict, Any

class RunnerAdapter(ABC):
    """Abstract adapter interface for runner implementations."""

    @abstractmethod
    async def initialize(self, context: RunnerContext) -> None:
        """
        Initialize the adapter with execution context.

        Args:
            context: RunnerContext containing session_id, workspace_path, environment

        Raises:
            ValueError: If prerequisites are missing (e.g., spec.md for plan phase)
            RuntimeError: If workspace initialization fails

        Side Effects:
            - Clones repositories to workspace
            - Validates prerequisite files
            - Configures git identity
        """
        pass

    @abstractmethod
    async def run(self) -> Dict[str, Any]:
        """
        Execute the runner session.

        Returns:
            dict: Result metadata with keys:
                - status: "completed" | "failed"
                - artifacts: List of {type, path, commitSHA}
                - error: Optional error message if status == "failed"

        Side Effects:
            - Sends WebSocket messages via shell._send_message()
            - Updates AgenticSession CR status via BOT_TOKEN
            - Commits and pushes artifacts to output repository
        """
        pass

    @abstractmethod
    async def handle_message(self, message: Dict[str, Any]) -> None:
        """
        Handle incoming messages from backend (e.g., user input in interactive mode).

        Args:
            message: Dict with keys:
                - type: "user.message" | "system.message"
                - payload: Message content
                - timestamp: ISO format timestamp

        Side Effects:
            - May queue message for processing by runner
            - May trigger state transitions in runner
        """
        pass
```

#### Validation Rules

- All methods must be implemented (Python ABC enforcement)
- `initialize()` must be called before `run()`
- `run()` must return dict with `status` key
- `handle_message()` must not block (use async queue if needed)

#### Relationships

- **Implemented by**: ClaudeCodeAdapter, LangGraphAdapter
- **Used by**: RunnerShell

---

### 3. ClaudeCodeAdapter (Unchanged)

**Location**: `/workspace/sessions/{session}/workspace/vTeam/components/runners/claude-code-runner/wrapper.py`

**Purpose**: Adapter wrapping existing Claude Code CLI. **NO MODIFICATIONS** (FR-022 compliance).

#### Implementation Status

**IMMUTABLE**: This entity exists in production and MUST NOT be modified. Included in data model for completeness only.

---

### 4. LangGraphAdapter (New)

**Location**: `/workspace/sessions/{session}/workspace/vTeam/components/runners/langgraph-runner/src/adapters/langgraph_adapter.py` (new file)

**Purpose**: Adapter implementing LangGraph-based runner. Implements RunnerAdapter interface.

#### Python Class Definition

```python
from typing import Dict, Any, Optional
from runner_shell.core.adapter import RunnerAdapter
from runner_shell.core.context import RunnerContext
from runner_shell.core.shell import RunnerShell
from .workflows.rfe_workflow import RFEWorkflow
from .state import RFEState

class LangGraphAdapter(RunnerAdapter):
    """LangGraph-based runner adapter for RFE workflows."""

    def __init__(self):
        self.shell: Optional[RunnerShell] = None
        self.context: Optional[RunnerContext] = None
        self.workflow: Optional[RFEWorkflow] = None
        self.state: Optional[RFEState] = None

    async def initialize(self, context: RunnerContext) -> None:
        """Initialize LangGraph runner with context."""
        self.context = context

        # Prepare workspace (reuse runner-shell utilities)
        await self._prepare_workspace()

        # Validate prerequisites based on detected phase
        phase = self._detect_phase()
        self._validate_prerequisites(phase)

        # Initialize LangGraph workflow
        self.workflow = RFEWorkflow(
            llm_model=context.get_env("LLM_MODEL", "claude-3-7-sonnet-latest"),
            temperature=float(context.get_env("LLM_TEMPERATURE", "0.7")),
            max_tokens=int(context.get_env("LLM_MAX_TOKENS", "4000"))
        )

        # Initialize state
        self.state = RFEState(
            phase=phase,
            prompt=context.get_env("PROMPT", ""),
            workspace_path=context.workspace_path,
            repos=self._load_repos_config(),
            rfe_content="",
            spec_content="",
            plan_content="",
            tasks_content="",
            error=None
        )

    async def run(self) -> Dict[str, Any]:
        """Execute LangGraph workflow."""
        try:
            # Execute phase graph
            result_state = await self.workflow.run(
                phase=self.state["phase"],
                state=self.state,
                websocket_callback=self._send_websocket_message
            )

            # Commit artifacts
            artifacts = await self._commit_artifacts(result_state)

            # Update CR status
            await self._update_cr_status("Completed", artifacts)

            return {
                "status": "completed",
                "artifacts": artifacts,
                "phase": result_state["phase"]
            }

        except Exception as e:
            error_msg = f"LangGraph execution failed: {str(e)}"
            await self._update_cr_status("Failed", [], error_msg)

            return {
                "status": "failed",
                "error": error_msg
            }

    async def handle_message(self, message: Dict[str, Any]) -> None:
        """Handle incoming backend messages."""
        # For MVP: Queue messages (interactive mode future enhancement)
        if self.workflow:
            await self.workflow.queue_user_message(message)

    # Private helper methods
    async def _prepare_workspace(self) -> None: ...
    def _detect_phase(self) -> str: ...
    def _validate_prerequisites(self, phase: str) -> None: ...
    def _load_repos_config(self) -> list: ...
    async def _send_websocket_message(self, msg: Dict) -> None: ...
    async def _commit_artifacts(self, state: RFEState) -> list: ...
    async def _update_cr_status(self, phase: str, artifacts: list, error: str = None) -> None: ...
```

#### State Transitions

```
initialize() → run() → handle_message() (optional, for interactive mode)
```

#### Relationships

- **Implements**: RunnerAdapter
- **Uses**: RFEWorkflow, RFEState, runner-shell utilities
- **Interacts with**: RunnerShell (via shell reference)

---

### 5. RFEState (New)

**Location**: `/workspace/sessions/{session}/workspace/vTeam/components/runners/langgraph-runner/src/state.py` (new file)

**Purpose**: Type-safe state container shared across LangGraph workflow phases.

#### Python TypedDict Definition

```python
from typing import TypedDict, Literal, Optional, List, Dict

PhaseType = Literal["ideate", "specify", "plan", "tasks"]

class RepoConfig(TypedDict):
    name: str
    input_url: str
    input_branch: str
    output_url: Optional[str]
    output_branch: Optional[str]

class RFEState(TypedDict):
    # Execution context
    phase: PhaseType
    prompt: str
    workspace_path: str
    repos: List[RepoConfig]

    # Accumulated artifacts
    rfe_content: str       # From ideate phase
    spec_content: str      # From specify phase
    plan_content: str      # From plan phase
    tasks_content: str     # From tasks phase

    # Error tracking
    error: Optional[str]

    # Internal execution metadata (not persisted)
    current_node: Optional[str]  # For debugging
    llm_messages: List[Dict]     # Conversation history
```

#### Field Descriptions

- **phase**: Current RFE Workflow phase being executed
- **prompt**: User's initial prompt (or continuation prompt)
- **workspace_path**: Absolute path to workspace directory
- **repos**: Multi-repo configuration from REPOS_JSON
- **{phase}_content**: Markdown content for each artifact (empty string if not yet generated)
- **error**: Error message if execution fails (None otherwise)

#### Validation Rules

- `phase` must be one of: "ideate", "specify", "plan", "tasks"
- For "specify" phase: `rfe_content` must be non-empty
- For "plan" phase: `spec_content` must be non-empty
- For "tasks" phase: `plan_content` must be non-empty

#### State Flow

```
Initial State → Ideate Graph → State + rfe_content
             → Specify Graph → State + spec_content
             → Plan Graph    → State + plan_content
             → Tasks Graph   → State + tasks_content
```

---

### 6. RFE Workflow Artifacts (Existing, Unchanged)

**Location**: Git repository at `specs/{branchName}/{artifact}.md`

**Purpose**: Structured markdown documents produced by runners following standardized templates.

#### Artifact Types

| Type | Filename | Producer Phase | Schema Location |
|------|----------|----------------|-----------------|
| RFE Document | `rfe.md` | Ideate | `.specify/templates/rfe-template.md` |
| Specification | `spec.md` | Specify | `.specify/templates/spec-template.md` |
| Implementation Plan | `plan.md` | Plan | `.specify/templates/plan-template.md` |
| Task Breakdown | `tasks.md` | Tasks | `.specify/templates/tasks-template.md` |

#### Validation

All artifacts must:
- Be valid markdown
- Follow template structure (mandatory sections present)
- Include metadata header (feature name, branch, date)
- Be committed to correct Git branch
- Be placed in `specs/{branchName}/` directory

#### Relationships

- **Produced by**: ClaudeCodeAdapter, LangGraphAdapter
- **Consumed by**: Subsequent RFE phases, vTeam UI
- **Stored in**: Git repository output branch

---

### 7. LangGraph Workflow Graphs (New)

**Location**: `/workspace/sessions/{session}/workspace/vTeam/components/runners/langgraph-runner/src/workflows/{phase}_graph.py` (4 files)

**Purpose**: LangGraph StateGraph instances defining execution flow for each RFE phase.

#### Graph Structure (Conceptual)

Each phase has a separate graph with internal nodes:

**Ideate Graph**:
```
START → load_context → ideate_llm → validate_rfe → write_artifact → END
                                          ↓
                                      (error) → retry_ideate (max 3)
```

**Specify Graph**:
```
START → load_rfe → specify_llm → validate_spec → write_artifact → END
                                      ↓
                                  (error) → retry_specify (max 3)
```

**Plan Graph**:
```
START → load_spec → plan_llm → validate_plan → write_artifact → END
                                     ↓
                                 (error) → retry_plan (max 3)
```

**Tasks Graph**:
```
START → load_plan → tasks_llm → validate_tasks → write_artifact → END
                                      ↓
                                  (error) → retry_tasks (max 3)
```

#### Node Types

- **load_{artifact}**: Reads prerequisite artifact from workspace into state
- **{phase}_llm**: Calls Claude via ChatAnthropic with phase-specific prompt and tools
- **validate_{artifact}**: Validates output artifact matches template schema
- **write_artifact**: Writes validated artifact to `specs/{branchName}/{artifact}.md`
- **retry_{phase}**: Error recovery node (max 3 attempts)

#### Relationships

- **Owned by**: RFEWorkflow class
- **Uses**: RFEState (shared state)
- **Calls**: LangChain tools (write_file, read_file, git_commit)

---

### 8. CreateAgenticSessionRequest (Modified)

**Location**: Backend API type definition

**Purpose**: API request body for creating new agentic sessions. Extended to include runner type selection.

#### TypeScript Type (Frontend)

```typescript
interface CreateAgenticSessionRequest {
  // NEW FIELD: Runner type selection
  runnerType?: "claude-code" | "langgraph";  // Optional, defaults to "claude-code"

  // Existing fields (unchanged)
  prompt: string;
  displayName?: string;
  llmSettings?: {
    model: string;
    temperature: number;
    maxTokens: number;
  };
  timeout?: number;
  interactive?: boolean;
  workspacePath?: string;
  parent_session_id?: string;

  // Multi-repo unified mapping
  repos?: Array<{
    input: {
      url: string;
      branch?: string;
    };
    output?: {
      url: string;
      branch?: string;
    };
  }>;
  mainRepoIndex?: number;

  // Runner behavior
  autoPushOnComplete?: boolean;

  // Context & Resource Configuration
  userContext?: {
    userId: string;
    displayName: string;
    groups: string[];
  };
  botAccount?: {
    name: string;
  };
  resourceOverrides?: {
    cpu?: string;
    memory?: string;
    storageClass?: string;
    priorityClass?: string;
  };
  environmentVariables?: Record<string, string>;
  labels?: Record<string, string>;
  annotations?: Record<string, string>;
}
```

#### Go Type (Backend)

```go
type CreateAgenticSessionRequest struct {
    // NEW FIELD: Runner type selection
    RunnerType *string `json:"runnerType,omitempty"`  // Optional, defaults to "claude-code"

    // Existing fields (unchanged)
    Prompt               string                `json:"prompt" binding:"required"`
    DisplayName          string                `json:"displayName,omitempty"`
    LLMSettings          *LLMSettings          `json:"llmSettings,omitempty"`
    Timeout              *int                  `json:"timeout,omitempty"`
    Interactive          *bool                 `json:"interactive,omitempty"`
    WorkspacePath        string                `json:"workspacePath,omitempty"`
    ParentSessionID      string                `json:"parent_session_id,omitempty"`
    Repos                []SessionRepoMapping  `json:"repos,omitempty"`
    MainRepoIndex        *int                  `json:"mainRepoIndex,omitempty"`
    AutoPushOnComplete   *bool                 `json:"autoPushOnComplete,omitempty"`
    UserContext          *UserContext          `json:"userContext,omitempty"`
    BotAccount           *BotAccountRef        `json:"botAccount,omitempty"`
    ResourceOverrides    *ResourceOverrides    `json:"resourceOverrides,omitempty"`
    EnvironmentVariables map[string]string     `json:"environmentVariables,omitempty"`
    Labels               map[string]string     `json:"labels,omitempty"`
    Annotations          map[string]string     `json:"annotations,omitempty"`
}
```

#### Validation Rules

**Backend Validation** (in CreateSession handler):

```go
// Validate runnerType
if req.RunnerType != nil {
    validRunnerTypes := []string{"claude-code", "langgraph"}
    if !contains(validRunnerTypes, *req.RunnerType) {
        c.JSON(400, gin.H{
            "error": fmt.Sprintf(
                "Unsupported runner type '%s'. Supported types: %s",
                *req.RunnerType,
                strings.Join(validRunnerTypes, ", ")
            ),
        })
        return
    }
}

// Apply default
runnerType := "claude-code"
if req.RunnerType != nil {
    runnerType = *req.RunnerType
}

// Populate CR spec
spec["runnerType"] = runnerType
```

#### Relationships

- **Sent by**: Frontend UI (session creation form)
- **Received by**: Backend API (CreateSession handler)
- **Transforms to**: AgenticSession CR spec

---

### 9. Operator Configuration (Modified)

**Location**: Operator environment variables

**Purpose**: Configuration for runner image selection.

#### Configuration Schema

```yaml
env:
  # NEW: LangGraph runner image
  - name: AMBIENT_LANGGRAPH_RUNNER_IMAGE
    value: "quay.io/ambient_code/vteam_langgraph_runner:latest"

  # Existing: Claude Code runner image (unchanged)
  - name: AMBIENT_CODE_RUNNER_IMAGE
    value: "quay.io/ambient_code/vteam_claude_runner:latest"

  # Existing: Content service image (unchanged)
  - name: CONTENT_SERVICE_IMAGE
    value: "quay.io/ambient_code/vteam_backend:latest"
```

#### Image Selection Logic (Go)

```go
// In handleAgenticSessionEvent function
func getRunnerImage(spec map[string]interface{}, config *config.Config) string {
    runnerType := "claude-code"  // default
    if rt, ok := spec["runnerType"].(string); ok && rt != "" {
        runnerType = rt
    }

    switch runnerType {
    case "langgraph":
        return config.AmbientLangGraphRunnerImage
    case "claude-code":
        return config.AmbientCodeRunnerImage
    default:
        log.Warnf("Unknown runner type '%s', defaulting to claude-code", runnerType)
        return config.AmbientCodeRunnerImage
    }
}
```

#### Relationships

- **Read by**: Operator (during Job provisioning)
- **Determines**: Which container image is used for runner Job

---

## Entity Relationships Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                     Frontend UI                                  │
│                 (Session Creation Form)                          │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ CreateAgenticSessionRequest
                         │ {runnerType: "langgraph"}
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│                     Backend API                                  │
│              (CreateSession Handler)                             │
│  - Validates runnerType                                          │
│  - Creates AgenticSession CR                                     │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ AgenticSession CR
                         │ spec.runnerType: "langgraph"
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│               Kubernetes Operator                                │
│           (Watch AgenticSession CRs)                             │
│  - Reads spec.runnerType                                         │
│  - Selects runner image via getRunnerImage()                     │
│  - Creates Kubernetes Job                                        │
└────────────────────────┬────────────────────────────────────────┘
                         │
         ┌───────────────┴───────────────┐
         │                               │
         ↓ (if langgraph)                ↓ (if claude-code)
┌──────────────────────────┐   ┌──────────────────────────┐
│  LangGraph Runner Job    │   │ Claude Code Runner Job   │
│  (Pod with container)    │   │  (Pod with container)    │
└──────────┬───────────────┘   └──────────┬───────────────┘
           │                              │
           │ Instantiates                 │ Instantiates
           ↓                              ↓
┌──────────────────────────┐   ┌──────────────────────────┐
│   LangGraphAdapter       │   │   ClaudeCodeAdapter      │
│  (implements interface)  │   │  (implements interface)  │
└──────────┬───────────────┘   └──────────┬───────────────┘
           │                              │
           │                              │
           └──────────┬───────────────────┘
                      │
                      │ Uses
                      ↓
           ┌──────────────────────────┐
           │   RunnerShell            │
           │  (Orchestrator)          │
           │  - WebSocket transport   │
           │  - Workspace management  │
           └──────────┬───────────────┘
                      │
                      │ Produces
                      ↓
           ┌──────────────────────────┐
           │  RFE Workflow Artifacts  │
           │  - rfe.md                │
           │  - spec.md               │
           │  - plan.md               │
           │  - tasks.md              │
           └──────────────────────────┘
```

---

## Data Flow

### Session Creation Flow

1. **User** submits form in Frontend UI with `runnerType: "langgraph"`
2. **Frontend** sends `CreateAgenticSessionRequest` with `runnerType` to Backend API
3. **Backend API** validates `runnerType` (returns 400 if invalid)
4. **Backend API** creates `AgenticSession` CR with `spec.runnerType: "langgraph"`
5. **Operator** watches CR, extracts `spec.runnerType`
6. **Operator** selects image via `getRunnerImage()` → returns `AMBIENT_LANGGRAPH_RUNNER_IMAGE`
7. **Operator** creates Kubernetes Job with LangGraph runner image

### Session Execution Flow

1. **Job Pod** starts, instantiates `LangGraphAdapter`
2. **RunnerShell** calls `adapter.initialize(context)`
3. **LangGraphAdapter** prepares workspace, validates prerequisites, initializes RFEWorkflow
4. **RunnerShell** calls `adapter.run()`
5. **LangGraphAdapter** executes phase graph (ideate/specify/plan/tasks)
6. **LangGraph Workflow** calls Claude API, uses tools (write_file, git_commit), streams to WebSocket
7. **LangGraphAdapter** commits artifacts, updates CR status to Completed
8. **RunnerShell** sends `session.completed` message, terminates

### Artifact Production Flow

1. **Phase Graph** (e.g., ideate_graph) executes `ideate_llm` node
2. **LLM Node** calls ChatAnthropic with prompt and tools
3. **Claude** generates artifact content, calls `write_file` tool
4. **write_file Tool** writes to `specs/{branchName}/rfe.md`
5. **validate_rfe Node** validates artifact against schema
6. **write_artifact Node** finalizes artifact, adds to state
7. **LangGraphAdapter** commits file via `git_commit` tool
8. **Git Push** sends artifact to output repository

---

## Schema Validation

### AgenticSession CR Validation

**OpenAPI Schema** (Kubernetes CRD):

```yaml
spec:
  properties:
    runnerType:
      type: string
      enum: ["claude-code", "langgraph"]
      default: "claude-code"
      description: "Runner implementation to use for session execution"
```

### RFE Artifact Validation

**Schema Location**: `.specify/templates/{artifact}-template.md`

**Validation Rules**:
- Must contain all mandatory sections from template
- Must have metadata header with feature name, branch, date
- Must be valid markdown (parseable by markdown parser)
- Must not exceed 50KB in size (quality check)

---

## Backward Compatibility

### Existing Sessions

- Sessions created before LangGraph support: `spec.runnerType` is `undefined`
- Operator behavior: Treats `undefined` as `"claude-code"` (default)
- Result: Zero impact on existing sessions

### API Requests

- Requests without `runnerType` field: Backend defaults to `"claude-code"`
- Result: Existing API clients work without modification

### Claude Code Runner

- No schema changes to ClaudeCodeAdapter
- No behavior changes to claude-code-runner implementation
- Result: 100% backward compatibility (FR-022 compliance)

---

## Summary

This data model extends vTeam with minimal changes:

- **1 new field**: `spec.runnerType` in AgenticSession CR
- **1 new configuration**: `AMBIENT_LANGGRAPH_RUNNER_IMAGE` environment variable
- **1 new adapter**: LangGraphAdapter implementing existing RunnerAdapter interface
- **4 new graphs**: LangGraph workflow graphs (internal to LangGraphAdapter)
- **1 new state**: RFEState TypedDict (internal to LangGraphAdapter)

**Zero breaking changes** to existing entities. All modifications are additive and maintain backward compatibility.
