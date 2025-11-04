# LangGraph Runner for RFE Workflow - Technical Specification

**Project:** Ambient Agentic Runner (vTeam) - Runner Selection for RFE Workflow
**Status:** Technical Design - Ready for Review
**Author:** Stella (Staff Engineer)
**Date:** 2025-11-04

---

## Overview

This specification package provides comprehensive technical guidance for adding a LangGraph runner option to the existing RFE workflow in the vTeam platform. The design enables users to choose between two execution engines (Claude Code or LangGraph) for the same RFE workflow, while maintaining 100% backward compatibility.

**Key Insight:** The existing runner-shell abstraction is well-architected for multi-runner support. We can add LangGraph runner with minimal changes to core infrastructure—just a single field in the RFEWorkflow CRD and operator image selection logic.

---

## Document Structure

This specification consists of four complementary documents:

### 1. [EXECUTIVE-SUMMARY.md](./EXECUTIVE-SUMMARY.md)
**Audience:** Architecture team, stakeholders, decision-makers

**Purpose:** High-level overview, business value, risk assessment, recommendations

**Key Sections:**
- Business value of runner choice
- Runner selection approach
- Implementation phases (2-3 weeks)
- Technical risks and mitigation
- Resource requirements
- Success metrics
- Decision required

**Read this if:** You need to understand the proposal at a strategic level and make go/no-go decisions.

---

### 2. [TECHNICAL-ARCHITECTURE.md](./TECHNICAL-ARCHITECTURE.md)
**Audience:** Staff engineers, architects, platform engineers

**Purpose:** Detailed technical design, component interactions, CRD specifications

**Key Sections:**
- Integration pattern (Runner-Shell abstraction)
- Runner field in RFEWorkflow CRD
- Runner lifecycle (CLI vs Graph execution)
- Operator image selection logic
- Technical risks and mitigation strategies
- Implementation phases with acceptance criteria
- File structure and code organization
- Complete code examples (adapters, operators)

**Read this if:** You need to understand the technical implementation details, review architecture decisions, or implement the solution.

---

### 3. [IMPLEMENTATION-PATTERNS.md](./IMPLEMENTATION-PATTERNS.md)
**Audience:** Senior/staff engineers, implementation team

**Purpose:** Concrete code patterns, best practices, production-ready implementations

**Key Patterns:**
1. **Template Loading** - SpecKit template parsing and integration
2. **Checkpoint Management** - SQLite with automatic pruning
3. **Streaming Output** - Chunking large outputs for WebSocket
4. **Observability & Metrics** - Execution tracking, debugging support
5. **Error Recovery** - Retry logic, backoff strategies
6. **Configuration Management** - Runner configuration options

**Read this if:** You're implementing the LangGraph runner and need battle-tested code patterns.

---

### 4. [QUICK-START.md](./QUICK-START.md)
**Audience:** Users creating RFE workflows with LangGraph runner

**Purpose:** Getting started guide, runner selection, troubleshooting

**Key Sections:**
- Creating RFE workflow with runner selection
- When to use each runner
- Configuration options
- Debugging tips
- Best practices
- Output comparison examples
- Troubleshooting guide

**Read this if:** You're creating RFE workflows and want to understand runner options.

---

## Recommended Reading Order

### For Decision Makers
1. Start with **EXECUTIVE-SUMMARY.md** (15 min read)
2. Review "Technical Risks" section in **TECHNICAL-ARCHITECTURE.md** (10 min)
3. Skim "Implementation Phases" for timeline understanding (5 min)

### For Implementation Team
1. Read **EXECUTIVE-SUMMARY.md** for context (10 min)
2. Deep dive into **TECHNICAL-ARCHITECTURE.md** (45 min)
3. Study **IMPLEMENTATION-PATTERNS.md** for code guidance (30 min)
4. Reference **QUICK-START.md** for user perspective (15 min)

### For Users Creating RFE Workflows
1. Start with **QUICK-START.md** (15 min)
2. Review runner selection criteria
3. Understand output equivalence

---

## Quick Reference

### Current Architecture

```
vTeam Platform (Current State)
├── Backend (Go)
│   ├── API Handlers
│   │   ├── RFE Workflows
│   │   └── Agentic Sessions
│   └── Types & CRDs
│       ├── RFEWorkflow
│       └── AgenticSession
├── Operator (Go)
│   ├── Watch AgenticSession CRs
│   ├── Create Kubernetes Jobs
│   └── Select Runner Image (Claude Code)
└── Runners
    ├── Runner-Shell (Python)
    │   ├── Protocol (WebSocket messages)
    │   ├── Transport (WebSocket)
    │   └── Context (Session management)
    └── Claude Code Runner (Python)
        ├── Wrapper (adapter.py)
        ├── SpecKit integration
        └── Git operations
```

### Proposed Architecture

```
vTeam Platform (With LangGraph Runner)
├── Backend (Go)
│   ├── API Handlers
│   │   ├── RFE Workflows (MODIFIED: accepts runner field)
│   │   └── Agentic Sessions (unchanged)
│   └── Types & CRDs
│       ├── RFEWorkflow (MODIFIED: runner enum field)
│       └── AgenticSession (unchanged)
├── Operator (Go)
│   ├── Watch AgenticSession CRs
│   ├── Create Kubernetes Jobs
│   └── Select Runner Image (MODIFIED: based on runner field)
│       ├── runner: "claude-code" → Claude Code Runner (default)
│       └── runner: "langgraph" → LangGraph Runner
└── Runners
    ├── Runner-Shell (Python) - UNCHANGED
    │   ├── Protocol (same interface)
    │   ├── Transport (same WebSocket)
    │   └── Context (same management)
    ├── Claude Code Runner (existing)
    │   └── ... (unchanged)
    └── LangGraph Runner (NEW)
        ├── Adapter (implements runner-shell interface)
        ├── RFE Graph (specify → plan → tasks)
        ├── SpecKit Template Loader
        ├── Checkpointer (SQLite persistence)
        └── Streaming (WebSocket output)
```

### Key Files to Modify

```
Existing Files (Modified):
  /components/operator/internal/handlers/sessions.go
    - Add getRunnerImageForSession() function
    - Select image based on parent RFEWorkflow's runner field

  /components/operator/internal/config/config.go
    - Add LangGraphRunnerImage field

  /components/manifests/crds/rfeworkflows-crd.yaml
    - Add spec.runner field (enum: claude-code, langgraph)
    - Default value: claude-code

  /components/backend/handlers/rfe.go
    - Accept runner field in RFEWorkflow creation
    - Validate runner enum values

New Files (Created):
  /components/runners/langgraph-runner/
    ├── adapter.py                   # LangGraph adapter
    ├── rfe_graph.py                 # RFE workflow graph (specify → plan → tasks)
    ├── template_loader.py           # SpecKit template parser
    ├── checkpointer.py              # Checkpoint management
    ├── streaming.py                 # Output streaming
    ├── observability.py             # Metrics tracking
    ├── error_handling.py            # Retry logic
    ├── Dockerfile                   # Container image
    └── requirements.txt             # Dependencies
```

---

## Design Principles

This design follows established patterns from the vTeam codebase:

1. **Interface Stability**: Runner-shell protocol remains unchanged - all runners implement the same interface
2. **Minimal CRD Changes**: Single field addition to existing RFEWorkflow CRD
3. **Backward Compatibility**: Existing RFE workflows continue to work without modification (default: claude-code)
4. **PVC-Centric State**: All session state (workspaces, checkpoints) persists on PVC
5. **Operator Pattern**: Operator watches CRs and creates Jobs with dynamic image selection
6. **WebSocket Transport**: Bidirectional communication via WebSocket (unchanged)
7. **Container Per Session**: Each session gets its own Job + Pod (unchanged)
8. **Output Equivalence**: Both runners produce identical spec.md, plan.md, tasks.md files

---

## Technical Highlights

### What Makes This Design Solid

1. **Zero Breaking Changes**: Existing RFE workflows continue to work. Default runner is claude-code.

2. **Clean Abstraction**: The runner-shell interface provides natural extension point. No hacks needed.

3. **Minimal Surface Area**: Single CRD field change. Operator image selection logic is 20 lines of code.

4. **State Management**: Checkpoints and workspaces both use PVC. Consistent storage pattern.

5. **SpecKit Compatibility**: LangGraph runner loads same templates, produces identical outputs.

6. **Operational Excellence**: Automatic checkpoint pruning prevents PVC exhaustion. Metrics for debugging.

7. **Performance**: Streaming with chunking handles large outputs. SQLite checkpointer is fast enough for MVP.

---

## Implementation Status

- [ ] Phase 1: Foundation (Week 1)
  - [ ] LangGraph adapter skeleton
  - [ ] Separate container image
  - [ ] Operator runner image selection
  - [ ] RFEWorkflow CRD runner field

- [ ] Phase 2: RFE Workflow Implementation (Week 2)
  - [ ] SpecKit template loading
  - [ ] RFE graph (specify → plan → tasks)
  - [ ] Checkpoint persistence (SQLite)
  - [ ] Session continuation
  - [ ] Error handling and retries

- [ ] Phase 3: Testing & Validation (Week 3)
  - [ ] Output equivalence verification
  - [ ] Backward compatibility testing
  - [ ] Performance benchmarking
  - [ ] Documentation and examples

---

## Success Criteria

### Technical Validation
- ✅ Can create RFEWorkflow with `runner: langgraph`
- ✅ Operator launches correct runner image based on runner field
- ✅ LangGraph executes RFE workflow (specify → plan → tasks)
- ✅ Checkpoints persist and sessions resume from parent
- ✅ SpecKit templates loaded correctly
- ✅ Output files (spec.md, plan.md, tasks.md) match Claude Code quality
- ✅ 100% backward compatibility with existing RFE workflows
- ✅ No regression in existing functionality

### Performance Targets
- RFE workflow execution overhead < 10% vs Claude Code
- Checkpoint save/load latency < 100ms
- WebSocket message latency < 50ms
- Support 10+ concurrent RFE sessions per node

### Output Equivalence
- ✅ spec.md structure matches Claude Code output
- ✅ plan.md structure matches Claude Code output
- ✅ tasks.md structure matches Claude Code output
- ✅ Content quality equivalent (validated by manual review)

---

## Open Questions & Next Steps

### Questions for Review
1. **Checkpointer Backend**: Is SQLite sufficient or need Postgres option for high scale?
2. **Checkpoint Pruning**: Keep last 10 checkpoints adequate or adjust based on usage?
3. **Template Parsing**: Should we parse SpecKit templates directly or subprocess to spec-kit CLI?
4. **Observability**: What metrics should we expose to Prometheus?

### Next Actions
1. **Schedule architecture review** (2 hours, all staff engineers)
2. **SpecKit integration review** (1 hour with SpecKit team)
3. **Prototype Phase 1** (1-week sprint, 1 engineer)
4. **Performance testing** (checkpoint I/O, output equivalence validation)

---

## Contact & Support

**Technical Lead:** Stella (Staff Engineer)
**Email:** stella@ambient-code.io
**Slack:** #vteam-architecture

**Document Feedback:** Open a PR against this repo
**Technical Questions:** Post in #vteam-engineering

---

## Version History

- **v2.0** (2025-11-04): Revised to simplified scope
  - Single RFE workflow with runner choice
  - No new CRDs - runner field in existing RFEWorkflow
  - LangGraph implements RFE workflow graph
  - Focus on output equivalence
- **v1.0** (2025-11-04): Initial technical specification (multi-workflow scope)

---

**Status:** 🟢 Ready for Review

This specification reflects the simplified scope: one workflow (RFE), two runner options (Claude Code, LangGraph). All technical decisions are documented with rationale. Implementation path is clear with 2-3 week timeline.
