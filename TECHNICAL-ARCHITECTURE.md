# Technical Architecture: LangGraph Runner for RFE Workflow

**Author:** Stella (Staff Engineer)
**Date:** 2025-11-04
**Status:** Technical Guidance for RFE

## Executive Summary

This document provides technical architecture guidance for adding a LangGraph runner option to the existing RFE workflow in the Ambient Agentic Runner (vTeam) platform. The analysis focuses on maintaining architectural integrity while introducing runner choice with minimal infrastructure changes.

**Key Finding:** The existing runner-shell abstraction is well-designed for multi-runner support. The primary implementation is a single CRD field addition and operator image selection logic. Both runners execute the same RFE workflow (specify → plan → tasks) and produce identical outputs.

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

from runner_shell.core.context import RunnerContext
from runner_shell.core.protocol import MessageType
from langgraph.graph import StateGraph
from langgraph.checkpoint.sqlite import AsyncSqliteSaver

class LangGraphAdapter:
    """Adapter for LangGraph execution of RFE workflow"""

    def __init__(self):
        self.context = None
        self.shell = None
        self.graph = None
        self.checkpointer = None

    async def initialize(self, context: RunnerContext):
        """
        Initialize LangGraph runtime:
        - Load RFE graph definition (specify → plan → tasks)
        - Load SpecKit templates from .specify/ directory
        - Set up checkpointer (SQLite for state)
        - Configure streaming output
        """
        self.context = context
        self.graph = self._build_rfe_graph()
        self.checkpointer = await self._setup_checkpointer()

    async def run(self):
        """
        Execute RFE workflow graph:
        - If continuing: load checkpoint from parent session
        - Stream node execution via WebSocket
        - Save checkpoints after each node
        - Write spec.md, plan.md, tasks.md to workspace
        """
        thread_id = self.context.session_id

        # Resume from parent if continuing
        parent_session = self.context.get_env('PARENT_SESSION_ID', '')
        if parent_session:
            thread_id = await self._get_parent_thread_id(parent_session)

        # Execute RFE graph with checkpoint persistence
        async for event in self.graph.astream_events(
            {"prompt": self.context.get_env("PROMPT", "")},
            config={"configurable": {"thread_id": thread_id}},
            version="v1"
        ):
            await self._handle_graph_event(event)

    async def handle_message(self, message: dict):
        """
        Handle backend messages:
        - end_session: gracefully terminate
        """

    def _build_rfe_graph(self) -> StateGraph:
        """
        Build RFE workflow graph with 3 nodes:
        - specify: Generate spec.md using SpecKit spec template
        - plan: Generate plan.md using SpecKit plan template
        - tasks: Generate tasks.md using SpecKit tasks template
        """
        from rfe_graph import create_rfe_graph
        return create_rfe_graph(self.context.workspace_path)
```

**Key Design Decisions:**

1. **No modifications to runner-shell core** - protocol.py, transport_ws.py, shell.py remain unchanged
2. **Adapter implements same interface** - `initialize()`, `run()`, `handle_message()`
3. **State persistence uses PVC** - LangGraph checkpointer writes to `/workspace/sessions/{session-id}/.langgraph/`
4. **WebSocket protocol unchanged** - same message types, just different content
5. **SpecKit template reuse** - Load templates from `.specify/templates/` like Claude Code runner

### Technical Risk: Dependency Conflicts

**Challenge:** LangGraph requires LangChain dependencies that may conflict with Claude SDK.

**Mitigation Strategy:**
```dockerfile
# Separate container images (RECOMMENDED)
FROM python:3.11 AS langgraph-runner
RUN pip install langgraph langchain-core langchain-anthropic
COPY . /app
WORKDIR /app
CMD ["python", "adapter.py"]
```

**Recommendation:** Build separate container images for each runner type. The operator can select the image based on runner field (see Section 3).

---

## 2. RFE Workflow Graph Design

### RFE Workflow Stages

Both runners execute the same 3-stage workflow:

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│ Specify  │ ──▸ │   Plan   │ ──▸ │  Tasks   │
└──────────┘     └──────────┘     └──────────┘
    │                │                │
    ▼                ▼                ▼
 spec.md         plan.md          tasks.md
```

### LangGraph Implementation

```python
# NEW FILE: /components/runners/langgraph-runner/rfe_graph.py

from langgraph.graph import StateGraph
from typing import TypedDict
from pathlib import Path

class RFEState(TypedDict):
    """State for RFE workflow"""
    prompt: str
    workspace_path: str
    spec_content: str
    plan_content: str
    tasks_content: str

async def specify_node(state: RFEState) -> RFEState:
    """
    Generate specification using SpecKit spec template.
    Reads: .specify/templates/spec-template.md
    Writes: spec.md
    """
    from template_loader import SpecKitTemplateLoader

    loader = SpecKitTemplateLoader(Path(state["workspace_path"]))
    spec_template = loader.load_template("spec-template.md")

    # Call Claude API with template as system prompt
    spec_content = await generate_with_claude(
        system_prompt=spec_template,
        user_prompt=state["prompt"]
    )

    # Write spec.md to workspace
    spec_path = Path(state["workspace_path"]) / "spec.md"
    spec_path.write_text(spec_content)

    return {**state, "spec_content": spec_content}

async def plan_node(state: RFEState) -> RFEState:
    """
    Generate implementation plan using SpecKit plan template.
    Reads: .specify/templates/plan-template.md, spec.md
    Writes: plan.md
    """
    from template_loader import SpecKitTemplateLoader

    loader = SpecKitTemplateLoader(Path(state["workspace_path"]))
    plan_template = loader.load_template("plan-template.md")

    # Call Claude API with spec.md context
    plan_content = await generate_with_claude(
        system_prompt=plan_template,
        user_prompt=f"Generate plan based on this spec:\n\n{state['spec_content']}"
    )

    # Write plan.md to workspace
    plan_path = Path(state["workspace_path"]) / "plan.md"
    plan_path.write_text(plan_content)

    return {**state, "plan_content": plan_content}

async def tasks_node(state: RFEState) -> RFEState:
    """
    Generate task breakdown using SpecKit tasks template.
    Reads: .specify/templates/tasks-template.md, plan.md
    Writes: tasks.md
    """
    from template_loader import SpecKitTemplateLoader

    loader = SpecKitTemplateLoader(Path(state["workspace_path"]))
    tasks_template = loader.load_template("tasks-template.md")

    # Call Claude API with plan.md context
    tasks_content = await generate_with_claude(
        system_prompt=tasks_template,
        user_prompt=f"Generate tasks based on this plan:\n\n{state['plan_content']}"
    )

    # Write tasks.md to workspace
    tasks_path = Path(state["workspace_path"]) / "tasks.md"
    tasks_path.write_text(tasks_content)

    return {**state, "tasks_content": tasks_content}

def create_rfe_graph(workspace_path: str) -> StateGraph:
    """Build RFE workflow graph"""
    graph = StateGraph(RFEState)

    # Add nodes
    graph.add_node("specify", specify_node)
    graph.add_node("plan", plan_node)
    graph.add_node("tasks", tasks_node)

    # Define linear workflow
    graph.set_entry_point("specify")
    graph.add_edge("specify", "plan")
    graph.add_edge("plan", "tasks")
    graph.set_finish_point("tasks")

    return graph.compile(checkpointer=None)  # Checkpointer set by adapter
```

### SpecKit Template Loading

```python
# NEW FILE: /components/runners/langgraph-runner/template_loader.py

from pathlib import Path
from typing import Dict

class SpecKitTemplateLoader:
    """Load SpecKit templates from .specify/ directory"""

    def __init__(self, workspace_path: Path):
        self.templates_dir = workspace_path / ".specify" / "templates"

    def load_template(self, template_name: str) -> str:
        """
        Load template file and return content.
        Raises FileNotFoundError if template missing.
        """
        template_path = self.templates_dir / template_name

        if not template_path.exists():
            raise FileNotFoundError(
                f"SpecKit template not found: {template_path}\n"
                f"Ensure repositories are seeded before running RFE workflow."
            )

        return template_path.read_text()

    def list_templates(self) -> Dict[str, Path]:
        """List all available templates"""
        if not self.templates_dir.exists():
            return {}

        return {
            f.name: f
            for f in self.templates_dir.glob("*.md")
        }
```

---

## 3. Runner Selection: Operator Logic

### Current State: Hardcoded Image

The current implementation hardcodes runner image in Job spec (`/components/operator/internal/handlers/sessions.go:398`):

```go
Image: appConfig.AmbientCodeRunnerImage,  // Hardcoded Claude Code runner
```

### Proposed Pattern: Dynamic Image Selection

**Step 1: Add runner field to RFEWorkflow CRD**

```yaml
# MODIFY: /components/manifests/crds/rfeworkflows-crd.yaml

apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: rfeworkflows.vteam.ambient-code
spec:
  # ... existing spec
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
        properties:
          spec:
            type: object
            properties:
              # ... existing fields (title, description, repos, etc.)

              runner:
                type: string
                enum:
                  - claude-code
                  - langgraph
                default: claude-code
                description: |
                  Execution engine for RFE workflow:
                  - claude-code: Interactive CLI execution (default)
                  - langgraph: Graph-based execution with checkpointing
```

**Step 2: Operator reads runner field**

```go
// MODIFY: /components/operator/internal/handlers/sessions.go

// Add after line 250 (Load config for this session)

// Determine runner image based on parent RFEWorkflow
runnerImage := getRunnerImageForSession(currentObj, appConfig)

// ...

// Replace line 398:
// Image: appConfig.AmbientCodeRunnerImage,
Image: runnerImage,

// Add helper function at end of file:

func getRunnerImageForSession(obj *unstructured.Unstructured, config *config.Config) string {
	spec, _, _ := unstructured.NestedMap(obj.Object, "spec")

	// Get parent workflow reference
	workflowRef, found, _ := unstructured.NestedMap(spec, "workflowRef")
	if !found {
		// No workflow ref - default to Claude Code
		return config.AmbientCodeRunnerImage
	}

	// Look up parent RFEWorkflow to get runner field
	workflowKind, _, _ := unstructured.NestedString(workflowRef, "kind")
	workflowName, _, _ := unstructured.NestedString(workflowRef, "name")

	if workflowKind != "RFEWorkflow" {
		// Not an RFE workflow - default to Claude Code
		return config.AmbientCodeRunnerImage
	}

	// Fetch RFEWorkflow CR
	ctx := context.Background()
	rfeWorkflow := &unstructured.Unstructured{}
	rfeWorkflow.SetGroupVersionKind(schema.GroupVersionKind{
		Group:   "vteam.ambient-code",
		Version: "v1alpha1",
		Kind:    "RFEWorkflow",
	})

	err := client.Get(ctx, types.NamespacedName{
		Name:      workflowName,
		Namespace: obj.GetNamespace(),
	}, rfeWorkflow)

	if err != nil {
		log.Printf("⚠️ Failed to fetch RFEWorkflow %s: %v, defaulting to Claude runner", workflowName, err)
		return config.AmbientCodeRunnerImage
	}

	// Read runner field from RFEWorkflow
	rfeSpec, _, _ := unstructured.NestedMap(rfeWorkflow.Object, "spec")
	runner, _, _ := unstructured.NestedString(rfeSpec, "runner")

	switch strings.ToLower(runner) {
	case "langgraph":
		if config.LangGraphRunnerImage == "" {
			log.Printf("⚠️ LangGraph runner image not configured, falling back to Claude runner")
			return config.AmbientCodeRunnerImage
		}
		log.Printf("✅ Using LangGraph runner for RFEWorkflow %s", workflowName)
		return config.LangGraphRunnerImage

	case "claude-code", "":
		// Default to Claude Code runner
		log.Printf("✅ Using Claude Code runner for RFEWorkflow %s", workflowName)
		return config.AmbientCodeRunnerImage

	default:
		log.Printf("⚠️ Unknown runner '%s' in RFEWorkflow %s, using Claude runner", runner, workflowName)
		return config.AmbientCodeRunnerImage
	}
}
```

**Step 3: Configure LangGraph runner image**

```go
// MODIFY: /components/operator/internal/config/config.go

type Config struct {
	// ... existing fields
	AmbientCodeRunnerImage string  // Existing: Claude Code runner
	LangGraphRunnerImage   string  // NEW: LangGraph runner
}

func LoadConfig() *Config {
	return &Config{
		// ... existing config
		AmbientCodeRunnerImage: os.Getenv("AMBIENT_CODE_RUNNER_IMAGE"),
		LangGraphRunnerImage:   os.Getenv("LANGGRAPH_RUNNER_IMAGE"),
	}
}
```

**Step 4: Environment variables**

```yaml
# Operator deployment manifest
env:
  - name: AMBIENT_CODE_RUNNER_IMAGE
    value: quay.io/ambient-code/claude-runner:latest
  - name: LANGGRAPH_RUNNER_IMAGE
    value: quay.io/ambient-code/langgraph-runner:latest
```

---

## 4. State Management Patterns

### Checkpoint Storage Location

```bash
# PVC structure for RFE sessions
/workspace/sessions/
  {session-id}/
    workspace/          # Git repos + RFE outputs
      spec.md
      plan.md
      tasks.md
      .specify/
        templates/
          spec-template.md
          plan-template.md
          tasks-template.md
    .langgraph/         # Checkpoints (LangGraph runner only)
      checkpoints.db    # SQLite database
    .claude/            # Claude SDK state (Claude Code runner)
```

### Session Continuation Pattern

**Claude Code Pattern (current):**
```python
# From wrapper.py:219-228
if is_continuation and parent_session_id:
    sdk_resume_id = await self._get_sdk_session_id(parent_session_id)
    if sdk_resume_id:
        options.resume = sdk_resume_id
        options.fork_session = False
```

**LangGraph Pattern (new):**
```python
async def _resume_from_checkpoint(self, parent_session_id: str):
    """
    Resume RFE workflow from parent's checkpoint:
    1. Read parent's thread_id from CR annotation
    2. Load checkpoint from shared PVC path
    3. Resume graph execution from last completed node
    """
    parent_thread_id = await self._get_thread_id_from_annotation(parent_session_id)

    # Checkpointer reads from /workspace/sessions/{parent}/.langgraph/
    checkpoint = await self.checkpointer.aget(
        config={"configurable": {"thread_id": parent_thread_id}}
    )

    if checkpoint:
        return parent_thread_id  # Use same thread for continuation
    else:
        raise RuntimeError(f"No checkpoint found for {parent_session_id}")
```

---

## 5. Technical Risks & Mitigation

### Risk 1: Output Quality Divergence

**Problem:** LangGraph runner outputs might differ in quality or structure from Claude Code runner.

**Mitigation:**
- **Automated testing:** Check for required sections (Overview, Goals, etc.)
- **Manual validation:** Review 10+ sample outputs side-by-side
- **SpecKit team review:** Validate template loading and prompt conversion
- **Regression prevention:** Lock in test cases for output structure

### Risk 2: SpecKit Template Compatibility

**Problem:** Template parsing or prompt conversion might differ between runners.

**Mitigation:**
- **Reuse SpecKit logic:** Use same template parsing where possible
- **Test all templates:** Validate spec-template.md, plan-template.md, tasks-template.md
- **Claude API consistency:** Ensure both runners use identical API parameters
- **Template evolution:** Coordinate with SpecKit team on template format changes

### Risk 3: Checkpoint Storage Growth

**Problem:** Checkpoints accumulate over time, filling PVC.

**Mitigation:**
- **Automatic pruning:** Keep only last 10 checkpoints per thread
- **Monitoring:** PVC usage metrics and alerts
- **Documentation:** Clear guidance on checkpoint lifecycle
- **Future scaling:** Add Postgres checkpointer option for high-volume deployments

### Risk 4: Backward Compatibility

**Problem:** Existing RFE workflows must continue to work without modification.

**Mitigation:**
- **Default value:** `runner: claude-code` is default in CRD schema
- **Testing:** Run 50+ existing RFE workflows, verify no behavior changes
- **Gradual rollout:** Deploy LangGraph runner as opt-in initially
- **Version annotation:** Track runner type in session CR for troubleshooting

---

## 6. Implementation Phases

### Phase 1: Foundation (Week 1)

1. **CRD changes**
   - Add `runner` field to RFEWorkflow CRD
   - Set default value to `claude-code`
   - Update API validation

2. **Operator changes**
   - Add `getRunnerImageForSession()` function
   - Add `LANGGRAPH_RUNNER_IMAGE` config
   - Test with mock runner field values

3. **LangGraph adapter skeleton**
   - Create adapter.py with runner-shell interface
   - Basic message exchange (no graph execution yet)
   - Build container image

**Success criteria:** Can create RFEWorkflow with `runner: langgraph` and operator launches LangGraph container.

### Phase 2: RFE Workflow Implementation (Week 2)

1. **SpecKit integration**
   - Implement SpecKitTemplateLoader
   - Test template loading from `.specify/templates/`
   - Validate template parsing

2. **RFE graph**
   - Build specify → plan → tasks graph
   - Implement each node with Claude API calls
   - Write spec.md, plan.md, tasks.md files

3. **Checkpoint persistence**
   - Set up SQLite checkpointer
   - Test checkpoint save/load
   - Implement automatic pruning

4. **Session continuation**
   - Load checkpoint from parent session
   - Resume from last completed node
   - Verify workspace state preserved

**Success criteria:** RFE workflow executes successfully on LangGraph runner, outputs match Claude Code quality.

### Phase 3: Testing & Validation (Week 3)

1. **Output equivalence**
   - Automated structural validation
   - Manual quality review (10+ samples)
   - Side-by-side comparison

2. **Backward compatibility**
   - Run 50+ existing RFE workflows
   - Verify default behavior unchanged
   - Test with and without runner field

3. **Performance benchmarking**
   - Measure execution time vs Claude Code
   - Checkpoint I/O latency
   - PVC usage patterns

4. **Documentation**
   - User guide for runner selection
   - Operations guide for troubleshooting
   - API documentation updates

**Success criteria:** 100% backward compatibility, output equivalence validated, documentation published.

---

## 7. File Structure

```
vTeam/
├── components/
│   ├── runners/
│   │   ├── runner-shell/          # Unchanged
│   │   │   └── runner_shell/
│   │   │       ├── protocol.py
│   │   │       ├── transport_ws.py
│   │   │       └── shell.py
│   │   ├── claude-code-runner/    # Unchanged
│   │   │   ├── wrapper.py
│   │   │   └── Dockerfile
│   │   └── langgraph-runner/      # NEW
│   │       ├── adapter.py         # LangGraph adapter
│   │       ├── rfe_graph.py       # RFE workflow graph
│   │       ├── template_loader.py # SpecKit template parser
│   │       ├── checkpointer.py    # Checkpoint management
│   │       ├── streaming.py       # Output streaming
│   │       ├── observability.py   # Metrics tracking
│   │       ├── Dockerfile
│   │       └── requirements.txt
│   ├── backend/
│   │   ├── handlers/
│   │   │   ├── rfe.go             # MODIFY: accept runner field
│   │   │   └── sessions.go        # Unchanged
│   │   └── types/
│   │       └── rfe.go             # MODIFY: add runner field
│   ├── operator/
│   │   └── internal/
│   │       ├── config/
│   │       │   └── config.go      # MODIFY: add LangGraphRunnerImage
│   │       └── handlers/
│   │           └── sessions.go    # MODIFY: add getRunnerImageForSession()
│   └── manifests/
│       └── crds/
│           ├── rfeworkflows-crd.yaml    # MODIFY: add runner field
│           └── agenticsessions-crd.yaml # Unchanged
```

---

## 8. Open Questions for Discussion

1. **Template Parsing Strategy**: Should LangGraph runner parse SpecKit templates directly or subprocess to spec-kit CLI?
   - *Recommendation*: Parse directly for performance, maintain compatibility with SpecKit team

2. **Checkpoint Pruning Policy**: Keep last N checkpoints per session - what is appropriate N value?
   - *Recommendation*: Keep last 10 checkpoints; configurable via environment variable

3. **Default Runner Value**: Is explicit default (`claude-code`) in CRD schema sufficient?
   - *Recommendation*: Yes, explicit default clearer for users

4. **Postgres Checkpointer**: Should we support Postgres from MVP or post-MVP?
   - *Recommendation*: SQLite for MVP, Postgres as optional upgrade for high scale

5. **Output Validation**: How do we validate that LangGraph outputs are equivalent to Claude Code outputs?
   - *Recommendation*: Automated structure checks + manual quality review of 10+ samples

---

## 9. Recommendations Summary

### Must Have (MVP)

1. ✅ **Single CRD field** - Add `runner` to RFEWorkflow (enum: claude-code, langgraph)
2. ✅ **Operator image selection** - Dynamic runner image based on runner field
3. ✅ **Separate container images** - Avoid dependency conflicts
4. ✅ **SpecKit integration** - Load templates, produce identical outputs
5. ✅ **Checkpoint persistence** - SQLite with automatic pruning
6. ✅ **Session continuation** - Resume from parent checkpoint
7. ✅ **Backward compatibility** - Default to claude-code, existing workflows unchanged

### Should Have (Post-MVP)

1. ⚠️ **Postgres checkpointer** - Better performance for large state
2. ⚠️ **Frontend runner selection UI** - Dropdown for runner choice
3. ⚠️ **Advanced observability** - Prometheus metrics, node timing
4. ⚠️ **Checkpoint management tools** - CLI for inspecting/pruning checkpoints

### Nice to Have (Future)

1. 💡 **Human-in-the-loop nodes** - Pause for approval between stages
2. 💡 **Validation nodes** - Check spec/plan quality before proceeding
3. 💡 **Additional runners** - If use cases emerge (CrewAI, custom runners)

---

## Conclusion

The vTeam architecture is well-positioned for runner choice. The runner-shell abstraction provides clean separation between execution engines and platform infrastructure. The main implementation effort is:

1. Adding single `runner` field to RFEWorkflow CRD (20 lines YAML)
2. Operator image selection logic (50 lines Go)
3. LangGraph runner implementation (500 lines Python)
4. SpecKit template integration (100 lines Python)
5. Testing and validation

**Estimated effort:** 2-3 weeks for MVP with full RFE workflow support.

**Key success factors:**
1. Keep runner-shell interface stable
2. Use separate container images to avoid dependency conflicts
3. Ensure SpecKit template compatibility
4. Validate output equivalence rigorously
5. Maintain 100% backward compatibility

This design allows incremental rollout - we can deploy LangGraph runner as opt-in while keeping existing RFE workflows unchanged. The architecture supports adding more runner types in the future without major refactoring.

---

**Next Steps:**
1. Review with architecture team
2. SpecKit integration review with SpecKit team
3. Prototype Phase 1 in sandbox environment
4. Performance testing: checkpoint I/O, output equivalence
5. Security review if needed

---

**Prepared by:** Stella (Staff Engineer)
**Contact:** stella@ambient-code.io
**Date:** 2025-11-04
