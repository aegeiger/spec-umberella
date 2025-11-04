# LangGraph Runner for RFE Workflow

**Feature Overview:**

The Ambient Agentic Runner (vTeam) platform currently supports the RFE (Request for Enhancement) workflow, which runs exclusively on the Claude Code runner with SpecKit templates. This enhancement adds a second runner option—LangGraph—for executing the same RFE workflow. Users will be able to choose between Claude Code runner (current, unchanged) or LangGraph runner (new, graph-based) when creating an RFE workflow. This matters to users because the LangGraph runner provides checkpointed execution (pause/resume at any stage), structured observability (graph node events), and foundation for future enhancements (human-in-the-loop review, validation gates, branching logic) while producing identical outputs (spec.md, plan.md, tasks.md) as the existing Claude Code runner.

**Goals:**

* **Add Runner Choice**: Allow users to select execution engine (Claude Code or LangGraph) when creating RFE workflows, with both runners producing identical specification outputs
* **Maintain 100% Backward Compatibility**: Ensure all existing RFE workflows continue to function without modification; Claude Code runner remains the default when runner is not specified
* **Enable Graph-Based Execution**: Provide access to LangGraph's stateful graph execution model for RFE workflow, including checkpointed state (resume from any stage), structured event streaming, and observable execution traces
* **Preserve Minimal Changes**: Add runner selection with minimal modification to existing infrastructure—single CRD field, operator image selection logic, new container image

**Who Benefits:**
* **Development Teams**: Gain ability to pause/resume RFE workflow at any stage (review spec before generating plan), better debugging with structured execution events
* **Platform Engineers**: Validate that runner-shell abstraction supports multiple execution engines as designed; establish pattern for future runner additions
* **Operations Teams**: Benefit from enhanced observability (graph node timing, checkpoint events) while maintaining consistent operational model (same Jobs, PVCs, WebSocket transport)

**Expected Outcomes:**
* **Today's State**: RFE workflows run only on Claude Code runner; execution is opaque CLI interaction; cannot pause/resume mid-workflow
* **Future State**: Users choose Claude Code (default, unchanged) or LangGraph (new option) when creating RFE workflow; LangGraph provides checkpoint resume, structured events, and foundation for approval gates; both runners produce identical outputs

**Out of Scope:**

* Additional workflow types beyond RFE (approval workflows, research pipelines, etc.) - single workflow only for this enhancement
* Migration of existing RFE workflows to LangGraph (existing workflows remain on Claude Code)
* Human-in-the-loop review gates (e.g., approve spec before generating plan) - foundation established but UI/interaction deferred to post-MVP
* Branching logic or conditional paths in RFE workflow (linear 3-stage execution: specify → plan → tasks)
* GUI-based graph builder/editor (LangGraph runner uses hardcoded RFE workflow graph)
* Real-time collaboration features (multiple users editing same workflow simultaneously)
* Workflow versioning and rollback capabilities (will be addressed in future enhancements)
* Additional runner types beyond Claude Code and LangGraph (e.g., CrewAI, AutoGPT)
* Cross-workflow orchestration (workflows calling other workflows)

**Requirements:**

**MVP Requirements** (Feature shifts if these slip):

* **[MVP-1] Runner Selection Field**: Add `runner` enum field to RFEWorkflow CRD with values `claude-code` (default) or `langgraph`; backend API accepts and validates runner field
* **[MVP-2] Operator Image Selection**: Modify operator to read `runner` field from RFEWorkflow and select appropriate container image (Claude Code runner image or LangGraph runner image) when creating AgenticSession Job
* **[MVP-3] LangGraph Runner Implementation**: Create LangGraph adapter implementing runner-shell interface with RFE workflow graph (specify → plan → tasks nodes), SpecKit template loading, and WebSocket streaming
* **[MVP-4] SpecKit Integration**: LangGraph runner reads SpecKit templates from `.specify/templates/` directory, uses templates as system prompts for Claude API calls, produces identical outputs (spec.md, plan.md, tasks.md) as Claude Code runner
* **[MVP-5] Checkpoint State Management**: Implement SQLite-based checkpoint persistence on PVC (`.langgraph/checkpoints.db`) with automatic pruning (keep last 10 checkpoints) to prevent storage exhaustion
* **[MVP-6] Identical Output Verification**: LangGraph runner produces spec.md, plan.md, tasks.md files with same structure and content quality as Claude Code runner; both runners are functionally equivalent from user perspective
* **[MVP-7] Backward Compatibility Testing**: Validate that existing RFE workflows (without `runner` field) continue to execute on Claude Code runner with no behavior changes; default value is `claude-code`

**Non-MVP Requirements** (Desirable but can slip):

* **[Post-MVP-1] Session Continuation**: Resume LangGraph session from parent checkpoint (continue from specify, plan, or tasks stage)
* **[Post-MVP-2] Human-in-the-Loop Gates**: Add interrupt nodes for user review (e.g., approve spec before generating plan); requires bidirectional WebSocket and frontend UI
* **[Post-MVP-3] Frontend Runner Selection UI**: Web interface dropdown for selecting runner when creating RFE workflow (currently requires API/kubectl)
* **[Post-MVP-4] Advanced Observability**: Prometheus metrics for graph execution performance, node-level timing, checkpoint I/O latency
* **[Post-MVP-5] Postgres Checkpointer**: Alternative checkpoint backend for high-scale deployments (SQLite sufficient for MVP)
* **[Post-MVP-6] Validation Nodes**: Optional graph nodes for validating spec completeness, plan feasibility before proceeding to next stage

**Done - Acceptance Criteria:**

The feature is complete and successful when:

* **AC-1 Runner Selection**: A user can create an RFEWorkflow with `runner: langgraph` via API, and the field is persisted in the Kubernetes Custom Resource
* **AC-2 Image Selection**: When an AgenticSession is created for an RFEWorkflow with `runner: langgraph`, the operator launches the LangGraph runner container image (not Claude Code runner)
* **AC-3 RFE Execution**: The LangGraph runner executes the 3-stage RFE workflow (specify → plan → tasks), loading SpecKit templates, calling Claude API, and producing spec.md, plan.md, tasks.md files in the workspace
* **AC-4 Output Equivalence**: The spec.md, plan.md, and tasks.md files generated by LangGraph runner are structurally identical and of equivalent quality to those generated by Claude Code runner for the same input prompt
* **AC-5 Checkpoint Persistence**: Checkpoints are saved to `.langgraph/checkpoints.db` after each graph node execution (specify, plan, tasks); checkpoint files are visible on PVC
* **AC-6 Backward Compatibility**: Creating an RFEWorkflow without specifying `runner` field (or with `runner: claude-code`) executes on Claude Code runner with no behavior changes; existing workflows unaffected
* **AC-7 Streaming Output**: LangGraph runner streams execution output to backend via WebSocket; frontend displays real-time progress updates identical to Claude Code runner UX
* **AC-8 Documentation**: README documents runner selection field, explains when to use each runner (Claude Code for interactive, LangGraph for checkpointed), provides example API request

**Use Cases - i.e. User Experience & Workflow:**

**Use Case 1: Create RFE Workflow with LangGraph Runner**

**Actor:** Product manager

**Main Success Scenario:**
1. User creates RFEWorkflow via API, specifying `runner: langgraph`, title "Add user authentication", and umbrella repo URL
2. Backend validates request, creates RFEWorkflow Custom Resource with `spec.runner: langgraph`
3. User clicks "Seed Repositories" - backend clones repos, adds SpecKit templates to `.specify/` directory (same as Claude Code)
4. User creates AgenticSession with prompt "Implement OAuth2 authentication with GitHub provider"
5. Operator detects new session, reads parent RFEWorkflow, sees `runner: langgraph`
6. Operator creates Kubernetes Job with LangGraph runner container image
7. LangGraph runner initializes, loads RFE graph definition (specify → plan → tasks)
8. Graph executes:
   - **Specify node**: Reads `.specify/templates/spec-template.md`, calls Claude API, writes `spec.md`
   - **Plan node**: Reads spec.md and plan template, calls Claude API, writes `plan.md`
   - **Tasks node**: Reads plan.md and tasks template, calls Claude API, writes `tasks.md`
9. Checkpoints saved after each node to `.langgraph/checkpoints.db`
10. Output streamed to frontend; user sees "Generating specification...", "Generating plan...", "Generating tasks..."
11. Session completes; user views spec.md, plan.md, tasks.md in workspace (identical quality to Claude Code output)

**Alternative Flow:**
* **A1**: Graph execution fails at plan node → Checkpoint preserves spec.md; user can restart from plan stage (post-MVP feature)
* **A2**: User stops session mid-execution → Checkpoint saved; can resume later
* **A3**: Network interruption → Graph execution pauses; resumes when connectivity restored

**Use Case 2: Existing RFE Workflow (Backward Compatibility)**

**Actor:** Product manager (existing user, unaware of new feature)

**Main Success Scenario:**
1. User creates RFEWorkflow via API (does NOT specify `runner` field)
2. Backend creates RFEWorkflow with `spec.runner: claude-code` (default value)
3. User seeds repos, creates session (same workflow as before)
4. Operator reads `runner: claude-code`, launches Claude Code runner container (existing behavior)
5. Claude Code executes RFE workflow with SpecKit (same as always)
6. User sees no difference in UX, outputs, or behavior

**Use Case 3: Compare Runner Outputs**

**Actor:** Platform engineer (validating LangGraph runner)

**Main Success Scenario:**
1. Engineer creates two RFEWorkflows with identical prompts: one with `runner: claude-code`, one with `runner: langgraph`
2. Executes both sessions in parallel
3. Compares outputs:
   - spec.md content structure (both have Overview, Goals, Requirements, etc.)
   - plan.md content structure (both have Architecture, Dependencies, Implementation sections)
   - tasks.md content structure (both have actionable task lists)
4. Validates quality equivalence (both runners produce comprehensive, actionable specifications)
5. Checks checkpoint files: `.langgraph/checkpoints.db` exists for LangGraph session, not for Claude Code session
6. Confirms LangGraph session can be resumed (post-MVP), Claude Code cannot

**Use Case Diagram:**

```
┌─────────────┐
│    User     │
└──────┬──────┘
       │
       │ Create RFEWorkflow
       ▼
┌─────────────────────────────────┐
│  Specify runner field           │
│  - runner: claude-code (default)│
│  - runner: langgraph            │
└────────┬────────────────────────┘
         │
         │ Backend creates RFEWorkflow CR
         ▼
┌─────────────────────────────────┐
│     RFEWorkflow CR              │
│  spec:                          │
│    runner: langgraph            │
│    repos: [...]                 │
└────────┬────────────────────────┘
         │
         │ User creates AgenticSession
         ▼
┌─────────────────────────────────┐
│    Operator Reconcile           │
│  1. Get AgenticSession          │
│  2. Lookup parent RFEWorkflow   │
│  3. Read runner field           │
│  4. Select container image      │
└────────┬────────────────────────┘
         │
         ├──────────────┬─────────────┐
         ▼              ▼             ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Claude Code  │ │  LangGraph   │ │  Future      │
│   Runner     │ │   Runner     │ │  Runners     │
│  (existing)  │ │  (new)       │ │  (deferred)  │
└──────┬───────┘ └──────┬───────┘ └──────────────┘
       │                │
       │ Both execute RFE workflow
       │ Both produce spec.md, plan.md, tasks.md
       │
       └────────┬───────┘
                ▼
       ┌──────────────────┐
       │ Stream to        │
       │ Frontend         │
       │ (WebSocket)      │
       └──────────────────┘
```

**Documentation Considerations:**

* **Runner Selection Guide**: Document when to use Claude Code vs. LangGraph runner (Claude Code: default, stable, interactive; LangGraph: checkpointed, observable, foundation for future features like approval gates)
* **API Reference Updates**: Update RFEWorkflow API documentation with `runner` field, enum values, default behavior
* **Quick Start Example**: Provide curl/API example for creating RFEWorkflow with `runner: langgraph`
* **Checkpoint Behavior**: Explain checkpoint persistence, pruning policy (keep last 10), storage location (`.langgraph/checkpoints.db`), and future resume capability
* **Output Equivalence**: Document that both runners produce functionally identical outputs; choice of runner is about execution model (CLI vs graph) not output quality
* **Troubleshooting Guide**: Common issues (LangGraph runner fails to load templates, checkpoint database corruption, output format differences) with resolution steps
* **Operator Configuration**: Document new environment variable `LANGGRAPH_RUNNER_IMAGE` for configuring LangGraph container image
* **Migration Path**: Clarify that there is NO migration needed; existing workflows remain on Claude Code; new workflows can opt-in to LangGraph

**Questions to answer:**

**Implementation Questions:**

1. **Template Reuse Strategy**: Should LangGraph runner execute SpecKit CLI commands (subprocess) or parse templates directly and call Claude API natively?
   * *Recommendation*: Hybrid approach - parse SpecKit templates (`.specify/templates/*.md`) to extract prompts, call Claude API directly via LangChain, avoid subprocess overhead

2. **Checkpoint Pruning Policy**: Keep last N checkpoints per session - what is appropriate N value?
   * *Recommendation*: Keep last 10 checkpoints; configurable via environment variable `CHECKPOINT_RETENTION_COUNT`

3. **Default Runner Value**: Should default be explicit (`claude-code`) or implicit (empty string means Claude Code)?
   * *Recommendation*: Explicit default `claude-code` in CRD schema; clearer for users and eliminates ambiguity

4. **Graph Definition Location**: Should RFE workflow graph be hardcoded in LangGraph runner or loaded from file (`.specify/graphs/rfe.py`)?
   * *Recommendation*: Hardcoded for MVP (single workflow, no customization needed); consider file-based in post-MVP if users request graph customization

**Operational Questions:**

5. **Checkpoint Storage Growth**: What is estimated checkpoint database size? How quickly does it grow?
   * *Needs Input*: Storage team input on PVC capacity and growth monitoring

6. **Runner Image Registry**: Where should LangGraph runner image be published? Same registry as Claude Code runner?
   * *Recommendation*: Same registry (`quay.io/ambient-code/langgraph-runner`), same tagging strategy (version tags + latest)

7. **Observability Metrics**: What metrics should LangGraph runner expose initially?
   * *Recommendation*: MVP metrics - session status, node execution time, checkpoint save latency; defer Prometheus integration to post-MVP

8. **Failure Scenarios**: What happens if LangGraph runner crashes mid-execution?
   * *Recommendation*: Checkpoint preserves state; operator restarts pod; session can resume from last checkpoint (post-MVP feature); MVP behavior: session marked failed, user restarts manually

**Integration Questions:**

9. **Frontend Changes**: Does frontend need changes to support runner selection?
   * *Recommendation*: Not for MVP (selection via API/kubectl only); add dropdown in post-MVP

10. **Seeding Compatibility**: Are there any changes needed to repo seeding logic for LangGraph runner?
    * *Answer*: No changes needed; both runners use same `.specify/templates/` directory structure

**Validation Questions:**

11. **Output Equivalence Testing**: How do we validate that LangGraph outputs are equivalent to Claude Code outputs?
    * *Recommendation*: Automated test suite that compares section headers, content length, markdown structure; manual review of 10 sample outputs for quality equivalence

12. **Backward Compatibility Testing**: What test cases ensure existing workflows are unaffected?
    * *Recommendation*: Run 50 existing RFE workflows without `runner` field; verify all execute on Claude Code runner; compare outputs to historical baselines

**Background & Strategic Fit:**

**Problem Context:**

The vTeam platform currently executes RFE workflows using the Claude Code runner, which wraps the Claude Code CLI tool. This approach works well for interactive, conversational specification generation, but has limitations:

* **Opaque Execution**: CLI interaction is black-box from observability perspective; difficult to understand what stage workflow is in
* **No Pause/Resume**: Cannot pause mid-workflow to review intermediate outputs (e.g., review spec before generating plan); must complete entire workflow or start over
* **Limited Extensibility**: Adding features like approval gates, validation steps, or conditional logic requires modifying Claude Code CLI, which is external dependency
* **Single Execution Model**: All RFE workflows forced into CLI interaction model, even when graph-based orchestration might be more appropriate

**Strategic Value:**

1. **Validates Runner-Shell Abstraction**: Proves that the runner-shell interface successfully abstracts execution engine; adding second runner validates architectural investment
2. **Foundation for Advanced Features**: Checkpointed execution enables future enhancements (human-in-the-loop review, resume from any stage, validation gates) without changing core infrastructure
3. **Technology Evaluation**: LangGraph represents state-of-the-art in stateful agent orchestration; adding as runner option allows team to gain experience with technology before broader adoption
4. **Risk Mitigation**: By maintaining Claude Code runner as default, we derisk LangGraph adoption; users opt-in to new runner, existing workflows unaffected
5. **Competitive Positioning**: Checkpointed, observable workflow execution differentiates vTeam from simpler automation tools

**Alignment with vTeam Architecture:**

The runner-shell abstraction was explicitly designed to support multiple execution engines:
* **Protocol Interface**: `MessageType`, `SessionStatus` enums work for any runner
* **Transport Layer**: WebSocket streaming is runner-agnostic
* **Context Management**: Workspace, environment variables handled uniformly
* **State Persistence**: PVC-based state works for `.claude/` or `.langgraph/` directories

This enhancement validates that architectural design delivers on its promise. Adding LangGraph runner requires:
* **Zero changes** to runner-shell protocol, transport, or context
* **Minimal changes** to operator (single function for image selection)
* **No changes** to backend API contract (AgenticSession spec unchanged)

**Technology Rationale:**

**Why LangGraph for RFE Workflow?**

LangGraph may seem overengineered for a simple 3-node linear graph (specify → plan → tasks), but it provides:

1. **Checkpointed Execution**: Automatic state persistence after each node; foundation for pause/resume
2. **Structured Events**: Graph emits events for node start/end, state updates; enables rich observability
3. **Extensibility Foundation**: Easy to add nodes (validation, approval, research) without rewriting orchestration
4. **Proven Technology**: LangGraph is production-ready, actively maintained, widely adopted in AI agent space
5. **Minimal Learning Curve**: Graph definition is simple Python; no complex DSL or configuration format

**Why Not Alternatives?**
* **Direct Implementation**: Could implement checkpointing ourselves, but reinventing state management is high-risk and error-prone
* **CrewAI**: More opinionated about multi-agent collaboration; less suitable for simple pipeline orchestration
* **Temporal/Airflow**: General-purpose workflow engines; overkill for AI agent workflows; not LLM-native

**LangGraph Trade-offs:**
* **Pros**: Battle-tested checkpointing, rich ecosystem, active development, LLM-native design
* **Cons**: Additional dependency (Python package), learning curve for team, checkpoint storage overhead (~50KB per checkpoint)

**Decision**: Benefits outweigh costs; LangGraph provides professional-grade state management we would otherwise have to build ourselves.

**Prior Art:**

Other platforms with multiple runner support:
* **GitHub Actions**: Workflows run on GitHub-hosted runners, self-hosted runners, or Actions Runner Controller (Kubernetes); runner selection via `runs-on` field
* **GitLab CI**: Pipelines run on shared runners, specific runners, or group runners; runner selection via tags
* **Jenkins**: Jobs run on controller or agents; agent selection via labels

vTeam's approach is similar:
* Single workflow type (RFE) with multiple execution engines (runners)
* Runner selection via explicit field in workflow specification
* Operator dispatches to appropriate runner based on field value
* Both runners produce equivalent outputs

**Customer Considerations**

**Deployment Considerations:**

* **Container Image**: LangGraph runner requires separate container image (`quay.io/ambient-code/langgraph-runner:latest`); ensure image registry has capacity
* **Resource Quotas**: LangGraph runner has similar resource profile to Claude Code runner (single-threaded, CPU-bound during LLM API calls); no quota changes expected
* **Network Policies**: LangGraph runner needs same network access as Claude Code runner (backend WebSocket, Anthropic API); no policy changes needed
* **Storage**: Checkpoint database adds ~50KB per session; with 1000 sessions, ~50MB total; negligible compared to workspace files (spec.md, plan.md, tasks.md)
* **Dependency Management**: LangGraph runner packages all Python dependencies in container image; no runtime pip installs required

**Migration Path for Existing Users:**

* **No Migration Needed**: Existing RFE workflows continue to use Claude Code runner by default; no user action required
* **Opt-In Model**: Users who want LangGraph benefits explicitly set `runner: langgraph` when creating new workflows
* **Decision Framework**: Document when to use each runner:
  - **Claude Code**: Default choice, proven, stable, interactive experience
  - **LangGraph**: Opt-in for checkpointed execution, structured observability, foundation for approval gates (post-MVP)
* **Training Materials**: Provide comparison guide, demo video showing checkpoint behavior, FAQ on runner differences

**Customer-Specific Requirements:**

* **Red Hat Customers**: Ensure LangGraph runner image is available in Red Hat container catalog with appropriate certifications; verify OpenShift compatibility (no privileged containers, runs as non-root)
* **Air-Gapped Deployments**: Package LangGraph runner image with all dependencies (LangGraph, LangChain, Anthropic SDK); no internet access required at runtime
* **Multi-Tenancy**: Runner selection is per-RFEWorkflow, which is namespace-scoped; tenants isolated as with existing Claude Code runner
* **Compliance**: LangGraph runner calls Anthropic API (same as Claude Code); no additional compliance concerns; checkpoint data stored on PVC (same encryption as workspace files)

**Support Considerations:**

* **Debugging**: Provide tools for inspecting checkpoint database (CLI tool: `checkpoint-inspect`), viewing graph execution trace, comparing outputs between runners
* **Error Messages**: LangGraph validation failures should provide clear guidance (e.g., "Failed to load SpecKit template: .specify/templates/spec-template.md not found. Ensure repositories are seeded.")
* **Performance**: If LangGraph runner is slower than Claude Code, investigate checkpoint I/O overhead; consider async checkpoint writes
* **Compatibility**: Document which LangGraph versions are supported; test checkpoint schema compatibility across LangGraph updates

**Cost Considerations:**

* **Infrastructure**: Minimal incremental cost - separate container image (~500MB), same compute resources as Claude Code runner
* **Anthropic API**: LangGraph runner makes same number of API calls as Claude Code runner (3 calls: specify, plan, tasks); equivalent cost per session
* **Storage**: Checkpoint database adds ~50KB per session; negligible storage cost
* **Development**: 2-3 weeks engineering effort for MVP; lower than expected due to simplified scope (no new workflow types, no LangGraph CRD)

**Customer Success Metrics:**

* **Adoption Rate**: % of RFE workflows created with `runner: langgraph` (target: 10% within 90 days, 25% within 180 days)
* **Output Quality Equivalence**: User satisfaction scores for LangGraph outputs vs Claude Code outputs (target: within 5% difference)
* **Session Success Rate**: % of LangGraph sessions completing successfully (target: >95%, matching Claude Code success rate)
* **Checkpoint Usage**: % of failed LangGraph sessions that resume from checkpoint in post-MVP (target: >50% utilization)
* **Support Tickets**: Number of LangGraph-related support tickets (target: <5 per month, indicating stable implementation)

**Risk Mitigation:**

* **Backward Compatibility**: Default to Claude Code runner; existing workflows unaffected; risk of breaking changes: **low**
* **Output Quality**: Validate LangGraph outputs match Claude Code quality through automated testing and manual review; risk of quality degradation: **medium** (mitigated by equivalence testing)
* **Performance**: Benchmark LangGraph runner against Claude Code runner; ensure comparable execution time; risk of performance regression: **low** (checkpoint writes are async)
* **Adoption**: If adoption is low (<5% after 180 days), maintain LangGraph runner as experimental feature and deprioritize enhancements; risk of wasted effort: **low** (2-3 week investment)
* **Maintenance Burden**: LangGraph runner adds second runner to maintain (dependencies, updates, bug fixes); mitigate by ensuring runner-shell abstraction minimizes duplicated code; risk: **medium** (accepted trade-off for innovation)