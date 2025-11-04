# Research: LangGraph Runner Support

**Feature**: LangGraph Runner Support
**Date**: 2025-11-04
**Status**: Research Complete

## Research Objectives

This research phase resolves all "NEEDS CLARIFICATION" items from the Technical Context and provides concrete implementation guidance for adding LangGraph runner support to the vTeam platform.

## Key Research Areas

1. **Existing vTeam Architecture** - Understanding ClaudeCodeAdapter interface and integration patterns
2. **LangGraph Framework** - Best practices for state machines, tool integration, and streaming
3. **Testing Strategy** - Comprehensive approach to ensure zero regression and validate new functionality

---

## 1. Existing vTeam Architecture

### ClaudeCodeAdapter Interface Contract

**Location**: `/workspace/sessions/agentic-session-1762293354/workspace/vTeam/components/runners/claude-code-runner/wrapper.py`

The existing runner implements a clean adapter interface:

```python
class ClaudeCodeAdapter:
    async def initialize(self, context: RunnerContext):
        """Setup runner with context, prepare workspace, validate prerequisites"""

    async def run(self) -> dict:
        """Execute runner session, return result dict"""

    async def handle_message(self, message: dict):
        """Process incoming backend messages"""
```

**Decision**: LangGraphAdapter MUST implement identical interface signature to enable polymorphic runner selection without operator changes.

**Rationale**: The runner-shell framework expects this exact interface and routes all lifecycle events through these three methods. Maintaining interface compatibility ensures zero changes to infrastructure code.

### Operator Image Selection Pattern

**Location**: `/workspace/sessions/agentic-session-1762293354/workspace/vTeam/components/operator/internal/handlers/sessions.go`

Current implementation:
```go
// Load from environment variable
appConfig := config.LoadConfig()
Image: appConfig.AmbientCodeRunnerImage  // From AMBIENT_CODE_RUNNER_IMAGE
```

**Decision**: Add new environment variable `AMBIENT_LANGGRAPH_RUNNER_IMAGE` for LangGraph runner container image. Operator will read `spec.runnerType` field from AgenticSession CR and select appropriate image.

**Rationale**: Follows established configuration pattern. Minimizes operator code changes - only need to add conditional image selection logic based on `runnerType`.

**Alternatives Considered**:
- Image naming convention (ambient-{runnerType}-runner) - Rejected because hardcoded patterns reduce flexibility
- ConfigMap-based mapping - Rejected as over-engineering for two runner types

### Multi-Repo Support Architecture

**Key Environment Variables**:
- `REPOS_JSON` - JSON array of repository configurations
- `MAIN_REPO_INDEX` - Which repository becomes the working directory
- Workspace structure: `/workspace/sessions/{session_id}/workspace/{repo_name}/`

**Decision**: LangGraph runner will reuse exact same multi-repo initialization logic from runner-shell framework. No custom implementation needed.

**Rationale**: Repository cloning and workspace management are already abstracted in runner-shell. LangGraphAdapter initialization should call the same workspace setup utilities used by ClaudeCodeAdapter.

### WebSocket Streaming Protocol

**Protocol Definition**: `/workspace/sessions/agentic-session-1762293354/workspace/vTeam/components/runners/runner-shell/runner_shell/core/protocol.py`

Message types:
- `AGENT_MESSAGE` - Structured agent outputs
- `SYSTEM_MESSAGE` - System logs and diagnostics
- `MESSAGE_PARTIAL` - Large message fragments with sequencing
- `AGENT_RUNNING` - Status updates

**Decision**: LangGraph runner will emit identical message types using RunnerShell's `_send_message()` method. LangGraph execution events map to protocol messages as follows:
- `on_llm_stream` → `MESSAGE_PARTIAL` (token streaming)
- `on_tool_start/end` → `AGENT_MESSAGE` (tool execution updates)
- `on_chain_end` → `AGENT_MESSAGE` (phase completion)

**Rationale**: Frontend and backend already parse these message types. Maintaining protocol compatibility ensures zero UI changes.

---

## 2. LangGraph Framework Best Practices

### State Machine Design

**Pattern**: TypedDict-based state with explicit phase transitions

```python
from typing import TypedDict, Literal

class RFEState(TypedDict):
    phase: Literal["ideate", "specify", "plan", "tasks"]
    prompt: str
    rfe_content: str
    spec_content: str
    plan_content: str
    tasks_content: str
    error: Optional[str]
```

**Decision**: Implement four sequential LangGraph graphs (one per RFE phase: Ideate, Specify, Plan, Tasks). Each graph has internal nodes for LLM calls, tool execution, and validation. Graphs share state through RFEState TypedDict.

**Rationale**: Separate graphs per phase provide clean boundaries, easier testing, and align with RFE Workflow structure. Shared state enables artifact accumulation across phases.

**Alternatives Considered**:
- Single monolithic graph - Rejected due to complexity and poor testability
- Separate state per phase - Rejected because phases need context from previous artifacts

### Tool Integration Pattern

**Recommended Approach**: LangChain's `@tool` decorator with ToolNode orchestration

```python
from langchain_core.tools import tool
from langgraph.prebuilt import ToolNode

@tool
def write_file(path: str, content: str) -> str:
    """Write content to file."""
    try:
        Path(path).write_text(content)
        return f"Successfully wrote {len(content)} chars to {path}"
    except Exception as e:
        return f"Error writing file: {str(e)}"

# Bind tools to LLM
llm_with_tools = llm.bind_tools([write_file, read_file, git_commit])
```

**Decision**: Implement core tools as:
- `write_file` / `read_file` - Artifact creation in specs/{branchName}/ directory
- `git_commit` / `git_push` - Git operations using existing runner-shell utilities
- `validate_artifact` - Schema validation for RFE artifacts
- `websocket_send` - Progress updates to frontend

Tools return error strings (not exceptions) so LLM can handle errors gracefully.

**Rationale**: `@tool` decorator provides automatic schema generation for LLM. ToolNode handles error serialization and retry logic. Pattern proven in production LangGraph implementations.

### LLM Provider Integration

**Stack**: Anthropic Claude via `langchain-anthropic`

```python
from langchain_anthropic import ChatAnthropic

llm = ChatAnthropic(
    model="claude-3-7-sonnet-latest",
    temperature=0.7,
    max_tokens=4000,
    anthropic_api_key=os.environ["ANTHROPIC_API_KEY"]
)
```

**Decision**: Use `ChatAnthropic` with phase-specific temperature tuning:
- Ideate phase: 0.8 (creative exploration)
- Specify phase: 0.7 (balanced reasoning)
- Plan/Tasks phases: 0.3 (structured output)

**Rationale**: ChatAnthropic is the official LangChain integration for Claude. Temperature tuning optimizes output quality per phase characteristics. API key management reuses existing vTeam patterns (injected via environment variable).

**Error Handling Categories**:
1. **Authentication failures** (401) → Fail fast, report to CR status
2. **Rate limits** (429) → Exponential backoff (3 retries max)
3. **Timeouts** → No retry, report partial results

### Streaming Output Architecture

**Pattern**: Multi-mode streaming with LangGraph's `.astream()` and `.astream_events()`

```python
# Token-level streaming
async for event in graph.astream_events(state, version="v2"):
    if event["event"] == "on_llm_stream":
        token = event["data"]["chunk"].content
        await websocket_send({"type": "token", "data": token})
```

**Decision**: Implement two streaming levels:
1. **Token streaming** - Individual Claude tokens via `on_llm_stream` events
2. **Node streaming** - Phase transitions and tool executions via `on_chain_end` events

Map to WebSocket protocol:
- Tokens → `MESSAGE_PARTIAL` with fragment IDs
- Node completions → `AGENT_MESSAGE` with structured payloads

**Rationale**: Token-level streaming provides real-time UX (aligns with Claude Code behavior). Node-level streaming enables progress tracking in frontend. Dual streaming matches existing Claude Code runner capabilities.

### Multi-Step Workflow Orchestration

**Pattern**: Sequential graph execution with prerequisite validation

```python
async def run_rfe_workflow(phase: str, state: RFEState):
    # Prerequisite checks
    if phase == "specify" and not state["rfe_content"]:
        raise ValueError("Specify phase requires rfe.md artifact")

    # Execute phase graph
    if phase == "ideate":
        result = await ideate_graph.ainvoke(state)
    elif phase == "specify":
        result = await specify_graph.ainvoke(state)
    # ... etc

    return result
```

**Decision**: Entry point in LangGraphAdapter.run() will:
1. Detect requested phase from prompt or environment variable
2. Load prerequisite artifacts from workspace into state
3. Execute appropriate phase graph
4. Validate output artifact structure
5. Commit artifacts to Git and report completion

**Rationale**: Aligns with RFE Workflow's linear phase progression. Prerequisite validation prevents invalid executions (e.g., Plan phase without Spec). Artifact validation ensures interoperability with other vTeam tools.

---

## 3. Testing Strategy

### Contract Testing (Critical for FR-022)

**Objective**: Ensure LangGraphAdapter produces identical outputs to ClaudeCodeAdapter

**Approach**:
1. **Interface contract tests** - Verify LangGraphAdapter implements initialize/run/handle_message
2. **Artifact schema tests** - Validate RFE artifacts match expected markdown structure
3. **Protocol compliance tests** - Verify WebSocket messages match protocol definitions

**Test Coverage Target**: 100% (blocking release)

**Example Test**:
```python
@pytest.mark.asyncio
async def test_langgraph_adapter_interface():
    adapter = LangGraphAdapter()
    context = RunnerContext(session_id="test", workspace_path="/tmp/test")

    # Verify interface methods exist
    assert hasattr(adapter, 'initialize')
    assert hasattr(adapter, 'run')
    assert hasattr(adapter, 'handle_message')

    # Verify async signatures
    await adapter.initialize(context)
    result = await adapter.run()
    assert isinstance(result, dict)
```

### Integration Testing

**Objective**: Validate operator image selection, multi-repo support, WebSocket streaming

**Approach**:
1. **Operator tests** - Use Kind cluster to test AgenticSession CR → Job creation with correct image
2. **Multi-repo tests** - Verify 2-5 repositories clone successfully and are accessible
3. **Streaming tests** - Mock WebSocket and validate message sequencing and timing (<2s SLA)

**Test Coverage Target**: 80%

**Critical Scenarios**:
- `runnerType: langgraph` → Uses AMBIENT_LANGGRAPH_RUNNER_IMAGE
- `runnerType: claude-code` → Uses AMBIENT_CODE_RUNNER_IMAGE
- No `runnerType` specified → Defaults to Claude Code (backward compatibility)
- Invalid `runnerType` → Returns 400 Bad Request with descriptive error

### Regression Testing

**Objective**: Ensure 100% of existing Claude Code functionality remains unchanged (FR-022)

**Approach**:
1. **Baseline capture** - Run existing Claude Code test suite before any changes
2. **Post-implementation verification** - Re-run identical test suite and compare results
3. **Code review gate** - Verify zero modifications to claude-code-runner/ directory

**Test Coverage Target**: 100% (blocking release)

**Verification Checklist**:
- [ ] Claude Code runner source code unchanged (git diff)
- [ ] Claude Code session creation API unchanged
- [ ] Claude Code default behavior unchanged (no runnerType → Claude Code)
- [ ] Existing Claude Code sessions continue without modification

### API Validation Testing

**Objective**: Validate runnerType parameter handling in CreateAgenticSessionRequest

**Test Cases**:
1. Valid values: `claude-code`, `langgraph`
2. Invalid values: `invalid-runner`, `auto-gpt`, empty string
3. Missing field: Should default to `claude-code`
4. Case sensitivity: Should reject `ClaudeCode` or `LangGraph` (lowercase only)

**Test Coverage Target**: 100%

**Expected Error Format**:
```json
{
  "error": "Unsupported runner type 'invalid-runner'. Supported types: claude-code, langgraph",
  "code": 400
}
```

### End-to-End Testing

**Objective**: Validate complete session lifecycle for all 4 RFE phases

**Test Flow**:
1. Create session via API with `runnerType: langgraph`
2. Verify Job created with AMBIENT_LANGGRAPH_RUNNER_IMAGE
3. Monitor WebSocket for execution progress
4. Verify artifacts committed to correct Git branch
5. Verify CR status updated to Completed
6. Verify artifacts pass schema validation

**Test Coverage**: One E2E test per RFE phase (4 total)

**Success Criteria**:
- Session completes within timeout (2-10 minutes)
- All expected artifacts present in specs/{branchName}/
- WebSocket updates received within 2 seconds of actual progress
- CR status transitions: Pending → Running → Completed

---

## 4. Dependency Analysis

### Required Python Packages

```
langgraph==0.1.x
langchain==0.3.x
langchain-anthropic==0.2.x
anthropic==0.30.x
pydantic==2.x
websockets==14.x
```

**Decision**: Pin major versions to ensure stability. Use Poetry or pip-tools for dependency locking.

**Rationale**: LangGraph 0.1.x is current stable version. LangChain 0.3.x includes required streaming APIs. Pydantic 2.x provides TypedDict validation.

### Infrastructure Dependencies

- **Kubernetes 1.25+** - Already available in vTeam platform
- **PVC for workspace** - Reuses existing PVC provisioning logic
- **Git repository access** - Reuses existing SSH key / token mechanisms
- **Anthropic API access** - Requires valid API key (same as Claude Code runner)

**No new infrastructure required** - All dependencies already satisfied by vTeam platform.

---

## 5. Implementation Decisions Summary

| Area | Decision | Rationale |
|------|----------|-----------|
| **Adapter Interface** | Implement identical initialize/run/handle_message signature | Enables polymorphic runner selection without operator changes |
| **Image Selection** | Add AMBIENT_LANGGRAPH_RUNNER_IMAGE environment variable | Follows established configuration pattern |
| **State Management** | TypedDict-based RFEState shared across phase graphs | Clean phase boundaries with artifact accumulation |
| **Tool Integration** | @tool decorator with ToolNode orchestration | Automatic schema generation and error handling |
| **LLM Provider** | ChatAnthropic with phase-specific temperatures | Official integration with quality optimization |
| **Streaming** | Dual-level (token + node) via astream_events() | Matches Claude Code UX expectations |
| **Workflow Orchestration** | Four sequential graphs with prerequisite validation | Aligns with RFE Workflow structure |
| **Testing Strategy** | Contract (100%), Integration (80%), Regression (100%) | Ensures zero regression and validates new functionality |
| **Multi-Repo** | Reuse runner-shell workspace management | Avoids duplication, proven implementation |
| **Error Handling** | Categorized (auth, rate-limit, timeout) with specific strategies | Production-grade reliability |

---

## 6. Risk Mitigation

### High-Risk Areas

1. **Breaking Claude Code** (Likelihood: Low, Impact: Critical)
   - **Mitigation**: 100% regression test coverage, code review gate, separate container image
   - **Validation**: Zero changes to claude-code-runner/ directory

2. **Artifact Format Mismatch** (Likelihood: Medium, Impact: High)
   - **Mitigation**: Contract tests validating RFE artifact schemas, automated schema validation
   - **Validation**: Artifacts pass same validation as Claude Code outputs

3. **Operator Image Selection Bug** (Likelihood: Low, Impact: High)
   - **Mitigation**: Comprehensive integration tests with Kind cluster, explicit test cases for each runnerType value
   - **Validation**: Operator provisions correct image in 100% of test scenarios

4. **WebSocket Protocol Violation** (Likelihood: Medium, Impact: Medium)
   - **Mitigation**: Protocol compliance tests, message sequencing validation
   - **Validation**: Frontend displays LangGraph outputs identically to Claude Code

5. **LangGraph Streaming Failures** (Likelihood: Medium, Impact: Medium)
   - **Mitigation**: Fallback to batch mode if streaming fails, timeout protection on all operations
   - **Validation**: Streaming SLA met (<2s latency) in 95% of test runs

### Medium-Risk Areas

6. **Multi-Repo Complexity** (Likelihood: Low, Impact: Medium)
   - **Mitigation**: Reuse proven runner-shell workspace logic, explicit multi-repo test scenarios
   - **Validation**: 2-5 repo scenarios pass integration tests

7. **Session Continuation** (Likelihood: Low, Impact: Low)
   - **Mitigation**: Initial implementation may not support continuation (document limitation), post-MVP feature
   - **Validation**: System rejects cross-runner continuation with clear error message

---

## 7. Alternatives Considered

### Why LangGraph vs Other Frameworks?

**Alternatives Evaluated**:
1. **CrewAI** - Agent-focused framework
   - Rejected: Less mature ecosystem, higher complexity for workflow orchestration
2. **AutoGen** - Multi-agent conversation framework
   - Rejected: Overkill for single-agent RFE workflow, steeper learning curve
3. **Pure LangChain LCEL** - Using only LangChain Expression Language
   - Rejected: No built-in state management or checkpointing, manual graph construction
4. **Custom State Machine** - Build from scratch
   - Rejected: Reinventing wheel, no streaming infrastructure, maintenance burden

**LangGraph Selected Because**:
- Native state machine design aligns with RFE phase structure
- Built-in streaming and checkpointing
- Proven production usage in LangChain ecosystem
- Active development and community support
- Lower implementation complexity vs custom solution

### Why Separate Graphs vs Monolithic Graph?

**Decision**: Four separate graphs (ideate, specify, plan, tasks)

**Alternative**: Single graph with conditional routing
- Rejected because:
  - Testing complexity (need to mock all phases for any test)
  - Poor separation of concerns
  - Difficult to reason about state transitions
  - Hard to add new phases post-MVP

**Separate graphs provide**:
- Clean phase boundaries
- Independent testing
- Easier debugging
- Simpler onboarding for new developers

---

## 8. Next Steps for Phase 1: Design

With research complete, Phase 1 will produce:

1. **data-model.md** - Entity definitions for:
   - AgenticSession CR with `spec.runnerType` field
   - RunnerAdapter interface contract
   - RFEState TypedDict structure
   - LangGraph graph node definitions

2. **contracts/** - API contracts for:
   - CreateAgenticSessionRequest with runnerType parameter
   - WebSocket protocol message types
   - RFE artifact schemas (rfe.md, spec.md, plan.md, tasks.md)

3. **quickstart.md** - Developer guide covering:
   - How to create session with LangGraph runner
   - Environment variable configuration
   - Testing LangGraph runner locally
   - Troubleshooting common issues

4. **Agent context update** - Technology additions for AI agent awareness

---

## Research Sign-off

All "NEEDS CLARIFICATION" items from Technical Context are now resolved. Implementation can proceed to Phase 1: Design.

**Research Complete**: 2025-11-04
**Confidence Level**: High (based on comprehensive codebase analysis, official LangGraph documentation, and production testing patterns)
