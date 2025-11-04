# Testing Framework Architecture for LangGraph Runner

**Document Version:** 1.0
**Created:** 2025-11-04
**Purpose:** Visual architecture and framework specifications

---

## 1. Testing Architecture Overview

### 1.1 Complete Testing Stack

```
┌────────────────────────────────────────────────────────────────┐
│                    TEST EXECUTION LAYER                         │
├────────────────────────────────────────────────────────────────┤
│  GitHub Actions  │  Local pytest  │  CI/CD Pipeline            │
│  Workflow        │  Development   │  Integration               │
└────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────┐
│                    TEST ORCHESTRATION                           │
├────────────────────────────────────────────────────────────────┤
│  pytest  │  pytest-asyncio  │  pytest-cov  │  pytest-mock    │
│  asyncio  │  unittest.mock  │  responses   │  faker          │
└────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────┐
│                   TEST CATEGORIES (6 Types)                    │
├────┬────────────┬──────────────┬────────────┬──────────────────┤
│ C1 │     C2     │      C3      │     C4     │    C5      │ C6  │
│    │            │              │            │            │     │
│Con-│Integration │ API Valid.   │ Regression │ Boundary & │ E2E │
│ract│ Testing    │ Testing      │ Testing    │ Negative   │Test │
│    │            │              │            │            │     │
│100%│  80%+      │    100%      │    100%    │    ~20%    │All  │
│    │ coverage   │  coverage    │ coverage   │ scenarios  │  4  │
│    │            │              │            │            │Phases│
└────┴────────────┴──────────────┴────────────┴────────────┴─────┘
                              ↓
┌────────────────────────────────────────────────────────────────┐
│                    COMPONENT UNDER TEST                         │
├────────────────────────────────────────────────────────────────┤
│  LangGraphAdapter  │  Operator  │  Backend API  │  Frontend UI │
└────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────┐
│                  EXTERNAL DEPENDENCIES                          │
├────────────────────────────────────────────────────────────────┤
│  Kubernetes  │  Git  │  Anthropic API  │  GitHub  │  WebSocket │
│  (Kind/K8s)  │(Local)│   (Mocked)      │ (Mocked) │  (Mocked)  │
└────────────────────────────────────────────────────────────────┘
```

### 1.2 Testing Workflow Diagram

```
Developer Creates PR
    ↓
┌─────────────────────────────────────┐
│  GitHub Actions Trigger             │
│  - Run on push to branch            │
│  - Run on PR creation               │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│  Lint & Type Checking               │
│  - black formatting                 │
│  - mypy type hints                  │
│  - pylint warnings                  │
│  (1 min)                            │
└─────────────────────────────────────┘
    ↓ (fail) → Block PR merge
    ↓ (pass)
┌─────────────────────────────────────┐
│  Unit & Contract Tests              │
│  - pytest tests/unit/               │
│  - pytest tests/contract/           │
│  - Coverage report                  │
│  (5 min)                            │
└─────────────────────────────────────┘
    ↓ (fail) → Block PR merge
    ↓ (pass)
┌─────────────────────────────────────┐
│  Integration Tests                  │
│  - Setup Kind cluster               │
│  - Deploy components                │
│  - Operator selection tests         │
│  - WebSocket streaming tests        │
│  (15 min)                           │
└─────────────────────────────────────┘
    ↓ (fail) → Block PR merge
    ↓ (pass)
┌─────────────────────────────────────┐
│  API Validation Tests               │
│  - RunnerType validation            │
│  - Error message quality            │
│  (5 min)                            │
└─────────────────────────────────────┘
    ↓ (fail) → Block PR merge
    ↓ (pass)
┌─────────────────────────────────────┐
│  Regression Tests                   │
│  - Claude Code workflows            │
│  - Backward compatibility           │
│  (10 min)                           │
└─────────────────────────────────────┘
    ↓ (fail) → CRITICAL ALERT - investigate
    ↓ (pass)
┌─────────────────────────────────────┐
│  Test Report Generated              │
│  - All test results                 │
│  - Coverage metrics                 │
│  - Performance benchmarks           │
│  - Recommendation (merge/retry)     │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│  PR Comment Posted                  │
│  - Summary of all tests             │
│  - ✓ All tests passing             │
│  - Coverage: 87% (+2%)             │
│  - Ready to merge                  │
└─────────────────────────────────────┘
    ↓
Approve & Merge to main
    ↓
Optional E2E Tests (on-demand)
├─ Multi-cluster testing
├─ GPU cluster testing
├─ FIPS cluster testing
└─ Load testing

    ↓
Release Candidate Deployment
```

---

## 2. Framework Components & Dependencies

### 2.1 Testing Framework Stack

```
Layer                 Tools/Libraries          Version    Purpose
─────────────────────────────────────────────────────────────────
Test Runner          pytest                   7.4.0      Main framework
                     pytest-asyncio           0.21.0     Async support

Coverage             pytest-cov               4.1.0      Code coverage
                     coverage                 7.3.0      Coverage analysis

Mocking              pytest-mock              3.11.1     Mock fixtures
                     responses                0.23.1     HTTP mocking
                     unittest.mock            stdlib     Python mocking

Async Testing        asyncio                  stdlib     Async execution
                     aiofiles                 23.1.0     Async file I/O

Test Data            faker                    19.0.0     Fake data generation
                     factory-boy              3.3.0      Test object factories

Code Quality         black                    23.9.1     Code formatting
                     mypy                     1.5.0      Type checking
                     pylint                   2.17.5     Code linting

K8s Testing          kubernetes               28.0.0     K8s client API
                     kind                     v0.20      Local K8s clusters

Version Control      GitPython                3.1.37     Git operations

WebSocket            websocket-client         1.6.1      WebSocket testing
```

### 2.2 Test Framework Architecture

```
┌────────────────────────────────────────────────────────────┐
│                    pytest Entry Point                       │
│                  (pytest.ini configuration)                 │
├────────────────────────────────────────────────────────────┤
│  Command: pytest tests/ -v                                  │
│  - Discovers test_*.py files                               │
│  - Loads conftest.py (global fixtures)                     │
│  - Registers markers                                        │
│  - Configures asyncio                                       │
└────────────────────────────────────────────────────────────┘
    ↓
┌────────────────────────────────────────────────────────────┐
│              conftest.py (Fixture Definitions)              │
├────────────────────────────────────────────────────────────┤
│  Session-Scoped:                                            │
│  ├─ event_loop                                             │
│  └─ Kubernetes cluster setup (if applicable)               │
│                                                            │
│  Module-Scoped:                                             │
│  ├─ mock_anthropic_client                                  │
│  ├─ mock_git_repo                                          │
│  └─ multi_repo_config                                      │
│                                                            │
│  Function-Scoped:                                           │
│  ├─ mock_env                                               │
│  ├─ runner_context                                         │
│  ├─ clean_workspace                                        │
│  ├─ langgraph_session_request                              │
│  └─ mock_runner_shell                                      │
└────────────────────────────────────────────────────────────┘
    ↓
┌────────────────────────────────────────────────────────────┐
│        Test Collection & Parameter Generation               │
├────────────────────────────────────────────────────────────┤
│  For each test_*.py file:                                   │
│  1. Parse test classes and methods                         │
│  2. Apply markers (@pytest.mark.asyncio)                   │
│  3. Resolve fixture dependencies                            │
│  4. Generate parameterized test variants                    │
│  5. Build execution graph                                   │
└────────────────────────────────────────────────────────────┘
    ↓
┌────────────────────────────────────────────────────────────┐
│            Test Execution & Isolation                       │
├────────────────────────────────────────────────────────────┤
│  For each test:                                             │
│  1. Setup fixtures (create temp dirs, mocks, etc)          │
│  2. Run test code (function or async function)             │
│  3. Capture output, errors, timing                         │
│  4. Teardown fixtures (cleanup)                            │
│  5. Record result (pass/fail/skip)                         │
└────────────────────────────────────────────────────────────┘
    ↓
┌────────────────────────────────────────────────────────────┐
│           Result Collection & Reporting                     │
├────────────────────────────────────────────────────────────┤
│  - Test count (passed/failed/skipped)                      │
│  - Code coverage report                                     │
│  - Performance metrics (test duration)                      │
│  - HTML report generation                                   │
│  - Artifact upload (for CI/CD)                             │
└────────────────────────────────────────────────────────────┘
```

---

## 3. Test Category Interconnections

### 3.1 Testing Dependency Graph

```
Contract Tests (C1)
├─ Define what "correct" means
├─ Input to: Unit tests, Integration tests
└─ GATES: Unit tests can't pass without contract compliance

    ↓ (feeds into)

Unit Tests (C2)
├─ Test individual adapter methods
├─ Must pass contract tests first
├─ Input to: Integration tests, E2E tests
└─ GATES: Integration tests require unit test pass

    ↓ (feeds into)

API Validation Tests (C3)
├─ Test backend request handling
├─ Independent of adapter implementation
├─ Input to: E2E tests
└─ GATES: E2E tests require API validation pass

Integration Tests (C4)
├─ Test operator + adapter interaction
├─ Requires both contract + unit tests pass
├─ Input to: E2E tests
└─ GATES: E2E tests require integration pass

Regression Tests (C5)
├─ Ensure Claude Code unchanged
├─ Can run in parallel with other tests
├─ Input to: Release gate checklist
└─ GATES: Must pass to deploy to production

    ↓ (all feed into)

E2E Tests (C6)
├─ Test complete session lifecycle
├─ Requires: contract, unit, integration, API tests pass
├─ Optional: regression tests (but recommended)
└─ GATES: Release gate - must pass for production

    ↓ (all feed into)

Production Release Gate
├─ All 6 categories must pass
├─ Coverage metrics validated
├─ No regressions detected
├─ Documentation complete
└─ Ready for production deployment
```

---

## 4. Test Execution Paths

### 4.1 Local Development Workflow

```
Developer starts work:
┌─────────────────────────────────┐
│ pytest tests/contract/ -v       │  Run contract tests first
│ (verify interface is correct)   │  (~2 min)
└─────────────────────────────────┘
           ↓ (pass)
┌─────────────────────────────────┐
│ pytest tests/unit/ -v           │  Develop and test adapter
│ --cov=langgraph_runner          │  simultaneously
│ (repeat until >85% coverage)    │  (~5 min each iteration)
└─────────────────────────────────┘
           ↓ (pass)
┌─────────────────────────────────┐
│ pytest tests/ -v                │  Full test suite
│ (all tests including E2E)       │  (~25 min, can be skipped
│                                 │   if not ready for E2E)
└─────────────────────────────────┘
           ↓ (pass)
┌─────────────────────────────────┐
│ git push to feature branch      │  Submit PR
│ (GitHub Actions runs            │
│  full pipeline)                 │
└─────────────────────────────────┘
```

### 4.2 CI/CD Pipeline Paths

```
On Pull Request:
┌───────────────────────────────────────────────────┐
│ 1. Lint & Type Check (1 min) → Fail? Block merge │
└───────────────────────────────────────────────────┘
    ↓ (pass)
┌───────────────────────────────────────────────────┐
│ 2. Unit + Contract Tests (5 min) → Fail? Block  │
└───────────────────────────────────────────────────┘
    ↓ (pass)
┌───────────────────────────────────────────────────┐
│ 3. Integration Tests (15 min, PARALLEL)          │
│   ├─ Operator tests                              │
│   ├─ Multi-repo tests                            │
│   ├─ WebSocket tests                             │
│   └─ Fail any? Block merge                       │
└───────────────────────────────────────────────────┘
    ↓ (pass)
┌───────────────────────────────────────────────────┐
│ 4. API Validation (5 min) → Fail? Block merge    │
└───────────────────────────────────────────────────┘
    ↓ (pass)
┌───────────────────────────────────────────────────┐
│ 5. Regression Tests (10 min) → Fail? ALERT      │
│                       (but doesn't block merge)   │
└───────────────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────────────┐
│ 6. Report Posted to PR                           │
│    ✓ All tests: PASS / FAIL / SKIPPED            │
│    ✓ Coverage: X% (trend)                        │
│    ✓ Status: Ready to Merge / Needs Fix          │
└───────────────────────────────────────────────────┘
    ↓
Approve & Merge to main
    ↓
Optional: On-Demand E2E Tests
├─ Trigger manually if needed
├─ Run on release candidate branch
└─ Full cluster validation
```

---

## 5. Mock Architecture

### 5.1 Mock Object Hierarchy

```
┌─────────────────────────────────────────────────────────┐
│           Real External Dependencies                    │
├─────────────────────────────────────────────────────────┤
│  • Kubernetes Cluster (kind://localhost:6443)           │
│  • Anthropic API (https://api.anthropic.com)            │
│  • GitHub API (https://api.github.com)                  │
│  • WebSocket Server (wss://backend:8080)                │
│  • Git Repositories (https://github.com/...)            │
└─────────────────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────────────────┐
│           Test Mocking Layer                            │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Unit Tests (mocks everything):                         │
│  ├─ mock_anthropic_client (responses library)           │
│  ├─ mock_websocket_server (unittest.mock)               │
│  ├─ mock_git_repo (tmp_path fixture)                    │
│  └─ mock_runner_shell (AsyncMock)                       │
│                                                         │
│  Integration Tests (partial mocking):                   │
│  ├─ Real Kind cluster (local K8s)                       │
│  ├─ Real git repos (tmp_path)                           │
│  ├─ mock_anthropic_client (responses)                   │
│  └─ real operator (deployed to cluster)                 │
│                                                         │
│  E2E Tests (minimal mocking):                           │
│  ├─ Real K8s cluster                                    │
│  ├─ Real git repos                                      │
│  ├─ mock_anthropic_client (controlled responses)        │
│  └─ Real WebSocket streaming                           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 5.2 Mock Response Patterns

```
Mock Anthropic Client:
├─ Successful response
│  └─ {"content": [{"text": "Sample spec"}]}
├─ API error response
│  └─ {"error": {"message": "invalid_api_key"}}
├─ Rate limit response
│  └─ HTTP 429 with retry-after header
└─ Timeout response
   └─ Connection timeout after N seconds

Mock WebSocket:
├─ Connect message
│  └─ {"type": "connected"}
├─ Progress message
│  └─ {"type": "message.partial", "id": "123", "data": "..."}
├─ Error message
│  └─ {"type": "error", "message": "..."}
└─ Close message
   └─ {"type": "closed", "code": 1000}

Mock Git Repository:
├─ Clone success
│  └─ Repository cloned to /tmp/workspace/repo
├─ Clone failure (non-existent)
│  └─ RuntimeError: Repository not found
├─ Authentication failure
│  └─ RuntimeError: Authentication failed
└─ Network timeout
   └─ RuntimeError: Connection timeout
```

---

## 6. Test Environment Configuration

### 6.1 pytest.ini Configuration

```ini
[pytest]
# Async support
asyncio_mode = auto

# Test discovery
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*

# Coverage
addopts =
    --cov=langgraph_runner
    --cov-report=html
    --cov-report=term-missing:skip-covered
    --cov-branch
    --cov-fail-under=85
    --timeout=30
    -v
    -ra

# Markers
markers =
    unit: Unit tests
    contract: Contract tests
    integration: Integration tests
    e2e: End-to-end tests
    asyncio: Async functions
    slow: Slow tests (>5s)
    websocket: WebSocket tests
    k8s: Kubernetes tests

# Console output
console_output_style = progress
```

### 6.2 Environment Variables for Testing

```bash
# pytest environment
export PYTHONPATH=${PYTHONPATH}:$(pwd)
export PYTEST_TIMEOUT=30

# Kind cluster
export KUBECONFIG=~/.kube/config
export KIND_CLUSTER_NAME=vteam-test

# Mocking
export MOCK_API_RESPONSES=true
export SKIP_REAL_API_CALLS=true

# Logging
export PYTEST_LOG_LEVEL=INFO
export DEBUG=false
```

---

## 7. Coverage Analysis Strategy

### 7.1 Coverage Metrics

```
Code Coverage Target: >85%
├─ Branch Coverage: >80%
│  ├─ Happy path: 100%
│  ├─ Error paths: 90%+
│  └─ Edge cases: 75%+
│
├─ Line Coverage: >90%
│  ├─ Core methods: 100%
│  ├─ Helper methods: 85%+
│  └─ Dead code: 0%
│
└─ Function Coverage: 100%
   ├─ All public methods: tested
   ├─ All private methods: tested indirectly
   └─ Exception handling: tested

Coverage Report:
├─ HTML report (coverage.html)
├─ Terminal output (term-missing)
├─ XML report (for CI/CD)
└─ Delta report (PR vs main)
```

### 7.2 Coverage Exclusions

```python
# Exclude from coverage analysis
# pragma: no cover

# Lines that should NOT be excluded:
├─ Exception handlers (must test error paths)
├─ Conditional branches (must test both paths)
├─ Async/await code (all must be tested)
└─ Critical paths (100% coverage required)

# Lines that CAN be excluded:
├─ Debug logging (if-__name__ == '__main__':)
├─ Version strings (automatically generated)
├─ Type stubs (type: ignore comments)
└─ Placeholder implementations (before final code)
```

---

## 8. Performance Baseline Tests

### 8.1 Performance Metrics Captured

```
For each test run:
├─ Execution time per test (in seconds)
├─ Memory usage peak (in MB)
├─ CPU utilization (percentage)
├─ I/O operations count
├─ Network calls (mocked count)
└─ Database queries (if applicable)

Baseline Performance Targets:
├─ Contract tests: <1 second each
├─ Unit tests: <2 seconds each
├─ Integration tests: <10 seconds each
├─ E2E tests: <60 seconds each
├─ Full suite: <25 minutes (CI/CD)
└─ Full suite: <10 minutes (local, parallel)
```

### 8.2 Performance Regression Detection

```
If test execution time increases by >20%:
├─ Investigate root cause
├─ Check for new test data or fixtures
├─ Review algorithm changes in adapter
├─ Check for resource contention
└─ Profile with pytest-benchmark if needed

Alert Thresholds:
├─ Unit tests >3s: Review
├─ Integration tests >15s: Review
├─ Full suite >30 min: Investigate
└─ Any test >100s: Critical alert
```

---

## 9. Troubleshooting Framework

### 9.1 Test Failure Resolution Flowchart

```
Test Fails
    ↓
Is it a timeout?
├─ YES → Increase pytest timeout
│       └─ Check for slow operations
│
└─ NO
    ↓
Is it an import error?
├─ YES → Verify Python path
│       └─ Check dependencies installed
│
└─ NO
    ↓
Is it an async error?
├─ YES → Verify pytest-asyncio installed
│       └─ Check asyncio_mode = auto in pytest.ini
│
└─ NO
    ↓
Is it a mock not working?
├─ YES → Verify mock is set up correctly
│       └─ Check mock is applied to right object
│
└─ NO
    ↓
Is it an environment variable missing?
├─ YES → Set in conftest.py mock_env
│       └─ Add to CI/CD environment
│
└─ NO
    ↓
Debug with: pytest -vv --tb=long --capture=no
```

---

## 10. Test Framework Quick Reference

### 10.1 Common Commands

```bash
# Discovery & Inspection
pytest tests/ --collect-only          # Show all tests that would run
pytest tests/ --markers               # Show all marker definitions
pytest tests/ -m "contract"           # Show contract tests

# Execution
pytest tests/ -v                      # Verbose output
pytest tests/ -vv                     # More verbose (full tracebacks)
pytest tests/unit/ -v                 # Specific directory
pytest tests/unit/test_adapter.py -v  # Specific file
pytest tests/ -k "test_run"           # Match test name pattern
pytest tests/ -m "not slow"           # Exclude slow tests
pytest tests/ --timeout=60            # Set timeout per test

# Coverage
pytest tests/ --cov=langgraph_runner
pytest tests/ --cov-report=html       # Generate HTML report
pytest tests/ --cov-report=term-missing

# Debugging
pytest tests/ -x                      # Stop on first failure
pytest tests/ --lf                    # Run last failed
pytest tests/ -vv --tb=long           # Full tracebacks
pytest tests/ -vv --capture=no        # Show print output
pytest tests/ --pdb                   # Enter debugger on failure

# Performance
pytest tests/ --durations=10          # Show 10 slowest tests
pytest tests/ --benchmark-only        # Run benchmarks only
```

### 10.2 Fixture Reference

```python
# In any test file, use these fixtures:

async def test_something(
    runner_context,                    # RunnerContext object
    mock_env,                          # Environment variables dict
    clean_workspace,                   # Temporary workspace path
    mock_runner_shell,                 # Mocked RunnerShell
    mock_anthropic_client,             # Mocked Anthropic API
    mock_git_repo,                     # Initialized git repo
    multi_repo_config,                 # REPOS_JSON string
    langgraph_session_request,         # Session request dict
    monkeypatch,                       # pytest monkeypatch
    tmp_path,                          # pytest temp directory
):
    """All fixtures available to tests."""
    pass
```

---

**Testing Framework Architecture Complete**

This document provides the technical foundation for implementing the LangGraph runner testing strategy. Combined with the Testing Strategy and Implementation Guide documents, it forms a complete, production-ready testing framework.

