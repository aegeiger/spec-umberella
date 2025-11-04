# Add LangGraph Runner Support to Ambient Agentic Runner (vTeam)

**Feature Overview:**

Enable the Ambient Agentic Runner platform (vTeam) to support multiple AI execution frameworks by adding a LangGraph runner option alongside the existing Claude Code with SpecKit runner. This enhancement provides an alternative execution path for agentic sessions without modifying the current Claude Code runner in any way. The LangGraph runner will support the same RFE Workflow phases (Ideate, Specify, Plan, Tasks, Implement, Review, Completed) using a LangGraph-based Runnable Agent that follows the same workflow specifications and produces the same artifacts (rfe.md, spec.md, plan.md, tasks.md), allowing users to choose between Claude Code's interactive development environment or LangGraph's graph-based orchestration capabilities based on their specific use case needs.

**Goals:**

* Enable runtime selection between Claude Code and LangGraph execution frameworks when creating agentic sessions
* Add LangGraph runner as a completely separate implementation path without any modifications to the existing Claude Code runner
* Maintain full feature parity with the existing Claude Code runner for core capabilities (WebSocket streaming, multi-repo support, session continuation, Git integration)
* Support the existing RFE Workflow phases using a LangGraph-based Runnable Agent that produces the same workflow artifacts (rfe.md, spec.md, plan.md, tasks.md)
* Provide a clean abstraction layer that makes adding future runner types straightforward
* Ensure existing Claude Code sessions and workflows continue to work without any modification
* Allow users to leverage LangGraph's strengths in workflow orchestration, state management, and complex multi-agent coordination
* Support projects that benefit from LangGraph's graph-based execution model with explicit state transitions and control flow

**Target Users:**
* Development teams needing structured workflow automation with explicit state management
* Organizations building complex multi-agent systems with conditional branching and parallel execution
* Users requiring deterministic, auditable agent workflows with clear execution paths
* Teams familiar with LangChain/LangGraph ecosystems who want to leverage existing LangGraph tools and components

**Out of Scope:**

* **ANY modifications to the existing Claude Code runner implementation** - the Claude Code runner remains completely untouched
* Support for runner types other than Claude Code and LangGraph in this initial implementation (but architecture should be extensible)
* **Mixed runner usage within the same project/workspace** - once a project chooses a runner type, all sessions in that project/workspace use the same runner
* Migration tools to convert Claude Code sessions to LangGraph sessions or vice versa
* Cross-runner session continuation (cannot start with Claude Code and resume with LangGraph)
* Real-time switching between runners within a single session or project
* Support for LangGraph Studio or other third-party LangGraph development tools
* Custom LangGraph graph definitions uploaded by users (initial implementation will use predefined graphs aligned with RFE Workflow phases)
* LangGraph Cloud integration or hosted LangGraph services

**Requirements:**

**MVP Requirements:**

1. **Runner Selection API** (MVP)
   - Extend `CreateAgenticSessionRequest` to include optional `runnerType` field (default: "claude-code", options: "claude-code", "langgraph")
   - Backend validates runner type and rejects invalid values
   - Frontend provides runner selection UI in session creation flow

2. **LangGraph Runner Implementation** (MVP)
   - Create new `langgraph-runner` component directory following the same structure as `claude-code-runner`
   - Implement `LangGraphAdapter` class conforming to the same interface as `ClaudeCodeAdapter` (`initialize()`, `run()`, `handle_message()`)
   - Support core capabilities: workspace management, Git clone/push, WebSocket streaming, multi-repo support
   - Integrate existing LangGraph-based Runnable Agent that follows RFE Workflow specifications
   - Support RFE Workflow phases: Ideate (interactive), Specify, Plan, Tasks, Implement with appropriate graph structures
   - Produce workflow artifacts compatible with RFE Workflow: rfe.md, spec.md, plan.md, tasks.md in specs/{branchName}/ directory
   - Integrate LangGraph framework with appropriate LLM provider configuration (Anthropic Claude models initially)

3. **Operator Runner Selection** (MVP)
   - Modify operator to select appropriate container image based on `runnerType` in CR spec
   - Support new configuration: `AMBIENT_LANGGRAPH_RUNNER_IMAGE` environment variable
   - Maintain backward compatibility: default to Claude Code if `runnerType` not specified
   - Ensure operator changes do not affect Claude Code runner job creation logic in any way

4. **RFE Workflow Support via LangGraph** (MVP)
   - Integrate existing LangGraph-based Runnable Agent that implements RFE Workflow phases
   - Support phase-appropriate graph execution: interactive for Ideate, automated for Specify/Plan/Tasks
   - Ensure LangGraph runner produces artifacts in the same locations and formats as Claude Code runner (specs/{branchName}/rfe.md, spec.md, plan.md, tasks.md)
   - Support prompt execution through LangGraph state machine with workflow phase awareness
   - Capture and stream LangGraph execution logs and agent outputs to WebSocket

5. **Documentation** (MVP)
   - Create `LANGGRAPH_RUNNER.md` documentation similar to existing `CLAUDE_CODE_RUNNER.md`
   - Update platform README with runner selection guidance
   - Document differences between Claude Code and LangGraph execution models

**Post-MVP Requirements:**

6. **Session Continuation Support** (Post-MVP)
   - Enable LangGraph sessions to be resumed using `PARENT_SESSION_ID`
   - Persist LangGraph checkpointing data alongside workspace in PVC
   - Support rehydrating graph state from previous session

7. **Advanced LangGraph Features** (Post-MVP)
   - Support custom graph definitions via configuration files in workspace
   - Enable LangGraph's human-in-the-loop capabilities for interactive sessions
   - Implement LangGraph's streaming and callback mechanisms for fine-grained progress updates

8. **Enhanced RFE Workflow Integration** (Post-MVP)
   - Support workflow phase transitions and validations within LangGraph graphs
   - Implement prerequisite checking (spec.md for Plan phase, plan.md for Tasks phase, etc.)
   - Create LangGraph graph definitions optimized for each RFE Workflow phase

9. **Multi-Agent Graph Support** (Post-MVP)
   - Leverage vTeam's existing agent personas (Emma, Stella, Ryan, etc.) as LangGraph nodes
   - Create graph templates for common multi-agent collaboration patterns
   - Support agent selection configurations (BALANCED, COMPREHENSIVE) within LangGraph execution

10. **Runner Performance Metrics** (Post-MVP)
    - Track and expose metrics per runner type (execution time, token usage, success rate)
    - Enable A/B testing between Claude Code and LangGraph for the same prompts

**Done - Acceptance Criteria:**

The feature is considered complete when:

1. **User Can Select Runner:** Users can specify `"runnerType": "langgraph"` in the API request or select "LangGraph" from a dropdown in the UI when creating a new agentic session
2. **LangGraph Sessions Execute:** A session created with `runnerType: langgraph` successfully:
   - Clones the specified repository(ies) to workspace
   - Executes the provided prompt through a LangGraph workflow
   - Produces RFE Workflow artifacts (rfe.md, spec.md, plan.md, tasks.md) in the same format and location as Claude Code runner
   - Streams execution updates to the WebSocket
   - Commits and pushes results to the output repository
   - Updates the CR status with completion/failure state
3. **Claude Code Runner Completely Unaffected:** The Claude Code runner code, configuration, and behavior remains 100% unchanged. Existing Claude Code sessions and new sessions with `runnerType: claude-code` (or default/omitted) continue to work exactly as before with zero modifications to the claude-code-runner implementation
4. **Multi-Repo Support:** LangGraph runner successfully handles multi-repo configurations via `REPOS_JSON` environment variable
5. **Error Handling:** LangGraph runner gracefully handles common errors (authentication failures, missing repos, LangGraph execution errors) and reports them via WebSocket and CR status
6. **Documentation Complete:** A developer can read the documentation and understand:
   - When to choose Claude Code vs LangGraph
   - How to create a session with each runner type
   - The architecture and extension points for adding new runners
7. **CI/CD Integration:** LangGraph runner Docker image builds successfully in CI pipeline and is published to the container registry

**Use Cases - i.e. User Experience & Workflow:**

**Use Case 1: RFE Workflow with LangGraph Runner**

*Scenario:* A product team wants to use LangGraph's structured workflow capabilities to manage an RFE Workflow from ideation through task breakdown.

1. User creates an RFE Workflow in vTeam UI with umbrella repo: `https://github.com/org/specs-repo`
2. System seeds the repository with .claude/, .specify/, and specs/{branchName}/ directories
3. User starts "Ideate" phase session:
   - Selects "LangGraph" from "Runner Type" dropdown
   - Interactive mode: true
   - Prompt: "Create an RFE for adding real-time metrics to the dashboard"
4. Backend creates `AgenticSession` CR with `spec.runnerType: langgraph` and labels `rfe-phase: ideate`
5. Operator provisions Job with LangGraph runner image
6. LangGraph runner executes interactive graph:
   - Clones umbrella repo to workspace
   - Runs LangGraph-based Runnable Agent in interactive mode
   - Produces `specs/ambient-metrics-dashboard/rfe.md`
   - Commits and pushes to feature branch
7. User starts "Specify" phase session with LangGraph runner (automated):
   - Reads rfe.md from workspace
   - Executes LangGraph Specify graph
   - Produces `specs/ambient-metrics-dashboard/spec.md`
8. User continues through Plan and Tasks phases using LangGraph runner, each producing the expected artifacts

**Use Case 2: Developer Uses Claude Code Runner (No Changes)**

*Scenario:* A developer wants to use Claude Code's interactive capabilities for the same RFE Workflow.

1. User navigates to existing RFE Workflow
2. User starts "Ideate" phase session
3. User selects "Claude Code" from "Runner Type" dropdown (or leaves as default)
4. User specifies:
   - Prompt: "Create an RFE for adding authentication features"
   - Interactive: `true`
5. User clicks "Start Session"
6. Backend creates `AgenticSession` CR with `spec.runnerType: claude-code` (or omitted)
7. Operator provisions Job with Claude Code runner image **using existing unchanged implementation**
8. Claude Code runner executes exactly as it does today:
   - Uses /speckit commands or interactive prompts
   - Supports SpecKit workflow phases
   - Produces rfe.md, spec.md, plan.md, tasks.md
9. User continues through workflow phases using Claude Code runner with zero differences from current behavior

**Use Case 3: Different Runner Choices Across Projects**

*Scenario:* A team uses different runners for different projects based on project characteristics.

1. **Project A: Authentication Feature (Claude Code Runner)**
   - User creates RFE Workflow in Project A workspace: `https://github.com/org/auth-specs`
   - All sessions in this project use Claude Code runner
   - User progresses through Ideate → Specify → Plan → Tasks phases with Claude Code
   - Produces: `specs/ambient-auth-feature/rfe.md`, `spec.md`, `plan.md`, `tasks.md`

2. **Project B: Analytics Pipeline (LangGraph Runner)**
   - User creates separate RFE Workflow in Project B workspace: `https://github.com/org/analytics-specs`
   - All sessions in this project use LangGraph runner
   - User progresses through Ideate → Specify → Plan → Tasks phases with LangGraph
   - Produces: `specs/ambient-analytics-pipeline/rfe.md`, `spec.md`, `plan.md`, `tasks.md`

3. Both projects use the same RFE Workflow structure and produce compatible artifacts, but each project/workspace consistently uses one runner type throughout all phases

**Use Case 4: Session Continuation (Post-MVP)**

*Scenario:* User wants to continue a previous LangGraph session with additional instructions.

1. User views completed session `session-abc123` in UI
2. User clicks "Continue Session" button
3. UI pre-fills form with:
   - Runner Type: `langgraph` (inherited from parent)
   - Parent Session ID: `session-abc123`
4. User adds new prompt: "Now generate visualizations for the report"
5. New session is created with same runner type and workspace is reused
6. LangGraph runner loads checkpointed state and continues from where it left off

**Alternative Flow: Unsupported Runner Type**

1. User submits API request with `"runnerType": "invalid-runner"`
2. Backend validates and returns `400 Bad Request` with error message: "Unsupported runner type 'invalid-runner'. Supported types: claude-code, langgraph"
3. User corrects request and resubmits

**Documentation Considerations:**

1. **New Documentation Required:**
   - `LANGGRAPH_RUNNER.md`: Comprehensive guide to the LangGraph runner (similar to existing `CLAUDE_CODE_RUNNER.md`)
     - Architecture and components
     - Environment variables
     - Graph definitions and customization
     - Comparison with Claude Code runner
   - Update `README.md` with runner selection section
   - API documentation update for `runnerType` field in `CreateAgenticSessionRequest`

2. **Extended Documentation:**
   - **When to Choose Which Runner:**
     - Claude Code: Interactive development, code editing, SpecKit workflows, exploratory tasks
     - LangGraph: Structured workflows, deterministic execution, complex state management, batch processing
   - **Architecture Documentation:**
     - Runner abstraction layer and adapter pattern
     - How to add new runner types in the future
     - Runner lifecycle and state management
   - **Migration Guide:**
     - How to adapt existing Claude Code workflows for LangGraph (when custom graphs are supported)
     - Limitations and trade-offs between runners

3. **Existing Documentation Updates:**
   - `CLAUDE_CODE_RUNNER.md`: Add section clarifying this is one of multiple supported runners
   - Operator README: Document new `AMBIENT_LANGGRAPH_RUNNER_IMAGE` configuration
   - Deployment guides: Include LangGraph runner image in deployment manifests

4. **Reference Documentation:**
   - Link to LangGraph official documentation for users wanting to understand graph concepts
   - Link to LangChain documentation for LLM integration patterns
   - Example LangGraph graph definitions for common use cases

**Questions to answer:**

1. **LangGraph Integration Details:**
   - How is the existing LangGraph-based Runnable Agent structured to support RFE Workflow phases?
   - What are the specific inputs and outputs expected by the LangGraph agent for each phase (Ideate, Specify, Plan, Tasks)?
   - How does the LangGraph agent handle the transition between interactive (Ideate) and automated (Specify/Plan/Tasks) modes?
   - What dependencies or environment configuration does the LangGraph agent require?

2. **State Management:**
   - Where should LangGraph checkpoint data be stored? (In PVC alongside workspace, or separate volume?)
   - How do we handle session continuation with LangGraph's checkpointing mechanism?
   - Should LangGraph state be exposed via API for external inspection?

3. **LLM Provider Integration:**
   - Should LangGraph runner support only Anthropic Claude models initially, or multiple providers?
   - How do we handle API key management for different LLM providers?
   - Should we reuse `llmSettings` from `CreateAgenticSessionRequest` or introduce LangGraph-specific configuration?

4. **Tool and Integration Support:**
   - Should LangGraph runner support the same MCP servers as Claude Code runner?
   - How do we handle LangGraph's tool/function calling compared to Claude SDK's tool system?
   - Should we expose LangGraph's built-in tools (Tavily search, etc.) or restrict to our controlled set?

5. **Performance and Resource Management:**
   - What are the resource requirements (CPU, memory) for LangGraph runner compared to Claude Code?
   - Should we impose different timeout defaults for LangGraph sessions?
   - How do we handle long-running LangGraph workflows that exceed typical session timeouts?

6. **Interactive Sessions:**
   - How should interactive mode work with LangGraph? (Pause at certain nodes? Human-in-the-loop?)
   - Should LangGraph support the same WebSocket message protocol for user input during execution?

7. **RFE Workflow Artifact Compatibility:**
   - How do we ensure LangGraph runner produces artifacts (rfe.md, spec.md, plan.md, tasks.md) with the exact same schema and structure as Claude Code runner?
   - Should LangGraph runner validate artifact format against templates/schemas before committing?
   - How do we handle prerequisite validation (spec.md exists before Plan, plan.md exists before Tasks) in LangGraph runner?
   - Should the LangGraph runner expose the same phase-based environment variables (WORKFLOW_PHASE, PARENT_RFE) as Claude Code runner?

8. **Error Handling and Observability:**
   - How do we capture and report LangGraph-specific errors (node failures, state validation errors)?
   - Should we expose LangGraph's execution trace for debugging?
   - How do we handle partial failures in LangGraph graphs (continue, rollback, or fail entire session)?

9. **Versioning and Compatibility:**
   - How do we handle LangGraph version updates that change graph API?
   - Should we support multiple LangGraph versions simultaneously?
   - How do we ensure LangGraph runner changes don't break existing sessions?

10. **Security and Isolation:**
    - Are there additional security considerations for LangGraph runner compared to Claude Code?
    - How do we sandbox LangGraph's execution environment?
    - Should we restrict certain LangGraph features (arbitrary code execution, external API calls)?

**Background & Strategic Fit:**

The Ambient Agentic Runner platform currently provides a single execution model: Claude Code with SpecKit. This approach excels at interactive development, code editing, and guided software engineering workflows. However, as the platform evolves to support broader automation use cases, users need more execution options.

**Market Context:**
- LangGraph has emerged as a leading framework for building stateful, multi-agent applications with LLMs
- LangGraph's graph-based model provides explicit control flow, state management, and deterministic execution
- Many organizations are investing in LangChain/LangGraph ecosystems and want to leverage existing skills and tools
- Competing platforms (Langflow, Flowise, n8n) offer visual workflow builders with LangGraph integration

**Strategic Benefits:**
1. **Differentiation:** Positions vTeam as a multi-framework platform rather than a Claude Code-only tool
2. **Flexibility:** Allows users to choose the best execution model for their use case
3. **Future-Proofing:** Establishes architecture patterns for adding more runners (OpenAI Assistants, AutoGPT, etc.)
4. **Ecosystem Leverage:** Taps into the growing LangGraph community and tooling ecosystem
5. **Use Case Expansion:** Enables structured workflows, batch processing, and deterministic automation alongside interactive development

**Technical Context:**
- Current architecture (runner-shell framework, adapter pattern) was designed with multi-runner support in mind
- WebSocket streaming protocol is runner-agnostic and can carry LangGraph events
- Kubernetes-based orchestration naturally supports different container images per runner type
- Multi-repo support and Git integration are implemented at the runner-shell level, so LangGraph gets them "for free"

**Related Initiatives:**
- vTeam agent personas (Emma, Stella, Ryan, etc.) could be represented as LangGraph nodes in future iterations
- RFE Workflow phases are already supported by the existing LangGraph-based Runnable Agent, providing immediate compatibility
- The LangGraph runner will leverage the same seeded repository structure (.claude/, .specify/, specs/) as Claude Code runner
- Future runners (CrewAI, AutoGen) could follow the same pattern established by this LangGraph implementation
- Teams can choose which runner type best fits each project's needs, with consistent RFE Workflow artifacts across all runner types

**Customer Considerations:**

**Migration and Learning Curve:**
- Customers familiar with Claude Code's interactive model need education on when to use LangGraph
- Clear documentation and examples are critical to prevent confusion
- Consider providing starter templates for common LangGraph workflow patterns

**Performance Expectations:**
- LangGraph workflows may have different performance characteristics than Claude Code sessions
- Customers need visibility into execution progress and cost (token usage) per runner type
- Consider runner-specific SLAs and timeout recommendations

**Integration Requirements:**
- Customers with existing LangGraph graphs may want to import them into vTeam
- Post-MVP support for custom graph definitions should be designed with customer workflows in mind
- Consider providing migration paths from standalone LangGraph apps to vTeam-hosted execution

**Support and Troubleshooting:**
- LangGraph errors and failure modes differ from Claude Code errors
- Support teams need training on both runner types and how to troubleshoot each
- Consider runner-specific logging and diagnostics to simplify debugging

**Cost Considerations:**
- LangGraph workflows may consume tokens differently than Claude Code sessions
- Provide customers with cost estimation tools per runner type
- Consider runner-specific rate limiting or quotas if costs differ significantly

**Compliance and Security:**
- Some customers may have restrictions on which AI frameworks can be used
- Ensure LangGraph runner meets the same security standards as Claude Code runner
- Document data flow and external API calls made by LangGraph components

**Organizational Adoption:**
- Some teams may prefer Claude Code while others prefer LangGraph
- Runner selection is made at the project/workspace level - all sessions within a project use the same runner
- Consider project-level runner preferences/defaults to simplify session creation
- Enable teams to standardize on one runner type per project for consistency

**Backward Compatibility:**
- **CRITICAL: Zero changes to Claude Code runner implementation** - all existing code, configuration, and behavior remains identical
- Existing Claude Code sessions must continue working unchanged
- API clients that don't specify `runnerType` should get Claude Code (current behavior)
- RFE Workflow artifacts produced by either runner follow the same schema and structure (enabling consistent tooling and validation across runner types)
- Ensure versioning strategy allows runner-specific feature rollouts without breaking changes
