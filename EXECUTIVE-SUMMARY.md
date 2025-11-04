# Executive Summary: LangGraph Workflow Integration

**Document Type:** Technical RFE Summary
**Author:** Stella (Staff Engineer)
**Date:** 2025-11-04
**Stakeholders:** Architecture Team, Product Management, Platform Engineering

---

## Overview

This document summarizes the technical architecture and implementation approach for adding LangGraph-based workflow support to the Ambient Agentic Runner (vTeam) platform. The proposal enables the platform to support multiple execution engines while maintaining backward compatibility with existing RFE workflows.

---

## Business Value

### What This Enables

1. **Workflow Diversity**: Support for declarative graph-based workflows alongside imperative CLI-based workflows
2. **Developer Choice**: Teams can choose the best execution model for their use case
3. **Advanced Patterns**: Enable complex agent coordination patterns (multi-agent, hierarchical, human-in-the-loop)
4. **Future Extensibility**: Foundation for adding more runner types (CrewAI, AutoGPT, custom runners)

### Use Cases Unlocked

- **Multi-step approval workflows** with graph-based state machines
- **Parallel agent execution** for concurrent task processing
- **Complex decision trees** with conditional branching
- **Long-running workflows** with persistent checkpointing
- **Human-in-the-loop patterns** with structured interrupts

---

## Technical Approach

### Core Architecture Decision

**Pattern: Multi-Runner Architecture with Adapter Pattern**

The existing runner-shell abstraction is well-designed and requires **no breaking changes**. We extend the platform by:

1. Creating a new LangGraph adapter that implements the same interface
2. Building a separate container image with LangGraph dependencies
3. Extending the operator to select runner image based on workflow type
4. Adding a new LangGraphWorkflow CRD parallel to RFEWorkflow

```
Current:
  RFEWorkflow → AgenticSession → Operator → Claude Code Runner

Proposed:
  RFEWorkflow → AgenticSession → Operator → Claude Code Runner
  LangGraphWorkflow → AgenticSession → Operator → LangGraph Runner
```

### Key Technical Decisions

| Decision | Rationale | Risk Mitigation |
|----------|-----------|-----------------|
| **Separate container images** | Avoid dependency conflicts between Claude SDK and LangGraph | Multi-stage Docker builds, pin versions |
| **CRD-per-workflow pattern** | Type safety, clear separation of concerns | Document migration path to generic template if needed |
| **SQLite for checkpoints** | Simple, no external dependencies | Add Postgres option for high-scale deployments |
| **PVC for all state** | Consistent with current architecture | Implement checkpoint pruning to manage disk usage |
| **Backward compatible** | Existing workflows continue to work unchanged | Default to RFE runner if workflowType not specified |

---

## Architecture Highlights

### 1. Runner Lifecycle Comparison

| Aspect | Claude Code (Current) | LangGraph (New) |
|--------|----------------------|-----------------|
| Execution Model | Interactive CLI process | Declarative graph traversal |
| State Persistence | `.claude` directory (SDK managed) | `.langgraph/checkpoints.db` |
| Resumption | SDK's built-in `resume` option | Thread ID + checkpoint lookup |
| Human-in-the-loop | Interactive message queue | Graph interrupt nodes |
| Streaming | Line-by-line terminal output | Node execution events |

### 2. Workflow Type Selection

The operator determines which runner image to use based on the `workflowType` field in AgenticSession:

```go
func getRunnerImageForSession(spec map[string]interface{}) string {
    workflowType := spec["workflowType"]

    switch workflowType {
    case "langgraph":
        return config.LangGraphRunnerImage
    case "rfe", "":  // Default for backward compatibility
        return config.AmbientCodeRunnerImage
    }
}
```

### 3. State Management

**Current Pattern (Claude Code):**
```
/workspace/sessions/{session-id}/
  workspace/      # Git repos
  .claude/        # SDK session state
```

**New Pattern (LangGraph):**
```
/workspace/sessions/{session-id}/
  workspace/           # Git repos
  .langgraph/          # Checkpoints
    checkpoints.db     # SQLite (default)
    thread_{id}.json   # Thread metadata
```

Both patterns coexist on the same PVC without conflict.

---

## Implementation Plan

### Phase 1: Foundation (Weeks 1-2)
**Goal:** Basic infrastructure in place

- Create LangGraph adapter skeleton
- Build separate container image
- Extend operator to select runner image
- Test with minimal graph execution

**Success Criteria:**
- Can create AgenticSession with `workflowType: langgraph`
- Operator launches LangGraph runner container
- Basic graph executes and sends messages

### Phase 2: Core Execution (Weeks 3-4)
**Goal:** Full graph execution with state management

- Implement graph loading from workspace
- SQLite checkpointer to PVC
- Session continuation (resume from checkpoint)
- Error handling and retries

**Success Criteria:**
- Complex graphs execute successfully
- State persists across pod restarts
- Can resume from parent session

### Phase 3: Human-in-the-Loop (Week 5)
**Goal:** Interactive workflows

- Interrupt handling at graph nodes
- WebSocket bidirectional communication
- Input timeout and fallback behavior
- Multi-turn conversations

**Success Criteria:**
- Graph pauses at interrupt nodes
- User provides input via UI
- Graph resumes with user input

### Phase 4: API & CRDs (Week 6)
**Goal:** Production-ready platform

- LangGraphWorkflow CRD schema
- Backend API handlers (CRUD)
- Frontend integration
- Documentation and examples

**Success Criteria:**
- Full lifecycle: create workflow → execute → monitor → resume
- End-to-end tests pass
- Documentation published

---

## Technical Risks & Mitigation

### Risk 1: Dependency Conflicts
**Impact:** High | **Probability:** Medium

**Mitigation:**
- Use separate container images (one per runner type)
- Pin all dependency versions
- CI tests for both runners in isolation

### Risk 2: Checkpoint Storage Growth
**Impact:** Medium | **Probability:** High

**Mitigation:**
- Implement automatic checkpoint pruning
- Keep only last N checkpoints per thread
- Monitor PVC usage metrics
- Add PVC size alerts

### Risk 3: Graph Security
**Impact:** High | **Probability:** Medium

**Mitigation:**
- AST validation before graph execution
- Whitelist allowed imports
- Block dangerous operations (exec, eval)
- Code review required for new graph templates

### Risk 4: Migration Complexity
**Impact:** Medium | **Probability:** Low

**Mitigation:**
- Maintain full backward compatibility
- Default to RFE runner if workflowType not specified
- Gradual rollout (opt-in initially)
- Comprehensive migration guide

---

## Resource Requirements

### Development Effort
- **Total:** 6-8 weeks (1 senior engineer)
- **Phase 1:** 2 weeks (foundation)
- **Phase 2:** 2 weeks (core execution)
- **Phase 3:** 1 week (human-in-the-loop)
- **Phase 4:** 1-2 weeks (API/CRDs)
- **Buffer:** 1 week (testing, docs, polish)

### Infrastructure
- **Container Registry:** Space for additional runner image (~500MB)
- **PVC Storage:** No increase (checkpoints stored in existing session PVCs)
- **Compute:** Same as current (one runner pod per session)
- **Testing Environment:** Sandbox cluster for prototyping

### Knowledge Transfer
- **Architecture Review:** 2 hours (with platform team)
- **Security Review:** 2 hours (with security team)
- **Developer Training:** 4 hours (how to build graphs)
- **Operations Training:** 2 hours (troubleshooting)

---

## Success Metrics

### Technical Metrics
- **Compatibility:** 100% of existing RFE workflows continue to work unchanged
- **Performance:** LangGraph execution latency < 10% overhead vs Claude Code
- **Reliability:** Graph execution success rate > 95%
- **Storage:** Checkpoint pruning keeps PVC growth < 10% per week

### Adoption Metrics
- **Week 4:** MVP deployed to development environment
- **Week 6:** First production LangGraph workflow
- **Week 8:** 3+ teams experimenting with LangGraph
- **Month 3:** 20% of new workflows use LangGraph

---

## Open Questions

### For Architecture Team
1. Should we support Python-only graphs or also JSON/YAML declarative format?
2. What's the long-term vision for workflow types? (5+? Consolidate?)
3. Do we need a shared graph library/registry?

### For Security Team
1. Is AST validation sufficient for graph definitions?
2. Should we require code review for all graph templates?
3. Any additional sandboxing needed?

### For Product Team
1. How should we position LangGraph vs RFE workflows to customers?
2. What's the migration story for existing RFE users?
3. Do we need a visual graph builder? (Phase 2?)

### For Operations Team
1. What's the PVC sizing recommendation for checkpoints?
2. How should we handle checkpoint backups?
3. What alerts do we need for graph execution?

---

## Related Documents

1. **TECHNICAL-ARCHITECTURE.md** - Detailed technical design, component interactions, API specifications
2. **IMPLEMENTATION-PATTERNS.md** - Code patterns, best practices, security considerations
3. **RFEWorkflow CRD** - `/workspace/sessions/agentic-session-1762286246/workspace/vTeam/components/manifests/crds/rfeworkflows-crd.yaml`
4. **AgenticSession CRD** - `/workspace/sessions/agentic-session-1762286246/workspace/vTeam/components/manifests/crds/agenticsessions-crd.yaml`

---

## Recommendations

### For MVP (Must Have)
✅ **Approve:**
- Separate container images for runner isolation
- LangGraphWorkflow CRD (parallel to RFEWorkflow)
- SQLite checkpointer with pruning
- Operator workflow type selection
- Backward compatibility (default to RFE)

### Post-MVP (Should Have)
⚠️ **Consider:**
- Postgres checkpointer for high-scale workloads
- Visual graph definition editor
- Graph testing framework
- Workflow marketplace for sharing graphs

### Future Exploration (Nice to Have)
💡 **Explore:**
- Unified WorkflowTemplate CRD (if 5+ workflow types)
- Cross-session graph sharing/composition
- Graph performance profiling tools
- Multi-region checkpoint replication

---

## Decision Required

**Recommendation:** Proceed with implementation as outlined.

**Rationale:**
1. Architecture is sound and extensible
2. Minimal risk to existing functionality
3. Clear implementation path with 6-8 week timeline
4. Unlocks significant new use cases
5. Foundation for future runner types

**Next Steps:**
1. Architecture review meeting (schedule within 1 week)
2. Security review of graph validation approach
3. Approve Phase 1 implementation (2-week sprint)
4. Assign engineer and provision sandbox environment

---

**Prepared by:** Stella (Staff Engineer)
**Contact:** stella@ambient-code.io
**Date:** 2025-11-04
