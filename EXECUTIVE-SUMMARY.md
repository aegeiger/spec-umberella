# Executive Summary: LangGraph Runner for RFE Workflow

**Document Type:** Technical RFE Summary
**Author:** Stella (Staff Engineer)
**Date:** 2025-11-04
**Stakeholders:** Architecture Team, Product Management, Platform Engineering

---

## Overview

This document summarizes the technical architecture and implementation approach for adding a LangGraph runner option to the existing RFE workflow in the Ambient Agentic Runner (vTeam) platform. The proposal enables users to choose their execution engine (Claude Code or LangGraph) while maintaining 100% backward compatibility.

---

## Business Value

### What This Enables

1. **Runner Choice**: Users select execution model (CLI-based vs graph-based) for RFE workflows
2. **Checkpointed Execution**: LangGraph runner provides pause/resume capability at any RFE stage
3. **Enhanced Observability**: Structured graph execution events for better debugging
4. **Foundation for Future Features**: Enables human-in-the-loop review gates, validation nodes (post-MVP)

### Use Cases Unlocked

- **Resume from checkpoint** after workflow interruption (network issue, pod restart)
- **Structured execution traces** showing which RFE stage (specify, plan, tasks) is active
- **Pause/resume workflows** for review before proceeding to next stage (post-MVP)
- **Consistent outputs** regardless of runner choice - same spec.md, plan.md, tasks.md

---

## Technical Approach

### Core Architecture Decision

**Pattern: Runner Selection via CRD Field**

The existing RFEWorkflow CRD and runner-shell abstraction are well-designed and require **minimal changes**. We extend the platform by:

1. Adding a `runner` enum field to RFEWorkflow CRD (claude-code | langgraph)
2. Modifying operator to read runner field and select appropriate container image
3. Creating LangGraph runner that implements same RFE workflow (specify → plan → tasks)
4. Ensuring both runners load SpecKit templates and produce identical outputs

```
Current:
  RFEWorkflow (no runner field) → Operator → Claude Code Runner

Proposed:
  RFEWorkflow { runner: "claude-code" } → Operator → Claude Code Runner (default)
  RFEWorkflow { runner: "langgraph" } → Operator → LangGraph Runner (new)
```

### Key Technical Decisions

| Decision | Rationale | Risk Mitigation |
|----------|-----------|-----------------|
| **Single CRD field** | Minimal change to existing infrastructure | Default to claude-code for backward compatibility |
| **Separate container images** | Avoid dependency conflicts between Claude SDK and LangGraph | Multi-stage Docker builds, pin versions |
| **Same RFE workflow** | Both runners execute specify → plan → tasks | SpecKit template compatibility, output validation |
| **SQLite for checkpoints** | Simple, no external dependencies | Add Postgres option for high-scale deployments |
| **Output equivalence** | Users see no difference in quality | Automated testing, manual review sampling |

---

## Implementation Plan

### Phase 1: Foundation (Week 1)
**Goal:** Basic infrastructure in place

- Add `runner` field to RFEWorkflow CRD
- Extend operator to select runner image based on runner field
- Create LangGraph adapter skeleton
- Build separate container image

**Success Criteria:**
- Can create RFEWorkflow with `runner: langgraph`
- Operator launches LangGraph runner container
- Basic message exchange via WebSocket works

### Phase 2: RFE Workflow Implementation (Week 2)
**Goal:** Full RFE workflow execution with checkpointing

- Implement SpecKit template loading in LangGraph runner
- Build RFE graph (specify → plan → tasks nodes)
- SQLite checkpointer to PVC
- Session continuation (resume from checkpoint)
- Error handling and retries

**Success Criteria:**
- RFE workflow executes successfully on LangGraph runner
- Outputs (spec.md, plan.md, tasks.md) match Claude Code quality
- Checkpoints persist across pod restarts
- Can resume from parent session

### Phase 3: Testing & Validation (Week 3)
**Goal:** Production readiness

- Output equivalence testing (automated + manual)
- Backward compatibility verification
- Performance benchmarking
- Documentation and examples

**Success Criteria:**
- 100% of existing RFE workflows work unchanged
- Output quality validated across 10+ test cases
- Performance within 10% of Claude Code runner
- Documentation published

---

## Technical Risks & Mitigation

### Risk 1: Output Quality Divergence
**Impact:** High | **Probability:** Medium

**Mitigation:**
- Automated structural validation (check for required sections)
- Manual quality review of 10+ sample outputs
- Side-by-side comparison testing
- SpecKit team review of template loading logic

### Risk 2: SpecKit Template Compatibility
**Impact:** High | **Probability:** Low

**Mitigation:**
- Reuse SpecKit template parsing logic where possible
- Test with all existing RFE templates
- Validate template → prompt conversion
- Ensure Claude API calls identical between runners

### Risk 3: Checkpoint Storage Growth
**Impact:** Medium | **Probability:** High

**Mitigation:**
- Implement automatic checkpoint pruning
- Keep only last 10 checkpoints per thread
- Monitor PVC usage metrics
- Add PVC size alerts

### Risk 4: Dependency Conflicts
**Impact:** Medium | **Probability:** Low

**Mitigation:**
- Use separate container images (one per runner type)
- Pin all dependency versions
- CI tests for both runners in isolation
- No shared Python environment

---

## Resource Requirements

### Development Effort
- **Total:** 2-3 weeks (1 senior engineer)
- **Phase 1:** 1 week (foundation)
- **Phase 2:** 1 week (RFE implementation)
- **Phase 3:** 1 week (testing & validation)

### Infrastructure
- **Container Registry:** Space for additional runner image (~500MB)
- **PVC Storage:** No increase (checkpoints stored in existing session PVCs)
- **Compute:** Same as current (one runner pod per session)
- **Testing Environment:** Sandbox cluster for prototyping

### Knowledge Transfer
- **Architecture Review:** 2 hours (with platform team)
- **SpecKit Integration Review:** 1 hour (with SpecKit team)
- **Operations Training:** 1 hour (troubleshooting, checkpoints)

---

## Success Metrics

### Technical Metrics
- **Compatibility:** 100% of existing RFE workflows continue to work unchanged
- **Performance:** LangGraph execution latency < 10% overhead vs Claude Code
- **Reliability:** RFE execution success rate > 95% (match Claude Code)
- **Storage:** Checkpoint pruning keeps PVC growth < 10% per week

### Adoption Metrics
- **Week 2:** MVP deployed to development environment
- **Week 3:** First production RFE workflow with LangGraph runner
- **Month 1:** 3+ teams experimenting with LangGraph runner
- **Month 3:** 10% of new RFE workflows opt into LangGraph

---

## Open Questions

### For Architecture Team
1. Is SQLite sufficient for checkpoint storage or should we support Postgres from day 1?
2. Should checkpoint pruning be configurable per-workflow or global setting?
3. What's the long-term vision for runner types? (Add more beyond Claude Code + LangGraph?)

### For SpecKit Team
1. Should LangGraph runner parse templates directly or subprocess to spec-kit CLI?
2. Are there template features that won't work with LangGraph runner?
3. How do we ensure template → Claude API prompt conversion is identical?

### For Operations Team
1. What's the PVC sizing recommendation for checkpoints?
2. How should we handle checkpoint backups?
3. What alerts do we need for checkpoint database issues?

---

## Related Documents

1. **TECHNICAL-ARCHITECTURE.md** - Detailed technical design, component interactions, CRD specifications
2. **IMPLEMENTATION-PATTERNS.md** - Code patterns, best practices, SpecKit integration
3. **QUICK-START.md** - User guide for runner selection
4. **RFEWorkflow CRD** - `/workspace/sessions/agentic-session-1762286246/workspace/vTeam/components/manifests/crds/rfeworkflows-crd.yaml`

---

## Recommendations

### For MVP (Must Have)
✅ **Approve:**
- Separate container images for runner isolation
- `runner` field in RFEWorkflow CRD (enum: claude-code, langgraph)
- SQLite checkpointer with pruning
- Operator runner image selection based on runner field
- Backward compatibility (default to claude-code)
- Output equivalence validation

### Post-MVP (Should Have)
⚠️ **Consider:**
- Postgres checkpointer for high-scale workloads
- Human-in-the-loop interrupt nodes (pause for review)
- Validation nodes (check spec completeness before plan)
- Frontend UI for runner selection

### Future Exploration (Nice to Have)
💡 **Explore:**
- Additional runner types (if use cases emerge)
- Cross-runner benchmarking tools
- Checkpoint replication for disaster recovery

---

## Decision Required

**Recommendation:** Proceed with implementation as outlined.

**Rationale:**
1. Architecture is sound with minimal changes to existing infrastructure
2. Minimal risk to existing functionality (default behavior unchanged)
3. Clear implementation path with 2-3 week timeline
4. Unlocks checkpointed execution and observable workflows
5. Foundation for future enhancements (human-in-the-loop, validation gates)

**Next Steps:**
1. Architecture review meeting (schedule within 1 week)
2. SpecKit integration review (coordinate with SpecKit team)
3. Approve Phase 1 implementation (1-week sprint)
4. Assign engineer and provision sandbox environment

---

**Prepared by:** Stella (Staff Engineer)
**Contact:** stella@ambient-code.io
**Date:** 2025-11-04
