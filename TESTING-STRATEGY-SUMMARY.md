# Testing Strategy Summary: LangGraph Runner Support

**Neil's Testing Strategy for vTeam LangGraph Integration**

---

## Executive Summary for Leadership

### The Testing Challenge
Adding LangGraph runner support to vTeam requires extensive testing to ensure:
1. **Contract Compliance** - LangGraphAdapter implements exact same interface as ClaudeCodeAdapter
2. **Zero Regression** - Claude Code runner unchanged (FR-022 critical requirement)
3. **Infrastructure Integration** - Operator correctly selects runner image via runnerType field
4. **Production Readiness** - Works across Standard, FIPS, Disconnected, and GPU clusters

### My Testing Strategy (High Level)

```
Risk Assessment → Test Planning → Implementation → Validation
      ↓                ↓              ↓              ↓
Identify 7 areas → Design 6 categories → Implement → Pre-deployment
of concern      of testing          automated   checklist
                                    tests
```

---

## 1. Testing Strategy Delivered

### What I Defined

**Three comprehensive documents:**

1. **LangGraph-Testing-Strategy.md** (Main Document - 800+ lines)
   - Complete testing framework for all 6 categories
   - 2,000+ lines of pseudocode test examples
   - CI/CD pipeline integration approach
   - Success metrics and release gates

2. **LangGraph-Test-Implementation-Guide.md** (Practical Implementation - 600+ lines)
   - Copy-paste ready pytest code
   - Fixture definitions for common scenarios
   - Real test class implementations
   - GitHub Actions workflow example

3. **TESTING-STRATEGY-SUMMARY.md** (This Document)
   - Executive summary for quick understanding
   - Implementation roadmap
   - Key decisions and their rationale
   - FAQ for common questions

### Who Should Read What

| Role | Read First | Then | Purpose |
|------|-----------|------|---------|
| QA Engineers | Implementation Guide | Testing Strategy | Understand how to run tests |
| Developers | Testing Strategy | Implementation Guide | Understand what's tested |
| DevOps | Testing Strategy (CI/CD section) | Implementation Guide | Understand pipeline integration |
| Leadership | This summary | Testing Strategy (metrics) | Understand coverage and timing |

---

## 2. Six Categories of Testing

### Category 1: Contract Testing (Interface Compliance)

**What:** Verify LangGraphAdapter and ClaudeCodeAdapter have identical interfaces

**Why:** Ensures polymorphic runner selection works; adapter can be swapped without breaking system

**How:**
```python
# Validate methods exist and have correct signatures
assert hasattr(LangGraphAdapter, "initialize")
assert hasattr(LangGraphAdapter, "run")
assert hasattr(LangGraphAdapter, "handle_message")

# Validate they're async and accept correct types
assert inspect.iscoroutinefunction(adapter.initialize)
assert sig.parameters["context"].annotation == RunnerContext
```

**Success Criteria:**
- 100% interface method coverage
- All type hints validated
- All async/await patterns match

**Status:** Ready to implement

---

### Category 2: Integration Testing (Component Interactions)

**Sub-categories:**

#### 2.1 Operator Behavior
**What:** Operator selects correct runner image based on runnerType

**How:**
```
1. Create AgenticSession CR with runnerType: langgraph
2. Operator watches CR and provisions Job
3. Verify Job image = AMBIENT_LANGGRAPH_RUNNER_IMAGE env var
4. Repeat with runnerType: claude-code
5. Test default (no runnerType specified)
```

**Test Count:** 8 operator tests

#### 2.2 Multi-Repo Support
**What:** LangGraph clones multiple repositories correctly

**How:**
```
1. Create session with REPOS_JSON containing 3+ repos
2. Verify each repo cloned to workspace/<name>/
3. Verify all repos have .git directories
4. Verify git remote URLs configured
```

**Test Count:** 6 multi-repo tests

#### 2.3 WebSocket Streaming
**What:** Real-time progress updates streamed to frontend within SLA

**How:**
```
1. Start LangGraph session
2. Monitor WebSocket messages
3. Verify messages arrive within 2 seconds
4. Verify message types match protocol
5. Test partial message fragmentation for large outputs
```

**Test Count:** 7 WebSocket tests

---

### Category 3: API Validation Testing

**What:** Backend validates runnerType parameter correctly

**Test Examples:**

```python
# Valid values accepted
assert create_session({...,"runnerType": "claude-code"}).status == 201
assert create_session({...,"runnerType": "langgraph"}).status == 201

# Invalid values rejected
assert create_session({...,"runnerType": "spark"}).status == 400
assert "Supported types: claude-code, langgraph" in error_message

# Default behavior
assert create_session({...}).spec.runnerType == "claude-code"  # Default
```

**Test Count:** 12 API validation tests

---

### Category 4: Regression Testing (Claude Code Protection)

**Critical Requirement:** FR-022 states "Claude Code runner implementation MUST remain completely unchanged"

**What:** Every Claude Code workflow must work identically

**Test Scope:**
- Single-repo execution
- Multi-repo execution
- Interactive mode
- Session continuation
- Error handling
- Artifact generation
- Default behavior (no runnerType specified)

**Test Count:** 10+ regression tests

**Key Assertion:**
```
100% of Claude Code workflows must pass with identical output
```

---

### Category 5: Boundary & Negative Tests

**What:** Edge cases and error conditions

**Examples:**

```python
# Boundary conditions
test_extremely_large_prompt()  # >100KB prompt
test_many_repositories()  # 15+ repos
test_very_deep_directory_structure()  # nested paths
test_special_characters_in_paths()  # hyphens, dots, underscores

# Negative scenarios
test_langgraph_api_authentication_failure()  # Invalid API key
test_repository_clone_failure()  # Network error
test_execution_timeout()  # Exceeds timeout period
test_artifact_schema_validation_failure()  # Malformed output
```

**Test Count:** 12 boundary tests

---

### Category 6: End-to-End Testing (Complete Lifecycle)

**What:** Full session from creation through artifact production

**Test Phases:**

| Phase | Input | Output | Validation |
|-------|-------|--------|-----------|
| Ideate | Prompt | rfe.md | Format matches RFE template |
| Specify | rfe.md + prompt | spec.md | References RFE correctly |
| Plan | spec.md + prompt | plan.md | Follows plan template |
| Tasks | plan.md + prompt | tasks.md | Breakdown structure correct |

**Cross-Runner Prevention:**
```
Cannot continue Claude Code session with LangGraph runner
Cannot continue LangGraph session with Claude Code runner
(Test explicitly validates error returned with clear message)
```

**Test Count:** 8+ E2E tests

---

## 3. Testing Pyramid (Test Distribution)

```
                    E2E Tests (8+)
                   /            \
                  /              \
              Integration      (20+ tests)
             /    |    |    \
            /     |    |     \
       Operator  Multi  WebSocket
       (8 tests) Repo   (7 tests)
                (6)

    Unit & Contract Tests (40+ tests)
    ──────────────────────────────────
    - Adapter methods
    - Error handling
    - Message handling
    - API validation

Total Test Count: ~100+ automated tests
```

---

## 4. Critical Risk Mitigation

### Risk: Claude Code Regression
**Impact:** High - Breaks existing users
**Mitigation:** 100% regression test suite + manual smoke test
**Status:** Covered

### Risk: Operator Image Selection Fails
**Impact:** High - Sessions won't start
**Mitigation:** Integration tests on Kind cluster
**Status:** Covered

### Risk: Cross-Runner Contamination
**Impact:** Medium - Data loss potential
**Mitigation:** Strict interface contract, validation tests
**Status:** Covered

### Risk: WebSocket Timeout
**Impact:** Medium - Poor UX
**Mitigation:** SLA testing (2-second latency)
**Status:** Covered

---

## 5. Implementation Roadmap

### Week 1-2: Foundation
- [ ] Setup pytest, conftest.py, fixtures
- [ ] Create Kind cluster for testing
- [ ] Implement contract tests
- [ ] Build mock objects (WebSocket, Anthropic client)

**Deliverable:** Test infrastructure ready

### Week 2-3: Adapter Testing
- [ ] Implement LangGraphAdapter
- [ ] Create unit tests (>85% coverage)
- [ ] Create error handling tests

**Deliverable:** LangGraphAdapter tested and verified

### Week 3-4: Integration Testing
- [ ] Operator image selection tests
- [ ] Multi-repo cloning tests
- [ ] WebSocket streaming tests

**Deliverable:** Full integration with operator validated

### Week 4: API & Regression
- [ ] Backend API validation tests
- [ ] Claude Code regression suite
- [ ] Cross-runner prevention tests

**Deliverable:** Zero regression on Claude Code

### Week 5: E2E & Release
- [ ] End-to-end workflow tests
- [ ] Multi-cluster validation (Standard, FIPS, GPU)
- [ ] Release gate checklist

**Deliverable:** Production-ready with test coverage

---

## 6. CI/CD Integration

### GitHub Actions Pipeline

```yaml
On: push to main or ambient-langgraph-runner branch

1. Unit Tests (5 min)
   ├─ pytest tests/unit/
   ├─ pytest tests/contract/
   └─ Coverage report

2. Integration Tests (15 min) [parallel]
   ├─ Operator selection tests
   ├─ Multi-repo tests
   └─ WebSocket tests

3. API Validation (5 min)
   ├─ RunnerType parameter validation
   ├─ Error message quality
   └─ Default behavior

4. Regression Tests (10 min)
   └─ Claude Code workflows

5. Report Summary (generated)
   └─ All results posted to PR

Total Pipeline Time: ~25 minutes
```

### Pre-Deployment Checklist

```
MUST PASS Before Release:
✓ All unit tests passing (100% criteria)
✓ All contract tests passing (interface validated)
✓ All API validation tests passing
✓ All regression tests passing (Claude Code unchanged)
✓ Integration tests passing on Kind cluster
✓ No code coverage regression (<1% drop)
✓ Documentation complete
✓ Cross-runner prevention validated
```

---

## 7. Test Data & Fixtures

### Provided Fixtures

```python
# Environment
mock_env  # Standard environment variables
runner_context  # RunnerContext for adapter

# Git Repositories
mock_git_repo  # Initialized git repo in temp directory
multi_repo_config  # REPOS_JSON with 3 repos

# API Objects
langgraph_session_request  # CreateAgenticSessionRequest payload
claude_code_session_request  # Claude Code equivalent

# Mock Objects
mock_runner_shell  # Mocked RunnerShell for testing
mock_anthropic_client  # Mocked Anthropic API client
mock_websocket_server  # Mock WebSocket for streaming tests

# Sample Artifacts
sample-rfe.md  # RFE template
sample-spec.md  # Specification template
```

### Test Data Reusability
- 15+ pre-built fixtures minimize test code duplication
- All fixtures parameterized for variant testing
- Clean isolation between tests (tmp_path, monkeypatch)

---

## 8. Success Metrics

### Coverage Targets

| Category | Target | Measurement |
|----------|--------|-------------|
| Code Coverage | >85% | LangGraphAdapter implementation |
| Test Coverage | 100% | All interface methods |
| Regression Coverage | 100% | All Claude Code workflows |
| API Coverage | 100% | All runnerType validation |
| Integration Coverage | >80% | Operator + multi-repo + WebSocket |
| E2E Coverage | All 4 phases | Ideate, Specify, Plan, Tasks |

### Quality Metrics

| Metric | Target | How Measured |
|--------|--------|-------------|
| Test Execution Time | <10 min (unit) | Local pytest run |
| Test Stability | 99% pass rate | Failures due to logic, not flakiness |
| Error Message Quality | 100% | Actionable + contextual |
| Documentation | 100% | Every test scenario documented |

---

## 9. FAQ: Common Questions

### Q: Why write 100+ tests for one feature?
**A:** LangGraph is a critical P1 feature that enables alternative execution framework. Testing must ensure:
- No regression on 100% of existing users (Claude Code)
- Correct operator behavior across all configurations
- WebSocket streaming works reliably
- Error messages are clear and actionable
- Cross-runner safety enforced

### Q: Can we skip regression testing?
**A:** NO. FR-022 explicitly requires "Claude Code runner implementation MUST remain completely unchanged." Regression testing is a hard requirement for production deployment.

### Q: What if a test fails?
**A:** Follow troubleshooting guide in Testing Strategy document. Common issues:
- asyncio timeout → increase timeout, check resources
- WebSocket connection refused → verify mock fixture
- Git command failed → provide credentials
- Image not found → load Docker image into Kind cluster

### Q: How long does the full test suite take?
**A:** ~25 minutes total in CI/CD (parallelized)
- Unit tests: 5 min
- Integration: 15 min (parallel)
- API validation: 5 min
- Regression: 10 min

### Q: Can I run tests locally without Kind cluster?
**A:** Yes. Unit and contract tests run on your machine (~5 min). Integration tests require Kind cluster (~30 min additional setup).

### Q: What if I need to modify an existing test?
**A:** Follow standard procedure:
1. Read current test to understand intent
2. Modify implementation as needed
3. Run: `pytest tests/ -v`
4. Update documentation if behavior changed
5. Submit PR with test changes

---

## 10. Key Decisions & Rationale

### Decision 1: Python pytest over Go testing
**Rationale:** LangGraphAdapter is Python-based, pytest has excellent async support, existing runner-shell already uses pytest patterns.

### Decision 2: 100% regression testing vs sampling
**Rationale:** Claude Code is production-critical (FR-022). Cannot afford to miss regression. Comprehensive suite gives confidence.

### Decision 3: Kind cluster for integration vs mocks
**Rationale:** Mocks are fragile and don't catch real operator issues. Kind cluster is lightweight and catches real integration problems.

### Decision 4: Contract testing before implementation
**Rationale:** Contract tests define what "correct" means before code exists. Prevents interface mismatch later.

### Decision 5: SLA testing for WebSocket (2-second latency)
**Rationale:** Users expect real-time feedback. 2 seconds is acceptable latency threshold; slower is poor UX.

---

## 11. Document Navigation

```
┌─ You are here: TESTING-STRATEGY-SUMMARY.md
│  (Executive overview & quick reference)
│
├─→ LangGraph-Testing-Strategy.md
│   (Complete testing framework - 2000+ lines of details)
│
└─→ LangGraph-Test-Implementation-Guide.md
    (Copy-paste ready code examples & pytest setup)
```

### Where to Start

- **New to this feature?** → Read Section 2 (Six Categories)
- **Need to implement tests?** → Go to Implementation Guide
- **Need details on specific test?** → See Testing Strategy document
- **Setting up CI/CD?** → See Section 6 (CI/CD Integration)
- **Troubleshooting test failure?** → See Section 9 (FAQ)

---

## 12. Next Steps

### For QA Engineers
1. Review this summary
2. Read LangGraph-Testing-Strategy.md (Sections 1-3)
3. Set up local test environment using Implementation Guide
4. Implement contract tests first
5. Wait for LangGraphAdapter implementation
6. Implement unit tests alongside adapter development

### For Developers
1. Read this summary (Sections 1-2)
2. Review LangGraph-Testing-Strategy.md (Sections 1-2 and 4)
3. Implement LangGraphAdapter following contract tests
4. Run unit tests frequently during development
5. Request QA review before merging

### For DevOps
1. Review Section 6 (CI/CD Integration)
2. Set up GitHub Actions workflow
3. Configure Kind cluster for integration tests
4. Set up code coverage reporting (Codecov)
5. Implement pre-deployment checklist validation

### For Leadership
1. Review this summary
2. Focus on Section 8 (Success Metrics)
3. Key commitments:
   - 100% regression on Claude Code (FR-022)
   - <25 min CI/CD pipeline
   - Production-ready with comprehensive testing
   - Clear release gate checklist

---

## 13. Contacts & Support

| Question | Contact | Location |
|----------|---------|----------|
| Testing strategy | Neil (QA Architect) | Testing Strategy doc |
| Implementation details | Neil + Dev team | Implementation Guide |
| CI/CD setup | DevOps + Neil | Section 6 |
| Test failures | See FAQ | Section 9 |

---

## Summary: Why This Strategy Works

✓ **Risk-Based:** Focuses on highest-impact areas first
✓ **Comprehensive:** Covers all 6 testing categories
✓ **Practical:** Copy-paste ready code examples
✓ **Scalable:** Modular test structure enables expansion
✓ **Maintainable:** Clear documentation and fixtures
✓ **Production-Ready:** Pre-deployment checklist ensures quality

---

**Document Version:** 1.0
**Created:** 2025-11-04
**Status:** Final - Ready for Implementation
**Next Review:** Post-Alpha Release Testing

