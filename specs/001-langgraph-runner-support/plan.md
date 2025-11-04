# Implementation Plan: LangGraph Runner Support

**Branch**: `001-langgraph-runner-support` | **Date**: 2025-11-04 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-langgraph-runner-support/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

Add LangGraph runner support to Ambient Agentic Runner (vTeam) platform as an alternative execution framework to Claude Code. The implementation will extend the API and operator to accept a `runnerType` parameter, implement a LangGraphAdapter following the same interface pattern as ClaudeCodeAdapter, and ensure zero regression to existing Claude Code functionality. The LangGraph runner will execute user prompts through LangGraph workflows while producing identical RFE Workflow artifacts (rfe.md, spec.md, plan.md, tasks.md) and maintaining full compatibility with existing multi-repo, WebSocket streaming, and Git integration features.

## Technical Context

**Language/Version**: Python 3.11 (matches existing runner-shell framework)
**Primary Dependencies**: LangGraph, LangChain, Anthropic SDK (Claude API client), existing runner-shell abstractions
**Storage**: Git repositories (primary), Kubernetes PVCs for workspace persistence, AgenticSession CR status in etcd
**Testing**: pytest (Python unit/integration tests), Kubernetes integration tests for operator behavior
**Target Platform**: Kubernetes (containerized runners as Jobs), Linux containers
**Project Type**: Web/Distributed System (backend API + Kubernetes operator + containerized runners + frontend UI)
**Performance Goals**: Session completion within 2-10 minutes for typical workflows, WebSocket updates within 2 seconds of progress
**Constraints**: Zero regression for Claude Code runner (no code changes), backward compatibility (default to Claude Code when runnerType unspecified), runner consistency within project scope
**Scale/Scope**: Multi-repo support (2-5 repos), concurrent session execution, RFE Workflow phase support (5 phases), two runner implementations

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### Initial Check (Before Phase 0)

**Status**: Constitution file is not yet populated (template only). Proceeding with standard software engineering principles:

- **Code Reuse**: Leveraging existing runner-shell abstractions, adapter pattern already established by ClaudeCodeAdapter
- **Backward Compatibility**: Enforced by FR-006 (default to Claude Code), FR-022 (zero changes to Claude Code)
- **Isolation**: LangGraph implementation completely separate from Claude Code runner (separate container image, adapter class)
- **Testing**: Integration testing required per FR-024 (CI/CD pipeline validation), acceptance scenarios defined in spec
- **Interface Contracts**: RunnerAdapter interface ensures polymorphic behavior across runner types

No violations identified. Standard extension pattern applied.

### Post-Design Re-evaluation (After Phase 1)

**Status**: PASS ✓

After completing Phase 1 design (data-model.md, contracts/, quickstart.md), the implementation maintains compliance with software engineering best practices:

#### Architecture Compliance

✅ **Separation of Concerns**: LangGraph runner is completely isolated from Claude Code runner:
- Separate container image (`AMBIENT_LANGGRAPH_RUNNER_IMAGE`)
- Separate adapter class (`LangGraphAdapter`)
- Separate directory structure (`runners/langgraph-runner/`)
- Zero code changes to Claude Code runner (FR-022 validated)

✅ **Interface Contracts**: RunnerAdapter abstract interface enforces consistent behavior:
- Three required methods: `initialize()`, `run()`, `handle_message()`
- Both adapters implement identical interface signatures
- Polymorphic runner selection via operator image selection logic

✅ **Backward Compatibility**: Multiple safeguards ensure zero regression:
- Default `runnerType: "claude-code"` when not specified (FR-006)
- API validation rejects invalid runner types with clear error messages
- Claude Code runner code remains completely unchanged
- Existing sessions continue without modification

✅ **Code Reuse**: Maximizes reuse of existing infrastructure:
- runner-shell framework (WebSocket, workspace management, Git integration) reused by both runners
- Multi-repo support logic shared
- Protocol definitions shared (message types, sequencing)
- Operator provisioning logic extended (not duplicated)

✅ **Testability**: Comprehensive testing strategy defined:
- Contract tests ensure interface compliance (100% coverage)
- Integration tests validate operator behavior (80% coverage)
- Regression tests protect Claude Code (100% coverage)
- End-to-end tests verify complete session lifecycle (4 phases)

✅ **Observability**: Design includes monitoring and debugging support:
- WebSocket protocol documented with all message types
- System messages for diagnostics and troubleshooting
- CR status updates for lifecycle tracking
- Structured logging patterns defined

#### Data Model Compliance

✅ **Minimal Changes**: Only additive modifications to existing schemas:
- 1 new field: `spec.runnerType` in AgenticSession CR (optional, defaults to "claude-code")
- 1 new config: `AMBIENT_LANGGRAPH_RUNNER_IMAGE` environment variable
- 0 breaking changes to existing entities

✅ **Schema Validation**: All entities have clear validation rules:
- OpenAPI schema for AgenticSession CR with enum constraints
- API request validation in backend (400 errors for invalid values)
- Artifact schema validation using RFE templates

✅ **Backward Compatibility**: All modifications are non-breaking:
- Existing CRs without `runnerType` field continue working (default behavior)
- Existing API clients without `runnerType` parameter continue working (default behavior)
- No changes to artifact formats or directory structures

#### Integration Patterns Compliance

✅ **Configuration Management**: Follows established patterns:
- Environment variables for image configuration (matches existing `AMBIENT_CODE_RUNNER_IMAGE`)
- CR-driven resource creation (matches existing operator pattern)
- Secret-based authentication (matches existing token management)

✅ **Error Handling**: Robust error handling strategy:
- Authentication failures fail fast with clear messages
- Rate limits trigger exponential backoff (3 retries max)
- Timeouts reported with partial results when possible
- All error scenarios documented in contracts

#### Documentation Compliance

✅ **Comprehensive Documentation**: All artifacts produced:
- research.md: Decisions, rationale, alternatives considered
- data-model.md: Entity definitions with relationships
- contracts/: API, CR schema, WebSocket protocol
- quickstart.md: User-facing guide with examples and troubleshooting

### Conclusion

**GATE STATUS**: ✅ PASS

The design maintains full compliance with architectural principles:
- Zero regression risk (Claude Code unchanged)
- Clean separation of concerns (isolated implementation)
- Maximum code reuse (shared infrastructure)
- Comprehensive testing strategy (contract, integration, regression)
- Clear documentation (all artifacts complete)

**No complexity violations identified.** Implementation can proceed to Phase 2 (tasks generation).

## Project Structure

### Documentation (this feature)

```
specs/001-langgraph-runner-support/
├── spec.md              # Feature specification (already created)
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)

**Note**: The actual vTeam repository structure is located at `/workspace/sessions/agentic-session-1762293354/workspace/vTeam`. This plan documents the expected structure based on the feature requirements.

```
# Backend API (handles CreateAgenticSessionRequest)
backend/
├── src/
│   ├── models/
│   │   └── agentic_session.py      # API models with runnerType field
│   ├── services/
│   │   └── session_service.py      # Validation logic for runnerType
│   └── api/
│       └── session_endpoints.py    # CreateAgenticSessionRequest handler
└── tests/
    ├── unit/
    └── integration/

# Kubernetes Operator (provisions Jobs from AgenticSession CRs)
operator/
├── src/
│   ├── controllers/
│   │   └── agentic_session_controller.py  # Reads spec.runnerType, selects image
│   └── config/
│       └── images.py                      # AMBIENT_LANGGRAPH_RUNNER_IMAGE config
└── tests/
    └── integration/

# LangGraph Runner (new implementation)
runners/
├── langgraph-runner/
│   ├── src/
│   │   ├── adapters/
│   │   │   └── langgraph_adapter.py      # LangGraphAdapter class
│   │   ├── workflows/
│   │   │   └── rfe_workflow.py           # LangGraph state machine
│   │   ├── tools/
│   │   │   └── rfe_tools.py              # Tool implementations
│   │   └── main.py                        # Entry point
│   ├── tests/
│   │   ├── unit/
│   │   └── integration/
│   ├── Dockerfile                         # Container image build
│   └── requirements.txt                   # LangGraph, LangChain, Anthropic SDK
│
└── claude-code-runner/                    # UNCHANGED (FR-022)
    └── [existing structure preserved]

# Frontend UI (runner selection dropdown)
frontend/
├── src/
│   ├── components/
│   │   └── SessionCreationForm.tsx        # Add runnerType dropdown
│   └── services/
│       └── sessionApi.ts                  # Pass runnerType to backend
└── tests/

# Shared Runner Shell (already exists, reused by both runners)
runner-shell/
├── src/
│   ├── git_integration.py                 # Shared Git operations
│   ├── websocket_client.py                # Shared WebSocket streaming
│   └── workspace_manager.py               # Multi-repo cloning, PVC management
└── tests/
```

**Structure Decision**: Web/Distributed System architecture selected. The codebase spans backend API (session validation), Kubernetes operator (Job provisioning), containerized runners (execution), and frontend UI (user interaction). LangGraph runner is added as a sibling to claude-code-runner under runners/ directory, sharing common abstractions from runner-shell/. This structure isolates the new runner implementation while reusing proven infrastructure components.

## Complexity Tracking

*Fill ONLY if Constitution Check has violations that must be justified*

No violations identified. Standard architectural extension with proper isolation.
