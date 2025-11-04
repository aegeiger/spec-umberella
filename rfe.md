# Multi-Workflow Support with LangGraph Integration

**Feature Overview:**

The Ambient Agentic Runner (vTeam) platform currently supports a single workflow type: the RFE (Request for Enhancement) workflow, which runs on Claude Code with SpecKit templates. This enhancement adds support for multiple workflow execution engines, starting with LangGraph as an alternative runner, enabling teams to leverage graph-based agent orchestration patterns while maintaining backward compatibility with existing workflows. This matters to users because it unlocks new automation patterns (approval workflows, research pipelines, multi-step decision trees) that are more naturally expressed as directed acyclic graphs than as interactive CLI sessions, expanding the platform's capabilities from specification generation to general-purpose agentic automation.

**Goals:**

* **Enable Multiple Workflow Types**: Allow users to create workflows using different execution engines (Claude Code, LangGraph, and future runners) on the same platform infrastructure
* **Maintain Backward Compatibility**: Ensure all existing RFE workflows continue to function without modification; default behavior remains unchanged
* **Leverage Graph-Based Patterns**: Provide users access to LangGraph's stateful graph execution model for workflows requiring branching logic, parallel execution, human-in-the-loop approval gates, and checkpointed state management
* **Preserve Platform Consistency**: Implement multi-workflow support using existing architectural patterns (runner-shell abstraction, Kubernetes operators, PVC state management, WebSocket transport) without requiring core infrastructure changes

**Who Benefits:**
* **Development Teams**: Gain flexibility to choose the right execution engine for each use case (interactive specification creation vs. automated approval workflows)
* **Platform Engineers**: Can extend vTeam with additional runner types in the future following the same pattern
* **Operations Teams**: Benefit from consistent observability, state management, and operational patterns across all workflow types

**Expected Outcomes:**
* **Today's State**: Users can only create RFE workflows using Claude Code + SpecKit; all automation must fit the interactive CLI model
* **Future State**: Users can create both RFE workflows (Claude Code) and LangGraph workflows (graph-based), choosing the appropriate execution model for their use case; platform supports extensible workflow type registration

**Out of Scope:**

* Migration of existing RFE workflows to LangGraph (existing workflows remain on Claude Code)
* Real-time collaboration features (multiple users editing same workflow simultaneously)
* Workflow versioning and rollback capabilities (will be addressed in future enhancements)
* GUI-based graph builder/editor (users define graphs in Python code)
* Support for additional runner types beyond Claude Code and LangGraph in MVP (framework established but implementations deferred)
* Cross-workflow orchestration (workflows calling other workflows)
* Workflow marketplace or sharing capabilities

**Requirements:**

**MVP Requirements** (Feature shifts if these slip):

* **[MVP-1] LangGraph Runner Implementation**: Create a LangGraph adapter that implements the runner-shell interface, supporting graph loading, execution, and streaming output via WebSocket
* **[MVP-2] Workflow Type Selection**: Extend AgenticSession CRD with `workflowType` field; modify operator to launch appropriate runner image based on workflow type (rfe → Claude Code, langgraph → LangGraph)
* **[MVP-3] Checkpoint State Management**: Implement SQLite-based checkpoint persistence on PVC with automatic pruning to prevent storage exhaustion
* **[MVP-4] LangGraphWorkflow CRD**: Define and implement Custom Resource Definition for LangGraph workflows with fields for graph definition, configuration, and execution parameters
* **[MVP-5] Backend API Handlers**: Create Go-based REST API handlers for LangGraph workflow CRUD operations (create, read, update, delete, list)
* **[MVP-6] Backward Compatibility**: Ensure 100% compatibility with existing RFE workflows; default to "rfe" workflow type when not specified
* **[MVP-7] Security Validation**: Implement Python AST validation and import whitelisting to prevent arbitrary code execution in user-defined graphs

**Non-MVP Requirements** (Desirable but can slip):

* **[Post-MVP-1] Human-in-the-Loop Support**: Bidirectional WebSocket communication for graph interrupt nodes requiring user input
* **[Post-MVP-2] Frontend UI Components**: Web interface for creating and configuring LangGraph workflows
* **[Post-MVP-3] Advanced Observability**: Prometheus metrics for graph execution performance, node-level timing, checkpoint I/O
* **[Post-MVP-4] Session Continuation**: Resume workflow execution from parent session's checkpoint state
* **[Post-MVP-5] Postgres Checkpointer**: Alternative checkpoint backend for high-scale deployments (SQLite sufficient for MVP)

**Done - Acceptance Criteria:**

The feature is complete and successful when:

* **AC-1 Workflow Execution**: A user can create an AgenticSession with `workflowType: langgraph` and the operator launches the LangGraph runner container (not Claude Code runner)
* **AC-2 Graph Execution**: The LangGraph runner successfully loads a user-defined graph (Python code), executes it end-to-end, and streams output to the backend via WebSocket
* **AC-3 State Persistence**: Checkpoints are saved to `.langgraph/checkpoints.db` on PVC after each node execution; session resumes from last checkpoint if interrupted
* **AC-4 API Operations**: Users can create, retrieve, update, and delete LangGraphWorkflow resources via REST API, and these operations correctly manage the underlying Kubernetes Custom Resources
* **AC-5 Backward Compatibility**: All existing RFE workflows continue to execute without modification; creating an AgenticSession without specifying `workflowType` defaults to "rfe" and launches Claude Code runner
* **AC-6 Security Validation**: Attempting to load a graph with dangerous operations (e.g., `os.system`, `eval`, `__import__`) results in validation failure with clear error message
* **AC-7 Operational Parity**: LangGraph workflows appear in session lists, support stop/delete operations, and stream output to frontend with same UX as Claude Code sessions
* **AC-8 Documentation**: Quick-start guide and example graphs are available for developers building LangGraph workflows

**Use Cases - i.e. User Experience & Workflow:**

**Use Case 1: Approval Workflow with Branching Logic**

**Actor:** Development team lead

**Main Success Scenario:**
1. User creates a LangGraphWorkflow defining an approval graph: PR Review → Manager Approval → Deploy (with rejection branches returning to author)
2. User creates AgenticSession with `workflowType: langgraph` and `workflowRef` pointing to the LangGraphWorkflow
3. Operator detects new session, launches LangGraph runner container
4. Graph executes: analyzes PR, requests manager input via HITL interrupt node
5. Manager provides approval/rejection via frontend
6. Graph resumes from checkpoint, proceeds to deployment or returns to author
7. Final state saved to checkpoint; session marked complete

**Alternative Flow:**
* **A1**: Manager doesn't respond within timeout → Graph proceeds with default action (rejection)
* **A2**: Graph execution fails mid-node → Checkpoint preserves state; user can retry from last successful node
* **A3**: User stops session → Checkpoint saved; new session can resume from parent checkpoint

**Use Case 2: Research Pipeline with Parallel Execution**

**Actor:** Research team

**Main Success Scenario:**
1. User defines graph: Query → [Search Papers || Search Patents || Search Standards] → Merge Results → Generate Report
2. Graph executes three search nodes in parallel (LangGraph parallel execution)
3. Results merge at synchronization node
4. Report generated and saved to workspace
5. Output streamed to frontend in real-time via WebSocket
6. Session completes successfully

**Use Case 3: Existing RFE Workflow (Backward Compatibility)**

**Actor:** Product manager (existing user)

**Main Success Scenario:**
1. User creates RFEWorkflow (same as today)
2. User clicks "Seed Repositories" (same as today)
3. Backend creates AgenticSession without specifying `workflowType` (defaults to "rfe")
4. Operator launches Claude Code runner (same as today)
5. Claude Code executes RFE workflow with SpecKit templates (same as today)
6. User sees no changes in UX or behavior

**Use Case Diagram:**

```
┌─────────────┐
│    User     │
└──────┬──────┘
       │
       ├──────────────────────┐
       │                      │
       ▼                      ▼
┌─────────────────┐    ┌─────────────────┐
│ Create RFE WF   │    │ Create LG WF    │
│ (Claude Code)   │    │ (LangGraph)     │
└────────┬────────┘    └────────┬────────┘
         │                      │
         ▼                      ▼
┌──────────────────────────────────────┐
│     Create AgenticSession            │
│  - RFE: workflowType="rfe" (default) │
│  - LG: workflowType="langgraph"      │
└────────┬─────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────┐
│          Operator Watch              │
│   - Detect new session               │
│   - Select runner image              │
└────────┬─────────────────────────────┘
         │
         ├──────────────┬───────────────┐
         ▼              ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Claude Code  │ │  LangGraph   │ │  Future      │
│   Runner     │ │   Runner     │ │  Runners     │
└──────┬───────┘ └──────┬───────┘ └──────────────┘
       │                │
       └────────┬───────┘
                ▼
       ┌──────────────────┐
       │ Stream to        │
       │ Frontend         │
       └──────────────────┘
```

**Documentation Considerations:**

* **Quick Start Guide**: Step-by-step tutorial for creating first LangGraph workflow with example graph (approval workflow)
* **Graph Definition Reference**: Documentation on supported LangGraph patterns, node types, state schemas, and checkpoint behavior
* **Migration Guide**: Guidance on when to use Claude Code (RFE workflows) vs. LangGraph (approval/research workflows); not a migration path but a decision framework
* **API Reference Updates**: Extend existing vTeam API docs with LangGraphWorkflow endpoints, request/response schemas
* **Security Best Practices**: Guidelines on safe graph definitions, import restrictions, avoiding dangerous operations
* **Troubleshooting Guide**: Common issues (checkpoint corruption, graph validation failures, timeout handling) with resolution steps
* **Operator Configuration**: Document new environment variables (LANGGRAPH_RUNNER_IMAGE) and ConfigMap changes
* **Extension Documentation**: Link to existing runner-shell documentation at `/components/runners/runner-shell/README.md` (no changes needed to runner-shell itself)

**Questions to answer:**

**Architectural Questions:**

1. **Graph Definition Format**: Should we support only Python-based graph definitions (via code), or also provide JSON/YAML declarative format for simpler workflows?
   * *Recommendation*: Start with Python-only (leverages full LangGraph API); consider declarative format post-MVP if user feedback indicates need

2. **Checkpointer Backend**: Is SQLite sufficient for checkpoint persistence, or should we provide Postgres option for high-scale deployments?
   * *Recommendation*: SQLite for MVP (simpler, no additional infrastructure); add Postgres option in Phase 2 if performance testing indicates need

3. **Workflow Registry**: Should we implement a central workflow registry/catalog, or rely on Git repositories as source of truth for workflow definitions?
   * *Recommendation*: Git-as-source-of-truth for MVP (consistent with RFE model); consider registry if we reach 20+ workflow types

4. **CRD Unification**: Should we create unified `WorkflowTemplate` CRD or maintain separate CRDs per workflow type (RFEWorkflow, LangGraphWorkflow)?
   * *Recommendation*: Separate CRDs for MVP (type safety, clear schemas); consider unified template if we exceed 5 workflow types

**Security Questions:**

5. **Graph Validation Depth**: Is AST-based validation sufficient, or do we need runtime sandboxing (e.g., gVisor, Kata Containers)?
   * *Recommendation*: AST validation + import whitelist for MVP; monitor for evasion attempts; escalate to sandboxing if needed

6. **Checkpoint Encryption**: Should checkpoints be encrypted at rest beyond PVC-level encryption?
   * *Recommendation*: PVC encryption sufficient for MVP; add application-level encryption if storing sensitive credentials in state

**Operational Questions:**

7. **Checkpoint Retention**: What is appropriate retention policy for checkpoints? (Current proposal: keep last 10)
   * *Needs Input*: Storage team input on PVC capacity and growth projections

8. **Timeout Configuration**: What are appropriate defaults for graph node timeouts and human-in-the-loop wait times?
   * *Recommendation*: 30-minute default timeout per node; 24-hour HITL wait time; both configurable per workflow

9. **Observability Metrics**: What Prometheus metrics should we expose?
   * *Recommendation*: Graph execution time, node execution time, checkpoint save/load latency, active sessions by workflow type

**Integration Questions:**

10. **Frontend Changes**: Should LangGraph workflows have distinct UI from RFE workflows, or unified session interface?
    * *Recommendation*: Unified session interface (both are AgenticSessions); add workflow-specific details panel

**Background & Strategic Fit:**

**Problem Context:**

The vTeam platform was designed as a "virtual team" of AI agents for software development automation. The initial implementation focused on the RFE (Request for Enhancement) workflow, leveraging Claude Code's interactive CLI model with SpecKit templates to generate comprehensive feature specifications. This pattern works excellently for its intended purpose: guided, conversational spec creation with human feedback.

However, teams have expressed interest in automating workflows that don't fit the interactive CLI model:
* **Approval workflows** requiring branching logic (approve/reject/escalate)
* **Research pipelines** with parallel execution (search multiple sources simultaneously)
* **Multi-step decision trees** with complex state machines (incident response, deployment gates)
* **Batch processing** with retry logic and error handling

These patterns are more naturally expressed as directed acyclic graphs than as sequential CLI interactions.

**Strategic Value:**

1. **Platform Extensibility**: Establishes vTeam as a general-purpose agentic automation platform, not just an RFE tool
2. **Competitive Positioning**: Multi-workflow support positions vTeam alongside platforms like Temporal, Airflow, but with AI-native execution
3. **Technology Investment**: LangGraph represents state-of-the-art in stateful agent orchestration; early adoption builds expertise
4. **Architecture Validation**: Tests whether the runner-shell abstraction (designed for extensibility) actually delivers on that promise

**Alignment with vTeam Architecture:**

The existing vTeam architecture was designed with extensibility in mind:
* **Runner-Shell Abstraction**: Protocol-based interface enabling multiple execution engines
* **Operator Pattern**: Workflow-agnostic job creation (just needs runner image selection)
* **PVC-Centric State**: All state persisted on PVC, whether `.claude/` or `.langgraph/`
* **WebSocket Transport**: Bidirectional streaming works for any runner type

This enhancement validates these architectural decisions by adding a second runner without modifying core infrastructure.

**Technology Rationale:**

**Why LangGraph?**
* **State Management**: Built-in checkpointing and state persistence (critical for long-running workflows)
* **Graph Execution Model**: Natural fit for branching logic, parallel execution, conditional paths
* **LangChain Ecosystem**: Access to extensive library of integrations (APIs, databases, tools)
* **Human-in-the-Loop**: Native support for interrupt nodes requiring human input
* **Observability**: Structured event streaming for debugging and monitoring

**Why Not Alternatives?**
* **CrewAI**: Sequential execution model, less suitable for complex branching
* **AutoGPT**: Autonomous loop model, harder to control and debug
* **Custom Framework**: Reinventing state management and checkpointing is high risk

**Prior Art:**

Similar multi-workflow patterns exist in:
* **GitHub Actions**: Workflows defined in YAML, multiple execution engines (Docker, VM, self-hosted)
* **Temporal**: Workflow orchestration with multiple SDKs (Go, Java, Python)
* **Kubeflow Pipelines**: DAG-based ML workflows with custom operators

vTeam's approach is differentiated by:
1. Kubernetes-native Custom Resource model
2. AI-agent-first execution (not general-purpose compute)
3. Runner-shell abstraction enabling polyglot runners

**Customer Considerations**

**Deployment Considerations:**

* **Image Registry**: LangGraph runner requires separate container image; ensure image registry has capacity and appropriate retention policies
* **Resource Quotas**: LangGraph graphs may have different resource profiles than Claude Code (e.g., parallel node execution); review namespace quotas
* **Network Policies**: Verify LangGraph runner can reach backend WebSocket endpoint; no additional network policy changes expected
* **Storage**: Checkpoint database (`.langgraph/checkpoints.db`) grows with workflow complexity; monitor PVC usage and implement pruning

**Migration Path for Existing Users:**

* **No Action Required**: Existing RFE workflows continue to work without changes; default behavior preserved
* **Opt-In Model**: Users explicitly choose LangGraph by setting `workflowType: langgraph` when creating sessions
* **Training & Enablement**: Provide workshops on when to use Claude Code vs. LangGraph; decision framework based on use case

**Customer-Specific Requirements:**

* **Red Hat Customers**: Ensure LangGraph runner image is available in Red Hat container catalog; verify OpenShift compatibility (no privileged containers required)
* **Air-Gapped Deployments**: Package LangGraph Python dependencies in container image (no runtime pip installs); provide offline documentation
* **Multi-Tenancy**: LangGraphWorkflow CRDs are namespace-scoped (same as RFEWorkflow); RBAC enforcement consistent across workflow types
* **Compliance**: Graph validation (AST parsing, import whitelist) prevents arbitrary code execution; meets security requirements for regulated industries

**Support Considerations:**

* **Debugging**: Provide tools for inspecting checkpoint state, replaying graph execution, viewing node-level logs
* **Error Messages**: Ensure validation failures provide clear, actionable error messages (e.g., "Import 'os.system' is not allowed. Use subprocess with explicit commands.")
* **Upgrade Path**: Checkpoint schema compatibility across LangGraph versions; test upgrades with checkpoint migration if schema changes

**Cost Considerations:**

* **Infrastructure**: Minimal incremental cost (separate container image, same Kubernetes Jobs model)
* **Anthropic API**: LangGraph workflows consume API credits based on LLM calls within graph nodes; provide cost estimation tool
* **Storage**: Checkpoint databases grow over time; implement pruning to manage PVC costs

**Customer Success Metrics:**

* **Adoption Rate**: % of customers creating LangGraph workflows within 90 days of release
* **Workflow Diversity**: Number of distinct workflow types created (target: 5+ unique patterns)
* **Session Success Rate**: % of LangGraph sessions completing successfully (target: >90%)
* **Time-to-Value**: Time from feature release to first production LangGraph workflow (target: <30 days)

**Risk Mitigation:**

* **Feature Flag**: Consider gating LangGraph workflows behind feature flag for phased rollout
* **Early Access Program**: Pilot with 3-5 strategic customers before GA
* **Fallback Plan**: If adoption is low or technical issues arise, maintain existing RFE workflow as stable baseline