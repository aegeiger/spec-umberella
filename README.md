# LangGraph Workflow Integration - Technical Specification

**Project:** Ambient Agentic Runner (vTeam) - Multi-Workflow Support
**Status:** Technical Design - Ready for Review
**Author:** Stella (Staff Engineer)
**Date:** 2025-11-04

---

## Overview

This specification package provides comprehensive technical guidance for adding LangGraph-based workflow support to the vTeam platform. The design enables multiple execution engines (Claude Code, LangGraph, future runners) while maintaining backward compatibility with existing RFE workflows.

**Key Insight:** The existing runner-shell abstraction is well-architected for multi-runner support. We can add LangGraph with minimal changes to core infrastructure.

---

## Document Structure

This specification consists of four complementary documents:

### 1. [EXECUTIVE-SUMMARY.md](./EXECUTIVE-SUMMARY.md)
**Audience:** Architecture team, stakeholders, decision-makers

**Purpose:** High-level overview, business value, risk assessment, recommendations

**Key Sections:**
- Business value and use cases
- Architecture approach and key decisions
- Implementation phases (6-8 weeks)
- Technical risks and mitigation
- Resource requirements
- Success metrics
- Decision required

**Read this if:** You need to understand the proposal at a strategic level and make go/no-go decisions.

---

### 2. [TECHNICAL-ARCHITECTURE.md](./TECHNICAL-ARCHITECTURE.md)
**Audience:** Staff engineers, architects, platform engineers

**Purpose:** Detailed technical design, component interactions, API specifications

**Key Sections:**
- Integration pattern (Runner-Shell abstraction)
- Workflow extensibility (CRD design)
- Runner lifecycle (CLI vs Graph execution)
- API & CRD design
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
1. **Graph Loading & Validation** - Secure Python AST parsing, import whitelisting
2. **Checkpoint Management** - SQLite with automatic pruning
3. **Human-in-the-Loop** - Timeout handling, fallback behavior
4. **Streaming Output** - Chunking large outputs for WebSocket
5. **Observability & Metrics** - Execution tracking, debugging support
6. **Error Recovery** - Retry logic, backoff strategies
7. **Configuration Management** - YAML-based graph configuration

**Read this if:** You're implementing the LangGraph runner and need battle-tested code patterns.

---

### 4. [QUICK-START.md](./QUICK-START.md)
**Audience:** Developers building LangGraph workflows

**Purpose:** Getting started guide, common patterns, troubleshooting

**Key Sections:**
- Creating your first graph
- Configuration options
- Common patterns (branching, parallel, retry)
- Debugging tips
- Best practices
- Example workflows
- Troubleshooting guide

**Read this if:** You're a developer building workflows on vTeam and need practical examples.

---

## Recommended Reading Order

### For Decision Makers
1. Start with **EXECUTIVE-SUMMARY.md** (15 min read)
2. Review "Technical Risks" section in **TECHNICAL-ARCHITECTURE.md** (10 min)
3. Skim "Implementation Phases" for timeline understanding (5 min)

### For Implementation Team
1. Read **EXECUTIVE-SUMMARY.md** for context (15 min)
2. Deep dive into **TECHNICAL-ARCHITECTURE.md** (60 min)
3. Study **IMPLEMENTATION-PATTERNS.md** for code guidance (45 min)
4. Reference **QUICK-START.md** for user perspective (20 min)

### For Developers Building Workflows
1. Start with **QUICK-START.md** (20 min)
2. Reference specific patterns in **IMPLEMENTATION-PATTERNS.md** as needed
3. Consult **TECHNICAL-ARCHITECTURE.md** for advanced topics

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
        ├── Claude SDK integration
        └── Git operations
```

### Proposed Architecture

```
vTeam Platform (With LangGraph)
├── Backend (Go)
│   ├── API Handlers
│   │   ├── RFE Workflows (existing)
│   │   ├── LangGraph Workflows (NEW)
│   │   └── Agentic Sessions (extended)
│   └── Types & CRDs
│       ├── RFEWorkflow (existing)
│       ├── LangGraphWorkflow (NEW)
│       └── AgenticSession (workflowType field added)
├── Operator (Go)
│   ├── Watch AgenticSession CRs
│   ├── Create Kubernetes Jobs
│   └── Select Runner Image (NEW: based on workflowType)
│       ├── workflowType: "rfe" → Claude Code Runner
│       └── workflowType: "langgraph" → LangGraph Runner
└── Runners
    ├── Runner-Shell (Python) - UNCHANGED
    │   ├── Protocol (same interface)
    │   ├── Transport (same WebSocket)
    │   └── Context (same management)
    ├── Claude Code Runner (existing)
    │   └── ... (unchanged)
    └── LangGraph Runner (NEW)
        ├── Adapter (implements runner-shell interface)
        ├── Graph Loader (secure loading)
        ├── Checkpointer (SQLite persistence)
        └── HITL Manager (human-in-the-loop)
```

### Key Files to Modify

```
Existing Files (Modified):
  /components/operator/internal/handlers/sessions.go
    - Add getRunnerImageForWorkflow() function
    - Select image based on spec.workflowType

  /components/operator/internal/config/config.go
    - Add LangGraphRunnerImage field

  /components/manifests/crds/agenticsessions-crd.yaml
    - Add spec.workflowType field (enum: rfe, langgraph)
    - Add spec.workflowRef field

New Files (Created):
  /components/runners/langgraph-runner/
    ├── adapter.py                   # LangGraph adapter
    ├── graph_loader.py              # Secure graph loading
    ├── checkpointer.py              # Checkpoint management
    ├── hitl_manager.py              # Human-in-the-loop
    ├── streaming.py                 # Output streaming
    ├── observability.py             # Metrics tracking
    ├── error_handling.py            # Retry logic
    ├── config_manager.py            # Configuration
    ├── Dockerfile                   # Container image
    └── requirements.txt             # Dependencies

  /components/manifests/crds/langgraphworkflows-crd.yaml
    # New CRD for LangGraph workflows

  /components/backend/handlers/langgraph.go
    # API handlers for LangGraph workflows

  /components/backend/types/langgraph.go
    # Go types for LangGraph
```

---

## Design Principles

This design follows established patterns from the vTeam codebase:

1. **Interface Stability**: Runner-shell protocol remains unchanged - all runners implement the same interface
2. **Separation of Concerns**: Each workflow type has its own CRD, handlers, and runner implementation
3. **Backward Compatibility**: Existing RFE workflows continue to work without modification
4. **PVC-Centric State**: All session state (workspaces, checkpoints) persists on PVC
5. **Operator Pattern**: Operator watches CRs and creates Jobs, no changes to this pattern
6. **WebSocket Transport**: Bidirectional communication via WebSocket (unchanged)
7. **Container Per Session**: Each session gets its own Job + Pod (unchanged)

---

## Technical Highlights

### What Makes This Design Solid

1. **Zero Breaking Changes**: Existing RFE workflows continue to work. Default behavior unchanged.

2. **Clean Abstraction**: The runner-shell interface provides natural extension point. No hacks needed.

3. **Type Safety**: CRD-per-workflow pattern gives us compile-time validation and clear schemas.

4. **State Management**: Checkpoints and workspaces both use PVC. Consistent storage pattern.

5. **Security First**: Graph loading includes AST validation, import whitelisting, dangerous operation blocking.

6. **Operational Excellence**: Automatic checkpoint pruning prevents PVC exhaustion. Metrics for debugging.

7. **Performance**: Streaming with chunking handles large outputs. SQLite checkpointer is fast enough for MVP.

---

## Implementation Status

- [ ] Phase 1: Foundation (Weeks 1-2)
  - [ ] LangGraph adapter skeleton
  - [ ] Separate container image
  - [ ] Operator workflow type selection
  - [ ] Basic graph execution test

- [ ] Phase 2: Core Execution (Weeks 3-4)
  - [ ] Graph loading with security validation
  - [ ] Checkpoint persistence (SQLite)
  - [ ] Session continuation
  - [ ] Error handling and retries

- [ ] Phase 3: Human-in-the-Loop (Week 5)
  - [ ] Interrupt handling
  - [ ] WebSocket bidirectional communication
  - [ ] Input timeout and fallback
  - [ ] Multi-turn conversations

- [ ] Phase 4: API & CRDs (Week 6)
  - [ ] LangGraphWorkflow CRD
  - [ ] Backend API handlers
  - [ ] Frontend integration
  - [ ] Documentation and examples

---

## Success Criteria

### Technical Validation
- ✅ Can create AgenticSession with `workflowType: langgraph`
- ✅ Operator launches correct runner image based on workflow type
- ✅ LangGraph executes complex graphs end-to-end
- ✅ Checkpoints persist and sessions resume from parent
- ✅ Human-in-the-loop works (pause, wait, resume)
- ✅ 100% backward compatibility with RFE workflows
- ✅ No regression in existing functionality

### Performance Targets
- Graph execution overhead < 10% vs Claude Code
- Checkpoint save/load latency < 100ms
- WebSocket message latency < 50ms
- Support 10+ concurrent graph executions per node

### Security Checklist
- ✅ Graph AST validation blocks dangerous operations
- ✅ Import whitelist enforced
- ✅ No arbitrary code execution paths
- ✅ Checkpoint data encrypted at rest (PVC encryption)

---

## Open Questions & Next Steps

### Questions for Review
1. **Graph Definition Format**: Python-only or also JSON/YAML?
2. **Checkpointer Backend**: SQLite sufficient or need Postgres option?
3. **Security**: Is AST validation adequate or need additional sandboxing?
4. **Observability**: What metrics should we expose to Prometheus?

### Next Actions
1. **Schedule architecture review** (2 hours, all staff engineers)
2. **Security review** (2 hours, security team)
3. **Prototype Phase 1** (2-week sprint, 1 engineer)
4. **Performance testing** (checkpoint I/O, concurrent sessions)

---

## Contact & Support

**Technical Lead:** Stella (Staff Engineer)
**Email:** stella@ambient-code.io
**Slack:** #vteam-architecture

**Document Feedback:** Open a PR against this repo
**Technical Questions:** Post in #vteam-engineering

---

## Version History

- **v1.0** (2025-11-04): Initial technical specification
  - Complete architecture design
  - Implementation patterns
  - Quick start guide
  - Executive summary

---

**Status:** 🟢 Ready for Review

This specification is complete and ready for architecture review. All technical decisions are documented with rationale. Implementation path is clear with concrete milestones.
