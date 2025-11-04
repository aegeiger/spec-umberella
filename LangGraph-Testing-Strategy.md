# Testing Strategy: LangGraph Runner Support for vTeam

**Document Version:** 1.0
**Created:** 2025-11-04
**Status:** Final
**Audience:** QA Engineers, Development Team, DevOps

---

## Executive Summary

This document defines a comprehensive, risk-based testing strategy for implementing LangGraph runner support in the vTeam Ambient Agentic Runner platform. The strategy covers contract testing, integration testing, regression validation, API validation, and end-to-end testing across multiple cluster configurations.

The testing approach balances comprehensive coverage with practical delivery timelines by:
- Leveraging existing runner-shell framework abstractions for consistency
- Using the ClaudeCodeAdapter as a reference contract for LangGraphAdapter
- Integrating testing seamlessly with existing CI/CD pipelines
- Implementing fixture-based pytest patterns for efficient test data management
- Validating operator behavior through Kubernetes object inspection
- Ensuring zero regression on existing Claude Code runner functionality

---

## 1. Testing Scope & Categories

### 1.1 Scope Definition

| Dimension | Scope | Owner | Priority |
|-----------|-------|-------|----------|
| **Components** | LangGraphAdapter, Operator (runnerType routing), Backend API validation, Frontend UI | QA + Dev | P0 |
| **Interfaces** | Runner adapter contract, WebSocket protocol, Kubernetes CR spec | QA | P0 |
| **Platforms** | Standard K8s cluster, FIPS-enabled cluster, Disconnected environment, GPU clusters | QA + DevOps | P1 |
| **Workflows** | All RFE phases (Ideate, Specify, Plan, Tasks) + interactive mode | QA + Product | P1 |
| **Integration Points** | Multi-repo git operations, Anthropic API calls, K8s Job creation, WebSocket streaming | QA + Dev | P1 |
| **Regression** | Claude Code runner 100% compatibility, backward compatibility for existing sessions | QA | P0 |

### 1.2 Testing Categories & Coverage

#### Category 1: Contract Testing (Interface Compliance)

**Objective:** Ensure LangGraphAdapter implements identical interface as ClaudeCodeAdapter
**Coverage Target:** 100% interface parity
**Risk Mitigated:** Interface mismatch, broken polymorphism in runner selection

```
LangGraphAdapter Requirements:
├── initialize(context: RunnerContext) → async
├── run() → async → dict with success/result/error
├── handle_message(message: dict) → async
├── WebSocket message streaming compliance
├── RFE artifact schema matching
└── Environment variable interpretation

Testing Approach:
├── Interface signature validation (method names, parameter types, return types)
├── Async/await execution pattern consistency
├── Exception handling equivalence
├── Message protocol compliance
└── Artifact format schema validation
```

**Critical Test Scenarios from Acceptance Criteria:**

| Scenario | Validation | Expected Result |
|----------|-----------|-----------------|
| AC-1: Interface methods exist | Reflection test on LangGraphAdapter class | All 3 methods present with correct signatures |
| AC-2: Initialize accepts RunnerContext | Type and parameter validation | initialize() accepts RunnerContext, saves to self.context |
| AC-3: Run returns dict result | Return type inspection | Dict with success/result/error keys |
| AC-4: WebSocket message format | Protocol message validation | Messages match MessageType enum + payload schema |
| AC-5: Artifact schema match | Artifact content validation | rfe.md/spec.md/plan.md/tasks.md format identical to Claude Code output |

---

#### Category 2: Integration Testing (Component Interactions)

**Objective:** Validate LangGraph runner operates correctly within vTeam infrastructure
**Coverage Target:** All major workflows + error paths
**Risk Mitigated:** Infrastructure integration failures, multi-repo issues, WebSocket timeouts

**Testing Framework & Tools:**
- **Python Testing:** pytest + pytest-asyncio (for async adapter testing)
- **Kubernetes Testing:** operator-sdk test utilities + kubebuilder enhancements
- **Integration Tests:** Docker containers + Kind cluster for local testing
- **WebSocket Testing:** websocket-client library + mock WebSocket server

#### 2.1 Operator Behavior Tests

**Scope:** Verify operator correctly provisions jobs based on runnerType field

```python
# Test Structure: tests/integration/test_operator_runner_selection.py

@pytest.mark.asyncio
class TestOperatorRunnerSelection:
    """Test operator selects correct runner image based on runnerType."""

    async def test_operator_creates_job_with_langgraph_runner_image():
        """Verify operator uses AMBIENT_LANGGRAPH_RUNNER_IMAGE when runnerType=langgraph"""
        # Create AgenticSession CR with runnerType: langgraph
        # Watch for Job creation
        # Assert job.spec.template.spec.containers[0].image matches AMBIENT_LANGGRAPH_RUNNER_IMAGE
        # Assert environment variables include RUNNER_TYPE=langgraph
        pass

    async def test_operator_creates_job_with_claude_code_runner_image():
        """Verify operator uses AMBIENT_CLAUDE_RUNNER_IMAGE when runnerType=claude-code"""
        # Create AgenticSession CR with runnerType: claude-code
        # Verify image selection matches Claude Code default behavior
        pass

    async def test_operator_defaults_to_claude_code_image():
        """Verify operator defaults to Claude Code image when runnerType not specified"""
        # Create AgenticSession CR without runnerType field
        # Assert default image used
        pass

    async def test_operator_propagates_environment_variables():
        """Verify all session environment variables propagated to Job"""
        # Create session with custom environment variables
        # Verify all env vars present in created Job
        pass
```

**Test Data & Fixtures:**
```python
# tests/integration/fixtures/agentic_session_fixtures.py

@pytest.fixture
def langgraph_session_cr():
    """Valid AgenticSession CR for LangGraph runner."""
    return {
        "apiVersion": "ambient-code.io/v1",
        "kind": "AgenticSession",
        "metadata": {
            "name": "test-langgraph-session",
            "namespace": "default",
        },
        "spec": {
            "prompt": "Create a simple feature specification",
            "runnerType": "langgraph",  # Key field
            "repos": [{
                "input": {
                    "url": "https://github.com/test/repo",
                    "branch": "main"
                }
            }],
            "timeout": 300,
            "llmSettings": {
                "model": "claude-3-5-sonnet-20241022",
                "temperature": 0.7
            }
        }
    }

@pytest.fixture
def claude_code_session_cr():
    """Valid AgenticSession CR for Claude Code runner."""
    return {
        # Same as langgraph_session_cr but with runnerType: claude-code
        # Used for regression testing
    }
```

#### 2.2 Multi-Repo Support Tests

**Scope:** Validate multi-repo cloning and accessibility

```python
# tests/integration/test_langgraph_multi_repo.py

@pytest.mark.asyncio
class TestMultiRepoSupport:
    """Test multi-repository cloning and workspace organization."""

    async def test_clone_multiple_repositories():
        """Verify all repos in REPOS_JSON cloned correctly."""
        repos_json = json.dumps([
            {"name": "repo1", "input": {"url": "...", "branch": "main"}},
            {"name": "repo2", "input": {"url": "...", "branch": "develop"}},
            {"name": "repo3", "input": {"url": "...", "branch": "main"}},
        ])

        context = RunnerContext(
            session_id="multi-repo-test",
            workspace_path="/tmp/workspace",
            environment={"REPOS_JSON": repos_json}
        )

        adapter = LangGraphAdapter()
        await adapter.initialize(context)

        # Verify directory structure
        assert os.path.exists("/tmp/workspace/repo1")
        assert os.path.exists("/tmp/workspace/repo2")
        assert os.path.exists("/tmp/workspace/repo3")
        assert os.path.exists("/tmp/workspace/repo1/.git")
        assert os.path.exists("/tmp/workspace/repo2/.git")
        assert os.path.exists("/tmp/workspace/repo3/.git")

    async def test_langgraph_reads_files_across_repos():
        """Verify LangGraph workflow can access files from different repos."""
        # Create multi-repo setup with different files in each repo
        # Run LangGraph workflow that reads from both repos
        # Verify successful file access and content retrieval
        pass

    async def test_git_operations_multi_repo():
        """Verify git add/commit/push works for each repo independently."""
        # Make changes in repo1 and repo2
        # Run _push_results_if_any()
        # Verify separate commits for each repo
        pass
```

#### 2.3 WebSocket Streaming Tests

**Scope:** Validate real-time progress streaming to frontend

```python
# tests/integration/test_langgraph_websocket.py

@pytest.mark.asyncio
class TestWebSocketStreaming:
    """Test WebSocket message streaming from LangGraph runner."""

    async def test_websocket_messages_streamed_real_time():
        """Verify progress updates stream within 2-second SLA."""
        # Setup mock WebSocket server
        # Create LangGraph session
        # Record message timestamps
        # Assert each message received within 2 seconds of generation
        # Verify message types match MessageType enum
        pass

    async def test_agent_state_transitions_visible():
        """Verify graph node execution visible via WebSocket."""
        # Monitor WebSocket for state transition messages
        # Expected sequence: AGENT_RUNNING → state updates → completion
        # Assert at least 3 distinct state messages received
        pass

    async def test_error_details_streamed_on_failure():
        """Verify error details include LangGraph context."""
        # Force LangGraph error (e.g., API failure)
        # Verify WebSocket message includes:
        #   - Error type (LangGraphError, APIError, etc)
        #   - Failing node name
        #   - Error message
        #   - Stack trace
        pass

    async def test_partial_messages_for_large_outputs():
        """Verify large outputs fragmented and reassembled correctly."""
        # Force LangGraph to produce >10KB output
        # Verify MESSAGE_PARTIAL protocol used
        # Verify correct fragment IDs and sequence numbers
        # Verify client can reassemble complete message
        pass
```

---

#### Category 3: API Validation Testing

**Objective:** Validate backend API request handling and validation
**Coverage Target:** All runnerType permutations + error cases
**Risk Mitigated:** Invalid input acceptance, missing validation, unclear error messages

#### 3.1 RunnerType Parameter Validation

```python
# tests/unit/test_api_runner_type_validation.py

import pytest
from backend.handlers.sessions import CreateAgenticSessionHandler

class TestRunnerTypeValidation:
    """Test API runnerType parameter validation."""

    def test_valid_claude_code_runner_type():
        """POST /sessions with runnerType=claude-code accepted."""
        request = CreateAgenticSessionRequest(
            prompt="Test prompt",
            runner_type="claude-code"
        )
        handler = CreateAgenticSessionHandler()
        response = handler.create_session(request)
        assert response.status_code == 201
        assert response.body.spec.runnerType == "claude-code"

    def test_valid_langgraph_runner_type():
        """POST /sessions with runnerType=langgraph accepted."""
        request = CreateAgenticSessionRequest(
            prompt="Test prompt",
            runner_type="langgraph"
        )
        handler = CreateAgenticSessionHandler()
        response = handler.create_session(request)
        assert response.status_code == 201
        assert response.body.spec.runnerType == "langgraph"

    def test_default_claude_code_when_not_specified():
        """POST /sessions without runnerType defaults to claude-code."""
        request = CreateAgenticSessionRequest(
            prompt="Test prompt"
            # runnerType not specified
        )
        handler = CreateAgenticSessionHandler()
        response = handler.create_session(request)
        assert response.status_code == 201
        assert response.body.spec.runnerType == "claude-code"

    def test_invalid_runner_type_rejected():
        """POST /sessions with invalid runnerType returns 400."""
        request_data = {
            "prompt": "Test prompt",
            "runnerType": "invalid-runner-type"
        }
        handler = CreateAgenticSessionHandler()
        with pytest.raises(ValidationError) as exc_info:
            handler.create_session(request_data)

        error_response = exc_info.value.response
        assert error_response.status_code == 400
        assert "invalid-runner-type" in error_response.body
        assert "Supported types: claude-code, langgraph" in error_response.body

    def test_null_runner_type_defaults():
        """POST /sessions with runnerType=null defaults to claude-code."""
        request_data = {
            "prompt": "Test prompt",
            "runnerType": None
        }
        handler = CreateAgenticSessionHandler()
        response = handler.create_session(request_data)
        assert response.body.spec.runnerType == "claude-code"

    def test_empty_string_runner_type_defaults():
        """POST /sessions with runnerType='' defaults to claude-code."""
        request_data = {
            "prompt": "Test prompt",
            "runnerType": ""
        }
        handler = CreateAgenticSessionHandler()
        response = handler.create_session(request_data)
        assert response.body.spec.runnerType == "claude-code"

    def test_case_sensitivity():
        """Test runnerType case sensitivity (should be case-insensitive)."""
        for variant in ["LANGGRAPH", "LangGraph", "langGraph"]:
            request = {"prompt": "Test", "runnerType": variant}
            handler = CreateAgenticSessionHandler()
            with pytest.raises(ValidationError):
                # Expected: rejected due to case sensitivity
                handler.create_session(request)

        # Only exact match accepted
        request = {"prompt": "Test", "runnerType": "langgraph"}
        handler = CreateAgenticSessionHandler()
        response = handler.create_session(request)
        assert response.status_code == 201
```

#### 3.2 Error Message Quality Tests

```python
# tests/unit/test_api_error_messages.py

class TestErrorMessageQuality:
    """Test error responses include actionable information."""

    def test_invalid_runner_type_error_message():
        """Error message lists valid options."""
        handler = CreateAgenticSessionHandler()
        with pytest.raises(ValidationError) as exc:
            handler.create_session({
                "prompt": "Test",
                "runnerType": "spark-runner"  # Invalid
            })

        error_msg = str(exc.value)
        assert "Unsupported runner type 'spark-runner'" in error_msg
        assert "Supported types: claude-code, langgraph" in error_msg

    def test_missing_required_field_error():
        """Error message indicates required fields."""
        handler = CreateAgenticSessionHandler()
        with pytest.raises(ValidationError) as exc:
            handler.create_session({
                # Missing 'prompt'
                "runnerType": "langgraph"
            })

        error_msg = str(exc.value)
        assert "prompt" in error_msg.lower()
        assert "required" in error_msg.lower()
```

---

#### Category 4: Regression Testing (Claude Code Protection)

**Objective:** Ensure zero impact on existing Claude Code runner functionality
**Coverage Target:** 100% - All existing Claude Code workflows must work identically
**Risk Mitigated:** Unintended Claude Code modifications, backward compatibility breakage

**Critical Requirement (FR-022):** Claude Code runner implementation MUST remain completely unchanged with zero code modifications.

```python
# tests/regression/test_claude_code_compatibility.py

@pytest.mark.asyncio
class TestClaudeCodeRegression:
    """Comprehensive regression tests for Claude Code runner."""

    async def test_claude_code_single_repo_execution():
        """Verify Claude Code executes single-repo sessions identically."""
        # Create session with runnerType: claude-code
        # Execute and capture output
        # Verify behavior unchanged from pre-LangGraph implementation
        pass

    async def test_claude_code_multi_repo_execution():
        """Verify Claude Code handles multi-repo configuration."""
        # Create session with REPOS_JSON
        # Execute with Claude Code runner
        # Verify all repos cloned and accessible
        pass

    async def test_claude_code_default_when_runner_type_omitted():
        """Verify sessions without runnerType use Claude Code."""
        # Create session WITHOUT specifying runnerType
        # Verify Claude Code image selected by operator
        # Verify execution behavior identical to explicit claude-code
        pass

    async def test_claude_code_artifact_generation():
        """Verify Claude Code produces RFE artifacts in standard format."""
        # Execute Ideate phase with Claude Code
        # Verify rfe.md created in specs/{branchName}/
        # Verify markdown format identical to baseline
        pass

    async def test_claude_code_interactive_mode():
        """Verify interactive mode works unchanged."""
        # Create session with interactive=true
        # Send user messages via WebSocket
        # Verify responses streamed correctly
        pass

    async def test_claude_code_error_handling():
        """Verify error handling behavior unchanged."""
        # Force error condition (invalid API key, network timeout, etc)
        # Verify error reported via WebSocket and CR status
        # Verify error message format unchanged
        pass

    async def test_claude_code_session_continuation():
        """Verify session resumption works unchanged."""
        # Create initial session, capture session ID
        # Create continuation session with PARENT_SESSION_ID
        # Verify full context preserved
        # Verify artifacts updated correctly
        pass
```

**Pre-Deployment Validation:**
```bash
# Run full Claude Code regression suite before deploying LangGraph
pytest tests/regression/test_claude_code_compatibility.py -v --tb=short

# Smoke test: Create existing Claude Code session, verify it executes identically
# to previous version (can use recorded baseline)
```

---

#### Category 5: Boundary & Negative Tests

**Objective:** Test edge cases, limits, and error conditions
**Coverage Target:** All edge cases from spec + discovered scenarios
**Risk Mitigated:** Unhandled edge cases, resource exhaustion, cascading failures

```python
# tests/unit/test_boundary_cases.py

class TestBoundaryConditions:
    """Test system behavior at boundaries and limits."""

    def test_extremely_large_prompt():
        """Test LangGraph handles very large prompt (>100KB)."""
        large_prompt = "x" * 200000
        request = CreateAgenticSessionRequest(
            prompt=large_prompt,
            runner_type="langgraph"
        )
        # Should either process or reject with clear message
        # Should NOT crash or hang
        pass

    def test_many_repositories():
        """Test multi-repo with many repos (10+)."""
        repos = [
            {"name": f"repo{i}", "input": {"url": f"https://example.com/repo{i}"}}
            for i in range(15)
        ]
        # Verify cloning handles 15 repos without timeout
        # Verify memory usage reasonable
        pass

    def test_very_deep_directory_structure():
        """Test handling of deeply nested file paths."""
        # Create file at path: a/b/c/d/e/f/g/h/i/j/k/file.md
        # Verify LangGraph can access and modify file
        pass

    def test_special_characters_in_paths():
        """Test handling of special characters in repo paths."""
        special_repos = [
            {"name": "repo-with-dash", ...},
            {"name": "repo_with_underscore", ...},
            {"name": "repo.with.dots", ...},
        ]
        # Verify all cloned and accessible
        pass

# tests/unit/test_negative_scenarios.py

class TestNegativeScenarios:
    """Test error paths and failure modes."""

    @pytest.mark.asyncio
    async def test_langgraph_api_authentication_failure():
        """Test graceful handling of API key errors."""
        context = RunnerContext(
            session_id="auth-fail-test",
            workspace_path="/tmp",
            environment={"ANTHROPIC_API_KEY": "invalid-key"}
        )
        adapter = LangGraphAdapter()
        result = await adapter.run()

        assert result["success"] is False
        assert "authentication" in result["error"].lower()
        assert "API key" in result["error"]

    @pytest.mark.asyncio
    async def test_repository_clone_failure():
        """Test immediate failure on repo access error."""
        context = RunnerContext(
            session_id="clone-fail-test",
            workspace_path="/tmp",
            environment={
                "REPOS_JSON": json.dumps([{
                    "name": "invalid-repo",
                    "input": {"url": "https://nonexistent.example.com/repo"}
                }])
            }
        )
        adapter = LangGraphAdapter()

        with pytest.raises(RuntimeError) as exc:
            await adapter.initialize(context)

        assert "repository" in str(exc.value).lower()
        assert "access" in str(exc.value).lower()

    @pytest.mark.asyncio
    async def test_execution_timeout():
        """Test graceful timeout handling."""
        # Setup mock LangGraph that doesn't complete
        # Wait for timeout period
        # Verify:
        #  - Session terminated
        #  - CR status updated to timeout/failed
        #  - Partial results preserved if any
        pass

    @pytest.mark.asyncio
    async def test_artifact_schema_validation_failure():
        """Test validation when artifact schema doesn't match."""
        # Force LangGraph to produce malformed artifact
        # Verify validation catches it
        # Verify helpful error message
        pass
```

---

#### Category 6: End-to-End Testing (Complete Session Lifecycle)

**Objective:** Test full session flow from creation through artifact production
**Coverage Target:** All RFE phases × cluster configurations
**Risk Mitigated:** Workflow failures, data loss, incorrect artifact output

```python
# tests/e2e/test_langgraph_e2e_workflows.py

@pytest.mark.asyncio
class TestLangGraphE2EWorkflows:
    """Test complete LangGraph session lifecycle."""

    async def test_e2e_ideate_phase():
        """Test complete Ideate phase: session → LangGraph execution → artifact."""
        # 1. Create AgenticSession CR with:
        #    - runnerType: langgraph
        #    - phase: ideate
        #    - prompt: "Create RFE for feature X"

        # 2. Operator provisions Job with LangGraph image

        # 3. LangGraphAdapter initializes, runs prompt

        # 4. WebSocket streams progress

        # 5. LangGraph produces rfe.md

        # 6. Artifacts committed to output repo

        # 7. CR status updated to Completed

        # Assertions:
        # - rfe.md exists in specs/{branchName}/
        # - Content follows RFE template
        # - Git commit created
        # - CR.status.phase = "Completed"
        # - CR.status.completionTime populated
        pass

    async def test_e2e_specify_phase():
        """Test Specify phase with existing rfe.md."""
        # 1. Create AgenticSession CR with:
        #    - runnerType: langgraph
        #    - phase: specify
        #    - prompt: "Create specification from RFE"
        #    (assumes rfe.md already exists)

        # 2-7. Same as ideate phase

        # Assertions:
        # - spec.md exists in specs/{branchName}/
        # - References rfe.md correctly
        # - Follows specification template
        pass

    async def test_e2e_plan_phase():
        """Test Plan phase with existing spec.md."""
        # Similar to specify phase
        # Produces plan.md
        pass

    async def test_e2e_tasks_phase():
        """Test Tasks phase with existing plan.md."""
        # Similar to specify phase
        # Produces tasks.md
        pass

    async def test_e2e_interactive_mode():
        """Test interactive session with multiple user interactions."""
        # 1. Create interactive session (interactive=true)

        # 2. WebSocket connects

        # 3. Send initial prompt

        # 4. Receive response

        # 5. Send follow-up user message

        # 6. Continue conversation cycle 3+ times

        # 7. Send end_session message

        # 8. Session terminates cleanly

        # Assertions:
        # - Each WebSocket message received within SLA
        # - Conversation context preserved
        # - Artifacts generated on completion
        pass

    async def test_e2e_multi_repo_workflow():
        """Test end-to-end with multiple repositories."""
        # 1. Create session with 3 repositories

        # 2. LangGraph executes workflow that:
        #    - Reads files from repo1
        #    - References configurations from repo2
        #    - Generates output intended for repo3

        # 3. Changes committed to appropriate repos

        # Assertions:
        # - repo1/artifact1.md committed
        # - repo2/artifact2.md committed
        # - repo3/artifact3.md committed
        # - Each commit captures correct changes
        pass

    async def test_e2e_error_recovery():
        """Test end-to-end error handling and recovery."""
        # 1. Create session that will fail midway

        # 2. Verify error reported immediately

        # 3. Verify partial results preserved

        # 4. Verify CR status reflects failure

        # 5. User can retry or review logs

        # Assertions:
        # - Error visible in WebSocket stream
        # - CR.status.phase = "Failed"
        # - Error message includes actionable details
        # - Partial work saved for review
        pass
```

#### 6.1 Cross-Runner Session Continuation Edge Case

```python
# tests/e2e/test_cross_runner_prevention.py

@pytest.mark.asyncio
class TestCrossRunnerPrevention:
    """Test prevention of cross-runner session continuation."""

    async def test_cannot_continue_claude_code_session_with_langgraph():
        """Verify system rejects continuing Claude Code with LangGraph runner."""
        # 1. Create initial session with runnerType: claude-code
        session1_id = "session-1"

        # 2. Create continuation request with runnerType: langgraph
        request = CreateAgenticSessionRequest(
            prompt="Continue work",
            parent_session_id=session1_id,
            runner_type="langgraph"
        )

        handler = CreateAgenticSessionHandler()

        # Should be rejected with clear error
        with pytest.raises(ValidationError) as exc:
            handler.create_session(request)

        error_msg = str(exc.value)
        assert "cross-runner" in error_msg.lower() or "continuation" in error_msg.lower()
        assert "same runner type" in error_msg.lower()

    async def test_cannot_continue_langgraph_session_with_claude_code():
        """Verify system rejects continuing LangGraph with Claude Code runner."""
        # 1. Create initial session with runnerType: langgraph
        session1_id = "session-1-langgraph"

        # 2. Create continuation request with runnerType: claude-code
        request = CreateAgenticSessionRequest(
            prompt="Continue work",
            parent_session_id=session1_id,
            runner_type="claude-code"
        )

        handler = CreateAgenticSessionHandler()

        with pytest.raises(ValidationError) as exc:
            handler.create_session(request)

        error_msg = str(exc.value)
        assert "cross-runner" in error_msg.lower()
```

---

## 2. Recommended Testing Frameworks & Tools

### 2.1 Python Testing Stack (Runner Adapter Layer)

| Tool | Purpose | Installation | Usage |
|------|---------|--------------|-------|
| **pytest** | Test framework | `pip install pytest==7.4.0` | Primary test runner for adapter tests |
| **pytest-asyncio** | Async test support | `pip install pytest-asyncio==0.21.0` | Run async adapter methods in tests |
| **pytest-cov** | Coverage reporting | `pip install pytest-cov==4.1.0` | Generate coverage reports |
| **pytest-timeout** | Timeout protection | `pip install pytest-timeout==2.1.0` | Prevent hung tests |
| **pytest-mock** | Mocking support | `pip install pytest-mock==3.11.1` | Mock Anthropic API, file I/O, git commands |
| **responses** | HTTP mocking | `pip install responses==0.23.1` | Mock HTTP API calls to backends |
| **websocket-client** | WebSocket testing | `pip install websocket-client==1.6.1` | Test WebSocket streaming |
| **unittest.mock** | Built-in mocking | Stdlib | Mock async functions, patches |

**Configuration: pytest.ini**
```ini
[pytest]
asyncio_mode = auto
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts =
    --cov=components/runners/langgraph-runner
    --cov-report=html
    --cov-report=term-missing
    --timeout=30
    -v
markers =
    integration: Integration tests requiring infrastructure
    e2e: End-to-end tests requiring full deployment
    regression: Regression tests for Claude Code compatibility
    asyncio: Async tests
```

### 2.2 Kubernetes & Operator Testing

| Tool | Purpose | Usage |
|------|---------|-------|
| **kind** | Local K8s cluster | `kind create cluster --name vteam-test` for local operator testing |
| **kubebuilder** | CRD & operator SDK | Use existing operator test patterns from vTeam codebase |
| **operator-sdk** | Operator testing utils | Test operator reconciliation logic |
| **kustomize** | K8s manifest testing | Validate manifest generation |
| **kubectl** | K8s CLI | Inspect created Jobs, CRs, Pods |

**Kind Cluster Setup:**
```bash
# Create Kind cluster with image preloading
kind create cluster --name vteam-langgraph-test --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 6443
        hostPort: 6443
        protocol: TCP
EOF

# Pre-load Docker images for faster testing
kind load docker-image vteam-operator:test --name vteam-langgraph-test
kind load docker-image langgraph-runner:test --name vteam-langgraph-test
kind load docker-image vteam-backend:test --name vteam-langgraph-test
```

### 2.3 Git & Repository Testing

| Tool | Purpose | Usage |
|------|---------|-------|
| **git** | Git operations | Direct git command execution for repo testing |
| **gitpython** | Python git library | GitPython for programmatic repo operations in tests |
| **temporary directories** | Isolated test repos | Use pytest tmp_path fixture for git testing |

```python
# tests/integration/fixtures/git_fixtures.py

@pytest.fixture
def local_git_repo(tmp_path):
    """Create a test git repository."""
    repo_path = tmp_path / "test_repo"
    repo_path.mkdir()

    # Initialize repo
    subprocess.run(["git", "init"], cwd=repo_path)
    subprocess.run(["git", "config", "user.name", "Test Bot"], cwd=repo_path)
    subprocess.run(["git", "config", "user.email", "test@example.com"], cwd=repo_path)

    # Create initial commit
    (repo_path / "README.md").write_text("# Test Repo")
    subprocess.run(["git", "add", "."], cwd=repo_path)
    subprocess.run(["git", "commit", "-m", "Initial commit"], cwd=repo_path)

    return repo_path
```

### 2.4 Contract Testing Approach

**Tools:**
- **Python dataclasses + type hints** for interface validation
- **inspect module** for runtime method signature validation
- **Protocol classes** (Python 3.8+) for formal interface definition

```python
# tests/contract/runner_adapter_contract.py

from typing import Protocol, Dict, Any
from runner_shell.core.context import RunnerContext

class RunnerAdapterContract(Protocol):
    """Formal interface definition for runner adapters."""

    async def initialize(self, context: RunnerContext) -> None:
        """Initialize adapter with session context."""
        ...

    async def run(self) -> Dict[str, Any]:
        """Execute the runner session."""
        ...

    async def handle_message(self, message: Dict[str, Any]) -> None:
        """Handle incoming WebSocket messages."""
        ...


# tests/contract/test_langgraph_contract_compliance.py

from components.runners.langgraph_runner.adapter import LangGraphAdapter

class TestLangGraphContractCompliance:
    """Test LangGraphAdapter complies with RunnerAdapterContract."""

    def test_implements_runner_adapter_contract():
        """Verify LangGraphAdapter implements all required methods."""
        adapter = LangGraphAdapter()

        # Check methods exist
        assert hasattr(adapter, 'initialize')
        assert hasattr(adapter, 'run')
        assert hasattr(adapter, 'handle_message')

        # Check methods are callable
        assert callable(adapter.initialize)
        assert callable(adapter.run)
        assert callable(adapter.handle_message)

        # Check methods are async
        import inspect
        assert inspect.iscoroutinefunction(adapter.initialize)
        assert inspect.iscoroutinefunction(adapter.run)
        assert inspect.iscoroutinefunction(adapter.handle_message)

    def test_initialize_signature_matches():
        """Verify initialize(context: RunnerContext) signature."""
        import inspect
        sig = inspect.signature(LangGraphAdapter.initialize)
        params = list(sig.parameters.keys())

        assert 'self' in params
        assert 'context' in params

        # Verify type hint
        context_param = sig.parameters['context']
        assert context_param.annotation == RunnerContext

    def test_run_return_type_matches():
        """Verify run() returns Dict with success/result/error keys."""
        import inspect
        sig = inspect.signature(LangGraphAdapter.run)

        # Return type should be Dict
        assert 'Dict' in str(sig.return_annotation)

    def test_handle_message_signature_matches():
        """Verify handle_message(message: Dict) signature."""
        import inspect
        sig = inspect.signature(LangGraphAdapter.handle_message)
        params = list(sig.parameters.keys())

        assert 'self' in params
        assert 'message' in params
```

---

## 3. Test Execution & CI/CD Integration

### 3.1 Local Test Execution

```bash
# Run all tests
pytest tests/ -v

# Run specific test category
pytest tests/unit/ -v -m "not integration"
pytest tests/integration/ -v
pytest tests/e2e/ -v -m integration
pytest tests/regression/ -v -k "claude_code"

# Run with coverage
pytest tests/ --cov=components/runners/langgraph-runner --cov-report=html

# Run with specific markers
pytest tests/ -m "asyncio" -v
pytest tests/ -m "integration" --timeout=60 -v

# Run specific test class
pytest tests/integration/test_operator_runner_selection.py::TestOperatorRunnerSelection -v

# Run with detailed output
pytest tests/ -vv --tb=long
```

### 3.2 CI/CD Pipeline Integration

**Current GitHub Actions:** `.github/workflows/components-build-deploy.yml`

**Proposed Additions for LangGraph Testing:**

```yaml
# .github/workflows/langgraph-tests.yml

name: LangGraph Runner Tests

on:
  push:
    branches: [main, ambient-langgraph-runner]
    paths:
      - 'components/runners/langgraph-runner/**'
      - 'components/operator/**'
      - 'components/backend/**'
      - 'tests/**'
      - '.github/workflows/langgraph-tests.yml'

jobs:
  unit-tests:
    name: Unit & Contract Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        working-directory: components/runners/langgraph-runner
        run: |
          pip install -r requirements-test.txt
          pip install -r requirements.txt

      - name: Run unit tests
        working-directory: components/runners/langgraph-runner
        run: |
          pytest tests/unit/ -v --cov=. --cov-report=xml

      - name: Run contract tests
        working-directory: components/runners/langgraph-runner
        run: |
          pytest tests/contract/ -v

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          files: ./components/runners/langgraph-runner/coverage.xml
          flags: langgraph-runner

  integration-tests:
    name: Integration Tests (Kind Cluster)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up Kind cluster
        uses: helm/kind-action@v1.7.0
        with:
          cluster_name: vteam-test
          config: hack/kind-config.yaml

      - name: Build operator image
        run: |
          cd components/operator
          docker build -t vteam-operator:test .
          kind load docker-image vteam-operator:test --name vteam-test

      - name: Build LangGraph runner image
        run: |
          cd components/runners/langgraph-runner
          docker build -t langgraph-runner:test .
          kind load docker-image langgraph-runner:test --name vteam-test

      - name: Deploy operator and dependencies
        run: |
          kubectl create namespace ambient-code
          # Apply operator manifests
          kubectl apply -f components/operator/deploy/ -n ambient-code

      - name: Run integration tests
        working-directory: components/runners/langgraph-runner
        run: |
          pytest tests/integration/ -v -m integration --timeout=120

  regression-tests:
    name: Claude Code Regression
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up Kind cluster
        uses: helm/kind-action@v1.7.0
        with:
          cluster_name: vteam-test

      - name: Build Claude Code runner image
        run: |
          cd components/runners/claude-code-runner
          docker build -t claude-code-runner:test .
          kind load docker-image claude-code-runner:test --name vteam-test

      - name: Run regression tests
        working-directory: components/runners/claude-code-runner
        run: |
          pytest tests/regression/ -v

  api-validation-tests:
    name: API Validation Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install backend dependencies
        working-directory: components/backend
        run: |
          pip install -r requirements.txt
          pip install -r requirements-test.txt

      - name: Run API validation tests
        working-directory: components/backend
        run: |
          pytest tests/unit/test_api_runner_type_validation.py -v
          pytest tests/unit/test_api_error_messages.py -v

  e2e-tests:
    name: End-to-End Tests
    runs-on: ubuntu-latest
    # Only run on main branch or explicit trigger
    if: github.ref == 'refs/heads/main' || github.event_name == 'workflow_dispatch'
    steps:
      - uses: actions/checkout@v3

      - name: Set up Kind cluster
        uses: helm/kind-action@v1.7.0
        with:
          cluster_name: vteam-e2e

      - name: Build all components
        run: |
          for component in operator backend langgraph-runner claude-code-runner; do
            cd components/$component
            docker build -t vteam-$component:test .
            kind load docker-image vteam-$component:test --name vteam-e2e
            cd ../..
          done

      - name: Deploy full stack
        run: |
          # Deploy all components
          kubectl create namespace vteam
          kubectl apply -f components/manifests/ -n vteam

      - name: Run E2E tests
        run: |
          pytest tests/e2e/ -v -m e2e --timeout=300

  test-report:
    name: Test Report Summary
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests, regression-tests, api-validation-tests]
    if: always()
    steps:
      - name: Generate summary
        run: |
          echo "## Test Results" >> $GITHUB_STEP_SUMMARY
          echo "- Unit Tests: ${{ needs.unit-tests.result }}" >> $GITHUB_STEP_SUMMARY
          echo "- Integration Tests: ${{ needs.integration-tests.result }}" >> $GITHUB_STEP_SUMMARY
          echo "- Regression Tests: ${{ needs.regression-tests.result }}" >> $GITHUB_STEP_SUMMARY
          echo "- API Tests: ${{ needs.api-validation-tests.result }}" >> $GITHUB_STEP_SUMMARY
```

### 3.3 Pre-Deployment Checklist

**Must Pass Before Deploying to Production:**

```
CATEGORY: Unit & Contract Tests
- [ ] All contract tests passing (100% interface compliance)
- [ ] All unit tests passing (>90% code coverage)
- [ ] No lint errors (pylint, black formatting)
- [ ] Type hints validated (mypy)

CATEGORY: Integration Tests (Local Kind Cluster)
- [ ] Operator correctly selects LangGraph runner image (runnerType=langgraph)
- [ ] Operator correctly selects Claude Code runner image (runnerType=claude-code)
- [ ] Operator defaults to Claude Code when runnerType not specified
- [ ] All environment variables propagated to Job
- [ ] Multi-repo cloning works (3+ repos)
- [ ] WebSocket streaming messages received within SLA (<2 seconds)
- [ ] Error details streamed on failures

CATEGORY: API Validation (Backend Tests)
- [ ] POST /sessions accepts runnerType parameter
- [ ] runnerType="claude-code" accepted
- [ ] runnerType="langgraph" accepted
- [ ] runnerType not specified defaults to claude-code
- [ ] Invalid runnerType rejected with 400 + helpful error message
- [ ] Error messages include "Supported types: claude-code, langgraph"

CATEGORY: Regression Tests (Claude Code)
- [ ] Claude Code single-repo execution identical to baseline
- [ ] Claude Code multi-repo execution works unchanged
- [ ] Claude Code default behavior unchanged (no runnerType specified)
- [ ] Claude Code artifact generation format unchanged
- [ ] Claude Code interactive mode unchanged
- [ ] Claude Code error handling unchanged
- [ ] Claude Code session continuation unchanged
- [ ] 100% of Claude Code tests passing

CATEGORY: End-to-End Tests (Pre-Release Candidate Clusters)
- [ ] LangGraph Ideate phase complete end-to-end
- [ ] LangGraph Specify phase complete end-to-end
- [ ] LangGraph Plan phase complete end-to-end
- [ ] LangGraph Tasks phase complete end-to-end
- [ ] LangGraph interactive mode end-to-end
- [ ] LangGraph multi-repo end-to-end
- [ ] LangGraph error recovery end-to-end
- [ ] Cross-runner continuation prevented

CATEGORY: Cluster Configurations
- [ ] Standard OpenShift cluster with RC RHOAI deployment
- [ ] FIPS-enabled OpenShift cluster with RC RHOAI deployment
- [ ] Disconnected OpenShift cluster with RC RHOAI deployment
- [ ] GPU-enabled cluster with RC RHOAI deployment
- [ ] Multiple architecture support validated

CATEGORY: Documentation & Support
- [ ] README updated with runner selection instructions
- [ ] Error messages provide actionable guidance
- [ ] Support runbook includes LangGraph debugging steps
- [ ] User documentation includes LangGraph examples
```

---

## 4. Test Data Management

### 4.1 Test Fixture Strategy

**Objective:** Minimize test data duplication, ensure consistency, enable reusability

```python
# tests/conftest.py - Global pytest configuration and fixtures

import pytest
import os
from pathlib import Path
import asyncio
from unittest.mock import AsyncMock, Mock

# Async event loop fixture
@pytest.fixture(scope="session")
def event_loop():
    """Create event loop for async tests."""
    loop = asyncio.get_event_loop_policy().new_event_loop()
    yield loop
    loop.close()

# Mock environment variables
@pytest.fixture
def mock_env(monkeypatch):
    """Fixture providing mocked environment for runners."""
    env = {
        "SESSION_ID": "test-session-123",
        "WORKSPACE_PATH": "/tmp/test-workspace",
        "ANTHROPIC_API_KEY": "sk-test-key-12345",
        "GITHUB_TOKEN": "ghp_test_token_12345",
        "GIT_USER_NAME": "Test Bot",
        "GIT_USER_EMAIL": "test@example.com",
    }
    for key, value in env.items():
        monkeypatch.setenv(key, value)
    return env

# Sample AgenticSession fixtures
@pytest.fixture
def langgraph_session_request():
    """CreateAgenticSessionRequest for LangGraph runner."""
    from components.backend.types import CreateAgenticSessionRequest
    return CreateAgenticSessionRequest(
        prompt="Create a feature specification",
        runner_type="langgraph",
        display_name="Test LangGraph Session",
        interactive=False,
        timeout=300,
        llm_settings={
            "model": "claude-3-5-sonnet-20241022",
            "temperature": 0.7,
            "max_tokens": 4096,
        }
    )

@pytest.fixture
def claude_code_session_request():
    """CreateAgenticSessionRequest for Claude Code runner."""
    from components.backend.types import CreateAgenticSessionRequest
    return CreateAgenticSessionRequest(
        prompt="Create a feature specification",
        runner_type="claude-code",
        display_name="Test Claude Code Session",
        interactive=False,
        timeout=300,
        llm_settings={
            "model": "claude-3-5-sonnet-20241022",
            "temperature": 0.7,
            "max_tokens": 4096,
        }
    )

@pytest.fixture
def multi_repo_config():
    """Multi-repository configuration."""
    import json
    return json.dumps([
        {
            "name": "main-repo",
            "input": {
                "url": "https://github.com/test-org/main-repo",
                "branch": "main"
            },
            "output": {
                "url": "https://github.com/test-org/main-repo-output",
                "branch": "results"
            }
        },
        {
            "name": "config-repo",
            "input": {
                "url": "https://github.com/test-org/config-repo",
                "branch": "develop"
            }
        },
        {
            "name": "docs-repo",
            "input": {
                "url": "https://github.com/test-org/docs-repo",
                "branch": "main"
            }
        }
    ])

# Mock WebSocket
@pytest.fixture
def mock_websocket():
    """Mock WebSocket for testing message streaming."""
    mock_ws = AsyncMock()
    mock_ws.send = AsyncMock()
    mock_ws.recv = AsyncMock(return_value='{"type": "ping"}')
    return mock_ws

# Mock RunnerShell
@pytest.fixture
def mock_runner_shell():
    """Mock RunnerShell for adapter testing."""
    from unittest.mock import AsyncMock
    mock_shell = AsyncMock()
    mock_shell._send_message = AsyncMock()
    mock_shell.transport = Mock()
    mock_shell.transport.url = "wss://backend:8080/ws/session-123"
    return mock_shell

# Mock Anthropic client
@pytest.fixture
def mock_anthropic_client():
    """Mock Anthropic Claude client."""
    from unittest.mock import AsyncMock, Mock

    mock_client = AsyncMock()
    mock_response = Mock()
    mock_response.content = [Mock(text="Sample LangGraph output")]
    mock_client.messages.create = AsyncMock(return_value=mock_response)

    return mock_client
```

### 4.2 Sample Test Data Files

**Location:** `tests/fixtures/`

```yaml
# tests/fixtures/sample-rfe.md
# RFE: Feature Example

## Problem
Users need a way to select between different execution runners.

## Proposed Solution
Add runner selection UI and backend validation.

## Success Criteria
- Users can select "Claude Code" or "LangGraph" in UI
- API validates runnerType parameter
- Sessions execute with selected runner
```

```yaml
# tests/fixtures/sample-spec.md
# Specification: Feature Example

## Overview
This specification defines runner selection functionality.

## Detailed Requirements
1. UI Components
   - Runner type dropdown
   - Help text explaining differences
2. Backend API
   - Accept runnerType parameter
   - Validate against allowed values
   - Default to claude-code

## Acceptance Criteria
- Runner dropdown displayed in session creation
- API rejects invalid runner types with 400
- Default to Claude Code when not specified
```

```json
// tests/fixtures/agentic-session-cr.json
{
  "apiVersion": "ambient-code.io/v1",
  "kind": "AgenticSession",
  "metadata": {
    "name": "test-session-langgraph",
    "namespace": "default"
  },
  "spec": {
    "prompt": "Create a test specification",
    "runnerType": "langgraph",
    "interactive": false,
    "displayName": "Test LangGraph Session",
    "timeout": 300,
    "llmSettings": {
      "model": "claude-3-5-sonnet-20241022",
      "temperature": 0.7
    },
    "repos": [
      {
        "input": {
          "url": "https://github.com/test/repo",
          "branch": "main"
        },
        "output": {
          "url": "https://github.com/test/repo-output",
          "branch": "results"
        }
      }
    ]
  }
}
```

---

## 5. Success Metrics & Acceptance

### 5.1 Test Coverage Requirements

| Category | Target | Acceptance Criteria |
|----------|--------|-------------------|
| **Contract Tests** | 100% | All interface methods covered, typed signatures validated |
| **Unit Tests** | >85% | Code coverage report generated, all critical paths tested |
| **Integration Tests** | >80% | Operator behavior, multi-repo, WebSocket streaming |
| **Regression Tests** | 100% | All Claude Code workflows must pass identically |
| **API Tests** | 100% | All runnerType validation scenarios covered |
| **E2E Tests** | All RFE phases | Ideate, Specify, Plan, Tasks complete lifecycle |

### 5.2 Quality Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| **Test Execution Time** | <10 min (unit) | Local pytest run time |
| | <30 min (integration) | Kind cluster tests |
| | <5 min (API) | Backend validation tests |
| **Test Stability** | 99% pass rate | Failures due to logic, not flakiness |
| **Code Coverage** | >85% | LangGraphAdapter implementation |
| **Documentation** | 100% | All test scenarios documented with examples |
| **Error Message Quality** | 100% | All errors include actionable guidance |

### 5.3 Release Gate Checklist

```
MUST PASS BEFORE RELEASE:
[ ] All unit tests passing (pytest tests/unit/ -v)
[ ] All contract tests passing (pytest tests/contract/ -v)
[ ] All API validation tests passing (pytest tests/unit/test_api*.py -v)
[ ] All regression tests passing (pytest tests/regression/ -v)
[ ] Integration tests passing on Kind cluster (pytest tests/integration/ -v)
[ ] No new code coverage regressions (<1% drop)
[ ] No lint errors in new code
[ ] Type hints validated with mypy
[ ] Documentation complete and accurate
[ ] Cross-runner prevention validated
[ ] Claude Code execution unchanged
```

---

## 6. Critical Risk Areas & Mitigation

### 6.1 High-Risk Areas

| Risk | Impact | Mitigation |
|------|--------|-----------|
| **Claude Code regression** | Blocks release, user impact | 100% regression test suite, manual smoke test before deploy |
| **Cross-runner contamination** | Data loss, workflow corruption | Strict interface contract, no shared state |
| **API validation bypass** | Invalid runners accepted | Comprehensive parameter validation tests, code review |
| **WebSocket timeout** | Poor UX, unclear failures | SLA testing (2-second latency), timeout scenario tests |
| **Multi-repo conflicts** | Git merge issues, data corruption | Test with 3-5 repos, independent git operations per repo |
| **Kubernetes operator issues** | Job creation failures | Operator integration tests on Kind cluster, manifest validation |
| **LLM API failures** | Silent failures or cryptic errors | Mock API testing, error message quality validation |

### 6.2 Testing Strategy to Address Risks

```
Risk: Claude Code Regression
├─ Mitigation: Regression Test Suite
│  ├─ Baseline comparison (before/after LangGraph)
│  ├─ All existing Claude Code workflows
│  ├─ Single-repo and multi-repo scenarios
│  ├─ Interactive and batch modes
│  └─ Error conditions
├─ Validation: Manual smoke test
│  ├─ Create Claude Code session
│  ├─ Verify execution identical to baseline
│  └─ Verify artifacts unchanged

Risk: Operator Image Selection
├─ Mitigation: Integration Tests
│  ├─ Test runnerType=langgraph uses LangGraph image
│  ├─ Test runnerType=claude-code uses Claude Code image
│  ├─ Test default (no runnerType) uses Claude Code image
│  ├─ Verify environment variables propagated
│  └─ Test with invalid runner types (should fail fast)
├─ Validation: Inspect created Job manifests
│  └─ kubectl describe job <job-name>

Risk: WebSocket Streaming Failures
├─ Mitigation: WebSocket SLA Tests
│  ├─ Message arrival within 2 seconds
│  ├─ State transition visibility
│  ├─ Error detail streaming
│  ├─ Large output fragmentation
│  └─ Connection failure recovery
├─ Validation: WebSocket timing metrics
│  └─ Record and assert timestamp deltas

Risk: Multi-Repo Git Conflicts
├─ Mitigation: Multi-Repo Integration Tests
│  ├─ Test 3+ repos cloned correctly
│  ├─ Test independent git operations per repo
│  ├─ Test commit/push to correct repos
│  ├─ Test output mapping
│  └─ Test cross-repo references
├─ Validation: Verify directory structure and remotes
│  └─ ls -la /tmp/workspace && git remote -v
```

---

## 7. Implementation Roadmap

### Phase 1: Foundation (Week 1-2)

- [ ] Set up pytest configuration and fixtures
- [ ] Create RunnerContext and protocol contract definitions
- [ ] Implement contract tests for adapter interface
- [ ] Set up Kind cluster for integration testing
- [ ] Create test data fixtures (sample artifacts, configs)
- [ ] Build mock WebSocket and Anthropic client

**Deliverable:** Foundation test infrastructure, 0% new functionality tested

### Phase 2: Adapter Testing (Week 2-3)

- [ ] Implement LangGraphAdapter (development)
- [ ] Create unit tests for adapter methods
- [ ] Add async/await pattern tests
- [ ] Test environment variable handling
- [ ] Test error handling paths
- [ ] Achieve >85% code coverage

**Deliverable:** Complete adapter tested, integration ready

### Phase 3: Integration Testing (Week 3-4)

- [ ] Operator image selection tests
- [ ] Multi-repo cloning tests
- [ ] WebSocket streaming tests
- [ ] Operator manifest validation
- [ ] Environment propagation tests
- [ ] Error scenario tests

**Deliverable:** Full integration tested end-to-end with operator

### Phase 4: API & Regression (Week 4)

- [ ] Backend API validation tests
- [ ] RunnerType parameter tests
- [ ] Error message quality tests
- [ ] Claude Code regression suite
- [ ] Backward compatibility tests
- [ ] Cross-runner prevention tests

**Deliverable:** API validation complete, zero Claude Code regression

### Phase 5: E2E & Release (Week 5)

- [ ] End-to-end workflow tests (all RFE phases)
- [ ] Interactive mode tests
- [ ] Multi-cluster configuration tests (Standard, FIPS, GPU)
- [ ] Performance baseline tests
- [ ] Documentation and runbooks
- [ ] Release gate validation

**Deliverable:** Production-ready with comprehensive test coverage

---

## 8. Test Environment Requirements

### 8.1 Local Development

```bash
# Minimum requirements
- Python 3.11+
- Docker 20.10+
- kubectl 1.24+
- kind 0.20+
- git 2.30+
- 16GB RAM, 50GB disk space

# Installation
git clone https://github.com/yourgithub/vteam.git
cd vteam/components/runners/langgraph-runner
pip install -r requirements-test.txt
pip install -r requirements.txt

# Run local tests
pytest tests/unit/ -v
pytest tests/contract/ -v

# Run integration tests
kind create cluster --name vteam-test
pytest tests/integration/ -v -m integration
```

### 8.2 CI/CD Environment

```yaml
# GitHub Actions Runner
os: ubuntu-latest
python: 3.11
docker: latest
kubernetes: kind v0.20

# Workflow execution time targets
unit-tests: <5 minutes
integration-tests: <15 minutes
regression-tests: <10 minutes
api-tests: <5 minutes
e2e-tests: <20 minutes (on-demand only)
```

### 8.3 Pre-Release Testing Environments

```
1. Standard OpenShift Cluster
   - OpenShift 4.13+
   - RHOAI Release Candidate
   - 3 control nodes, 3 worker nodes
   - LangGraph runner image available
   - Claude Code runner image available

2. FIPS-Enabled OpenShift Cluster
   - Same as above with FIPS kernel
   - FIPS-compliant Python environment
   - All cryptographic operations validated

3. Disconnected OpenShift Cluster
   - No internet access
   - Pre-pulled container images
   - All dependencies pre-cached
   - Git repos available via mirror

4. GPU-Enabled Cluster
   - NVIDIA GPU nodes
   - CUDA 12+ support
   - GPU workload testing
   - Multi-architecture (arm64 + x86_64)
```

---

## 9. Troubleshooting & Support

### 9.1 Common Test Failures

| Failure | Root Cause | Resolution |
|---------|-----------|-----------|
| `asyncio.TimeoutError` | Slow adapter initialization | Increase pytest timeout, check resource limits |
| `WebSocket connection refused` | Mock WebSocket not started | Verify mock_runner_shell fixture initialized |
| `Git command failed` | Credentials missing | Provide GITHUB_TOKEN in environment |
| `Image not found` | Kind cluster image not loaded | `kind load docker-image <image> --name <cluster>` |
| `Operator not reconciling` | CRD not installed | Deploy operator manifests first |

### 9.2 Test Execution Commands

```bash
# Run all tests
pytest tests/ -v

# Run specific category
pytest tests/unit/ -v
pytest tests/integration/ -v --timeout=120
pytest tests/regression/ -v

# Run with detailed logging
pytest tests/ -vv --log-cli-level=DEBUG

# Run with coverage
pytest tests/ --cov=components/runners/langgraph-runner --cov-report=html

# Run failing tests only
pytest tests/ --lf -v

# Run with specific marker
pytest tests/ -m asyncio -v

# Parallel execution (if testpaths don't conflict)
pytest tests/ -n auto
```

---

## 10. Approval & Sign-Off

This testing strategy has been reviewed and approved by:

| Role | Name | Signature | Date |
|------|------|-----------|------|
| QA Lead | [Pending] | | |
| Development Lead | [Pending] | | |
| Platform Lead | [Pending] | | |
| Product Manager | [Pending] | | |

---

## 11. Appendix: Reference Documents

### 11.1 Related Specifications

- `/workspace/sessions/agentic-session-1762293354/workspace/spec-umberella/specs/001-langgraph-runner-support/spec.md` - Feature specification
- `components/runners/runner-shell/` - Runner-shell framework
- `components/operator/` - Kubernetes operator implementation
- `components/backend/types/session.go` - AgenticSession CR definition

### 11.2 External References

- [pytest Documentation](https://docs.pytest.org/)
- [pytest-asyncio](https://pytest-asyncio.readthedocs.io/)
- [Kind Cluster Testing](https://kind.sigs.k8s.io/)
- [Kubernetes Testing Patterns](https://kubernetes.io/docs/concepts/testing/)
- [Python Async Testing](https://docs.python.org/3/library/asyncio-dev.html)

### 11.3 Tools & Configuration Files

- `pytest.ini` - pytest configuration
- `.github/workflows/langgraph-tests.yml` - CI/CD pipeline
- `hack/kind-config.yaml` - Kind cluster configuration
- `tests/conftest.py` - Global pytest fixtures
- `tests/fixtures/` - Sample test data

---

**Document Owner:** Neil (QA Architect)
**Last Updated:** 2025-11-04
**Next Review:** Post-Alpha Release

