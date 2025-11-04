# Feature Specification: LangGraph Runner Support

**Feature Branch**: `001-langgraph-runner-support`
**Created**: 2025-11-04
**Status**: Draft
**Input**: User description: "Add LangGraph runner support to Ambient Agentic Runner (vTeam)"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - LangGraph Session Execution (Priority: P1)

A development team wants to create an agentic session using LangGraph's structured workflow capabilities to execute an RFE Workflow phase.

**Why this priority**: This is the core MVP capability - enabling users to select and successfully execute LangGraph runner for any workflow task. Without this working, no other LangGraph features matter.

**Independent Test**: Can be fully tested by creating a session with `runnerType: langgraph`, providing a simple prompt, and verifying the session executes, produces output artifacts, and updates CR status. Delivers immediate value by providing an alternative execution framework.

**Acceptance Scenarios**:

1. **Given** a user has access to the vTeam platform and a valid repository, **When** they create a new agentic session via API with `runnerType: "langgraph"` and a prompt, **Then** the system provisions a LangGraph runner job, executes the prompt through LangGraph workflow, produces expected artifacts in specs/{branchName}/, and reports completion via WebSocket and CR status
2. **Given** a LangGraph session is executing, **When** the workflow processes the user's prompt, **Then** execution progress is streamed to the WebSocket in real-time and visible to the user
3. **Given** a LangGraph session completes successfully, **When** the user checks the output repository, **Then** workflow artifacts (rfe.md, spec.md, plan.md, or tasks.md depending on phase) are committed to the correct branch and directory structure
4. **Given** a LangGraph session encounters an error during execution, **When** the error occurs, **Then** the system reports the error details via WebSocket, updates CR status to failed, and logs diagnostic information

---

### User Story 2 - Runner Selection via UI and API (Priority: P1)

A user needs to choose between Claude Code and LangGraph execution frameworks when creating a new agentic session.

**Why this priority**: Critical for MVP - users must have a clear way to select the runner type. Without this, users cannot access the LangGraph functionality even if it's implemented.

**Independent Test**: Can be tested by verifying the UI presents a runner selection dropdown (Claude Code, LangGraph), API accepts `runnerType` parameter with validation, and sessions are created with the correct runner configuration. Delivers value by making the choice accessible to users.

**Acceptance Scenarios**:

1. **Given** a user is creating a new agentic session in the UI, **When** they view the session creation form, **Then** a "Runner Type" dropdown is displayed with options "Claude Code" (default) and "LangGraph"
2. **Given** a user submits a session creation API request with `runnerType: "langgraph"`, **When** the backend processes the request, **Then** the request is validated, the AgenticSession CR is created with `spec.runnerType: langgraph`, and appropriate LangGraph runner image is selected
3. **Given** a user submits a session creation request with `runnerType: "invalid-runner"`, **When** the backend validates the request, **Then** a 400 Bad Request error is returned with message "Unsupported runner type 'invalid-runner'. Supported types: claude-code, langgraph"
4. **Given** a user creates a session without specifying `runnerType`, **When** the backend processes the request, **Then** the system defaults to Claude Code runner maintaining backward compatibility

---

### User Story 3 - Claude Code Runner Unchanged (Priority: P1)

Existing users continue to use Claude Code runner exactly as before without any behavior changes.

**Why this priority**: Critical business constraint - ensures zero regression for existing users and workflows. Maintaining backward compatibility is essential for production stability and user trust.

**Independent Test**: Can be tested by running existing Claude Code sessions (created before LangGraph support) and new sessions with `runnerType: claude-code` or default, verifying they execute identically to previous behavior. Delivers value by protecting existing user workflows from disruption.

**Acceptance Scenarios**:

1. **Given** a user creates a new session without specifying `runnerType` or with `runnerType: "claude-code"`, **When** the session executes, **Then** the Claude Code runner is used with identical behavior to the pre-LangGraph implementation
2. **Given** an existing Claude Code session from before LangGraph support, **When** the user views or continues the session, **Then** all functionality works exactly as before with no code changes to claude-code-runner implementation
3. **Given** a user familiar with current Claude Code workflows, **When** they continue using the platform after LangGraph support is added, **Then** they experience zero changes in Claude Code behavior, UI interactions, or artifact outputs

---

### User Story 4 - Multi-Repo Support with LangGraph (Priority: P2)

A user working on a multi-repository project needs LangGraph runner to clone and work with multiple repositories simultaneously.

**Why this priority**: Important for production usage since many real-world projects span multiple repositories, but not required for basic functionality. Users can still benefit from LangGraph with single-repo projects.

**Independent Test**: Can be tested by creating a LangGraph session with `REPOS_JSON` environment variable containing multiple repository configurations, and verifying all repositories are cloned correctly and accessible during execution. Delivers value by supporting real-world multi-repo project structures.

**Acceptance Scenarios**:

1. **Given** a user configures a session with `runnerType: langgraph` and multiple repositories via `REPOS_JSON`, **When** the session initializes, **Then** all specified repositories are cloned to the workspace in the correct directory structure
2. **Given** a LangGraph runner is executing with multiple repositories available, **When** the workflow needs to access files from different repositories, **Then** all repositories are accessible and the workflow can read/write across them
3. **Given** a LangGraph session completes work across multiple repositories, **When** the session finalizes, **Then** changes are committed and pushed to the correct repositories based on configuration

---

### User Story 5 - RFE Workflow Phase Support (Priority: P2)

A product team uses LangGraph runner to progress through RFE Workflow phases (Ideate, Specify, Plan, Tasks, Implement) with proper artifact generation.

**Why this priority**: Important for alignment with existing vTeam workflows and ensuring LangGraph produces compatible outputs, but basic execution can work without full phase awareness. Can be refined post-MVP.

**Independent Test**: Can be tested by executing LangGraph sessions for each RFE Workflow phase and verifying the correct artifacts are produced in the expected formats and locations. Delivers value by ensuring LangGraph integrates seamlessly with existing RFE Workflow tooling.

**Acceptance Scenarios**:

1. **Given** a user creates a LangGraph session for the "Ideate" phase with interactive mode, **When** the session executes, **Then** the system runs in interactive mode, produces rfe.md in specs/{branchName}/ directory, and the artifact follows the expected RFE template structure
2. **Given** a user creates a LangGraph session for the "Specify" phase, **When** the session executes with an existing rfe.md, **Then** the system reads the rfe.md, produces spec.md in the same directory, and the spec follows the required specification format
3. **Given** a user creates a LangGraph session for the "Plan" phase, **When** the session executes with an existing spec.md, **Then** the system produces plan.md with detailed implementation plan matching the plan template structure
4. **Given** a user creates a LangGraph session for the "Tasks" phase, **When** the session executes with an existing plan.md, **Then** the system produces tasks.md with actionable task breakdown matching the tasks template format

---

### User Story 6 - Observability and Debugging (Priority: P3)

A developer troubleshooting a LangGraph session needs visibility into execution details, errors, and state transitions.

**Why this priority**: Valuable for production support and debugging, but basic error reporting covers most needs. Enhanced observability can be added based on operational experience.

**Independent Test**: Can be tested by reviewing WebSocket messages, CR status updates, and logs from LangGraph sessions to verify sufficient diagnostic information is captured. Delivers value by reducing troubleshooting time.

**Acceptance Scenarios**:

1. **Given** a LangGraph session is executing, **When** the user monitors the WebSocket connection, **Then** they receive real-time updates about graph node execution, state transitions, and progress indicators
2. **Given** a LangGraph session fails with an error, **When** the error occurs, **Then** the error message includes the failing graph node, error details, LangGraph-specific context, and is visible in both WebSocket stream and CR status
3. **Given** a support engineer investigates a failed LangGraph session, **When** they access the session logs, **Then** logs contain sufficient detail to diagnose the issue including LangGraph execution trace and state information

---

### Edge Cases

- What happens when a user tries to continue a Claude Code session with `runnerType: langgraph` specified? System should reject the request with a clear error message stating cross-runner continuation is not supported
- How does the system handle a LangGraph execution that exceeds the session timeout? System should terminate the session gracefully, update CR status to timeout/failed, and provide partial results if any artifacts were produced
- What happens when LangGraph runner cannot access the LLM provider API (network error, invalid credentials)? System should fail fast with authentication error, report via WebSocket and CR status, and avoid repeated failed attempts
- How does the system behave if LangGraph runner is selected but the LangGraph runner image is not available in the container registry? Operator should fail to create the Job and update CR status with image pull error
- What happens when a LangGraph session produces artifacts that don't match the expected RFE Workflow schema? System should validate artifact structure and either fail with validation errors or include warnings in the output
- How does the system handle a repository clone failure during LangGraph runner initialization? System should fail the session immediately with clear error about repository access, avoiding wasted execution time
- What happens when multiple sessions for the same project/workspace try to use different runner types? System enforces project-level runner consistency (all sessions in a project use the same runner type once established)

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST extend the `CreateAgenticSessionRequest` API to accept an optional `runnerType` field with allowed values "claude-code" (default) or "langgraph"
- **FR-002**: System MUST validate the `runnerType` parameter and reject requests with invalid values by returning a 400 Bad Request error with a descriptive message
- **FR-003**: Backend MUST create AgenticSession Custom Resource (CR) with `spec.runnerType` field populated from the API request
- **FR-004**: Operator MUST select the appropriate container image based on the `runnerType` value in the AgenticSession CR spec
- **FR-005**: Operator MUST use the `AMBIENT_LANGGRAPH_RUNNER_IMAGE` environment variable to determine which container image to use for LangGraph runner jobs
- **FR-006**: System MUST default to Claude Code runner when `runnerType` is not specified in the API request, maintaining backward compatibility
- **FR-007**: LangGraph runner MUST implement a `LangGraphAdapter` class with methods `initialize()`, `run()`, and `handle_message()` conforming to the same interface as `ClaudeCodeAdapter`
- **FR-008**: LangGraph runner MUST clone all repositories specified in the `REPOS_JSON` environment variable to the workspace directory structure
- **FR-009**: LangGraph runner MUST execute user prompts through a LangGraph-based state machine workflow
- **FR-010**: LangGraph runner MUST stream execution progress and agent outputs to the WebSocket connection in real-time
- **FR-011**: LangGraph runner MUST produce RFE Workflow artifacts (rfe.md, spec.md, plan.md, tasks.md) in the specs/{branchName}/ directory matching the same format and structure as Claude Code runner
- **FR-012**: LangGraph runner MUST commit and push completed artifacts to the configured output repository using Git
- **FR-013**: LangGraph runner MUST update the AgenticSession CR status field with completion state (completed/failed) and relevant metadata
- **FR-014**: LangGraph runner MUST integrate with Anthropic Claude models as the initial LLM provider
- **FR-015**: LangGraph runner MUST support both interactive and automated execution modes based on session configuration
- **FR-016**: System MUST prevent cross-runner session continuation (cannot start with Claude Code and resume with LangGraph or vice versa)
- **FR-017**: System MUST enforce project-level runner consistency where all sessions within a project/workspace use the same runner type
- **FR-018**: LangGraph runner MUST handle authentication failures with LLM provider by failing fast and reporting clear error messages
- **FR-019**: LangGraph runner MUST handle repository access errors during initialization by failing the session immediately with descriptive errors
- **FR-020**: LangGraph runner MUST gracefully handle execution timeouts by terminating the workflow and providing partial results if available
- **FR-021**: Frontend UI MUST display a "Runner Type" dropdown in the session creation form with options "Claude Code" and "LangGraph"
- **FR-022**: Claude Code runner implementation MUST remain completely unchanged with zero code modifications
- **FR-023**: System MUST generate and expose documentation describing when to choose Claude Code vs LangGraph and how to create sessions with each runner type
- **FR-024**: LangGraph runner Docker image MUST build successfully in CI/CD pipeline and be published to the container registry

### Key Entities

- **AgenticSession CR**: Kubernetes Custom Resource representing a single agentic session execution; includes `spec.runnerType` field determining which runner implementation to use
- **Runner Adapter**: Interface abstraction implemented by both ClaudeCodeAdapter and LangGraphAdapter; defines common methods (initialize, run, handle_message) enabling polymorphic runner selection
- **Workspace**: Persistent volume claim (PVC) containing cloned repositories, execution artifacts, and session state; shared across session continuations
- **RFE Workflow Artifacts**: Structured markdown files (rfe.md, spec.md, plan.md, tasks.md) produced by runners following standardized templates; enables tool interoperability regardless of runner type
- **LangGraph State Machine**: Graph-based workflow orchestrating LLM calls, tool usage, and state transitions; specific to LangGraph runner implementation

## Assumptions *(optional)*

- An existing LangGraph-based Runnable Agent implementation is available that supports RFE Workflow phases
- The platform currently uses Kubernetes-based orchestration with Custom Resources (CRs) for session management
- The existing runner-shell framework provides Git integration, WebSocket streaming, and multi-repo support at a shared abstraction level
- The Claude Code runner uses an adapter pattern that can be replicated for LangGraph
- Users understand the conceptual difference between interactive development tools and structured workflow orchestration
- The platform has established RFE Workflow artifact templates (rfe.md, spec.md, plan.md, tasks.md) with defined schemas
- Standard session timeout values are already configured and can be reused for LangGraph sessions
- Container registry infrastructure exists for publishing runner images
- CI/CD pipeline can be extended to build additional container images

## Dependencies *(optional)*

- **External**: LangGraph Python framework and its dependencies (LangChain, required Python packages)
- **External**: Anthropic API access and valid API keys for Claude model usage
- **Internal**: Existing runner-shell framework providing base abstractions
- **Internal**: Kubernetes operator implementation that provisions Jobs from AgenticSession CRs
- **Internal**: Frontend UI codebase for adding runner selection dropdown
- **Internal**: Backend API service for validating and processing CreateAgenticSessionRequest
- **Internal**: Persistent volume claim (PVC) infrastructure for workspace storage
- **Internal**: Git repository access mechanisms (SSH keys, tokens) already configured
- **Internal**: WebSocket streaming infrastructure for real-time updates
- **Internal**: RFE Workflow templates stored in .specify/ directory structure

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Users can successfully create and execute LangGraph sessions that complete from prompt to artifact production within typical timeout periods (2-10 minutes depending on complexity)
- **SC-002**: LangGraph runner produces RFE Workflow artifacts (rfe.md, spec.md, plan.md, tasks.md) that are structurally identical and interchangeable with Claude Code runner outputs, validated by automated schema checks
- **SC-003**: 100% of existing Claude Code sessions and workflows continue functioning with zero regression after LangGraph support is deployed
- **SC-004**: LangGraph runner successfully handles multi-repo configurations with 2-5 repositories, cloning and accessing all repositories correctly
- **SC-005**: Users receive real-time execution feedback via WebSocket with updates appearing within 2 seconds of actual workflow progress
- **SC-006**: API validation catches and rejects invalid runner types with appropriate error responses in 100% of test cases
- **SC-007**: LangGraph runner Docker image builds successfully in CI pipeline with build time under 10 minutes
- **SC-008**: Documentation enables a developer unfamiliar with the codebase to understand runner selection, create sessions with both runner types, and troubleshoot common issues within 30 minutes of reading
- **SC-009**: 95% of LangGraph session errors are reported with sufficient detail (error type, context, failing component) to enable diagnosis without additional logging
- **SC-010**: Operator correctly routes session creation requests to appropriate runner images based on `runnerType` field in 100% of cases

