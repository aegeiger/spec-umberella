# LangGraph Runner Test Implementation Guide

**Document Version:** 1.0
**Created:** 2025-11-04
**Purpose:** Practical implementation examples for testing framework

---

## 1. Quick Start: Setting Up Local Testing

### 1.1 Directory Structure

```
components/runners/langgraph-runner/
├── langgraph_runner/
│   ├── __init__.py
│   ├── adapter.py              # LangGraphAdapter implementation
│   ├── graph.py                # LangGraph state machine
│   ├── tools.py                # Tool definitions
│   └── utils.py                # Utility functions
├── tests/
│   ├── __init__.py
│   ├── conftest.py             # Pytest fixtures
│   ├── contract/
│   │   ├── __init__.py
│   │   └── test_adapter_contract.py
│   ├── unit/
│   │   ├── __init__.py
│   │   ├── test_adapter_methods.py
│   │   ├── test_error_handling.py
│   │   └── test_message_handling.py
│   ├── integration/
│   │   ├── __init__.py
│   │   ├── test_operator_integration.py
│   │   ├── test_websocket_streaming.py
│   │   └── test_multi_repo.py
│   ├── fixtures/
│   │   ├── __init__.py
│   │   ├── agentic_sessions.py
│   │   ├── git_repos.py
│   │   └── sample_data.py
│   └── e2e/
│       ├── __init__.py
│       └── test_workflows.py
├── requirements.txt
├── requirements-test.txt
├── pytest.ini
├── pyproject.toml
└── Dockerfile
```

### 1.2 Installation & Setup

```bash
#!/bin/bash
# setup-test-environment.sh

set -e

# Navigate to langgraph-runner component
cd components/runners/langgraph-runner

# Create virtual environment
python3.11 -m venv venv
source venv/bin/activate

# Install runtime dependencies
pip install -r requirements.txt

# Install testing dependencies
pip install -r requirements-test.txt

# Install pre-commit hooks (optional)
pre-commit install

# Run initial verification
pytest tests/ -v --collect-only
echo "Test environment ready!"
```

### 1.3 requirements-test.txt

```
# Testing frameworks
pytest==7.4.0
pytest-asyncio==0.21.0
pytest-cov==4.1.0
pytest-timeout==2.1.0
pytest-mock==3.11.1

# Mocking and fixtures
responses==0.23.1
websocket-client==1.6.1
faker==19.0.0

# Code quality
pylint==2.17.5
black==23.9.1
mypy==1.5.0
isort==5.12.0

# Kubernetes testing
kubernetes==28.0.0

# Git operations
GitPython==3.1.37

# Type stubs
types-requests
types-websocket
```

---

## 2. Pytest Configuration & Fixtures

### 2.1 pytest.ini

```ini
[pytest]
# Python 3.11+ async support
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

# Test markers
markers =
    unit: Unit tests (no external dependencies)
    contract: Contract/interface compliance tests
    integration: Integration tests (requires infrastructure)
    e2e: End-to-end tests (requires full deployment)
    asyncio: Async test functions
    slow: Tests that take >5 seconds
    websocket: WebSocket-related tests
    k8s: Kubernetes-related tests
```

### 2.2 pyproject.toml

```toml
[build-system]
requires = ["setuptools>=65.0"]
build-backend = "setuptools.build_meta"

[project]
name = "langgraph-runner"
version = "0.1.0"
description = "LangGraph-based runner for vTeam Ambient Code"
requires-python = ">=3.11"

[tool.black]
line-length = 100
target-version = ['py311']

[tool.isort]
profile = "black"
line_length = 100

[tool.mypy]
python_version = "3.11"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = false

[tool.pytest.ini_options]
asyncio_mode = "auto"
```

### 2.3 tests/conftest.py - Global Fixtures

```python
"""Global pytest configuration and fixtures for LangGraph runner tests."""

import os
import json
import asyncio
import pytest
from pathlib import Path
from unittest.mock import AsyncMock, Mock, MagicMock
from typing import Dict, Any

# ============================================================================
# Event Loop Fixture
# ============================================================================

@pytest.fixture(scope="session")
def event_loop():
    """Create and set up event loop for async tests."""
    loop = asyncio.get_event_loop_policy().new_event_loop()
    asyncio.set_event_loop(loop)
    yield loop
    loop.close()


# ============================================================================
# Environment Fixtures
# ============================================================================

@pytest.fixture
def mock_env(monkeypatch):
    """Mock environment variables for runner execution."""
    env_vars = {
        "SESSION_ID": "test-session-123456",
        "WORKSPACE_PATH": "/tmp/test-workspace",
        "ANTHROPIC_API_KEY": "sk-ant-test-key-12345",
        "GITHUB_TOKEN": "ghp_test_token_12345",
        "GIT_USER_NAME": "Test LangGraph Bot",
        "GIT_USER_EMAIL": "langgraph-bot@ambient-code.local",
        "BACKEND_API_URL": "http://backend:8080",
        "PROJECT_NAME": "test-project",
        "BOT_TOKEN": "bearer-test-token",
    }

    for key, value in env_vars.items():
        monkeypatch.setenv(key, value)

    return env_vars


@pytest.fixture
def clean_workspace(tmp_path):
    """Create clean workspace directory for testing."""
    workspace = tmp_path / "workspace"
    workspace.mkdir(parents=True, exist_ok=True)

    # Initialize git configuration
    (workspace / ".gitconfig").write_text(
        "[user]\n    name = Test Bot\n    email = test@example.com\n"
    )

    return workspace


# ============================================================================
# RunnerContext Fixtures
# ============================================================================

@pytest.fixture
def runner_context(clean_workspace, mock_env):
    """Create RunnerContext for testing."""
    from runner_shell.core.context import RunnerContext

    return RunnerContext(
        session_id=mock_env["SESSION_ID"],
        workspace_path=str(clean_workspace),
        environment=mock_env,
    )


# ============================================================================
# Request Fixtures (Type Objects)
# ============================================================================

@pytest.fixture
def langgraph_session_request() -> Dict[str, Any]:
    """CreateAgenticSessionRequest for LangGraph runner."""
    return {
        "prompt": "Create a simple feature specification for a todo app",
        "runner_type": "langgraph",
        "display_name": "Test LangGraph Session",
        "interactive": False,
        "timeout": 300,
        "llm_settings": {
            "model": "claude-3-5-sonnet-20241022",
            "temperature": 0.7,
            "max_tokens": 4096,
        },
        "repos": [
            {
                "input": {
                    "url": "https://github.com/test-org/main-repo",
                    "branch": "main",
                },
                "output": {
                    "url": "https://github.com/test-org/main-repo-output",
                    "branch": "results",
                },
            }
        ],
    }


@pytest.fixture
def claude_code_session_request() -> Dict[str, Any]:
    """CreateAgenticSessionRequest for Claude Code runner."""
    return {
        "prompt": "Create a simple feature specification for a todo app",
        "runner_type": "claude-code",
        "display_name": "Test Claude Code Session",
        "interactive": False,
        "timeout": 300,
        "llm_settings": {
            "model": "claude-3-5-sonnet-20241022",
            "temperature": 0.7,
            "max_tokens": 4096,
        },
    }


@pytest.fixture
def multi_repo_config() -> str:
    """Multi-repository configuration as REPOS_JSON."""
    repos = [
        {
            "name": "main-repo",
            "input": {
                "url": "https://github.com/test-org/main-repo",
                "branch": "main",
            },
            "output": {
                "url": "https://github.com/test-org/main-repo-output",
                "branch": "results",
            },
        },
        {
            "name": "config-repo",
            "input": {
                "url": "https://github.com/test-org/config-repo",
                "branch": "develop",
            },
        },
        {
            "name": "docs-repo",
            "input": {
                "url": "https://github.com/test-org/docs-repo",
                "branch": "main",
            },
        },
    ]
    return json.dumps(repos)


# ============================================================================
# Mock Object Fixtures
# ============================================================================

@pytest.fixture
def mock_runner_shell():
    """Mock RunnerShell for testing adapter."""
    shell = AsyncMock()
    shell.session_id = "test-session-123"
    shell._send_message = AsyncMock()
    shell.transport = Mock()
    shell.transport.url = "wss://backend:8080/api/projects/test-project/sessions/test-session-123/ws"
    return shell


@pytest.fixture
def mock_anthropic_client():
    """Mock Anthropic Claude API client."""
    client = AsyncMock()

    # Mock successful message response
    mock_message = Mock()
    mock_message.content = [Mock(text="Sample LangGraph output")]
    mock_message.usage = Mock(
        input_tokens=100,
        output_tokens=50,
    )

    client.messages.create = AsyncMock(return_value=mock_message)

    return client


@pytest.fixture
def mock_websocket_server():
    """Mock WebSocket server for testing streaming."""
    import asyncio
    from unittest.mock import AsyncMock

    class MockWebSocketServer:
        def __init__(self):
            self.messages_sent = []
            self.messages_received = []

        async def send(self, message):
            self.messages_sent.append(message)

        async def recv(self):
            if self.messages_received:
                return self.messages_received.pop(0)
            await asyncio.sleep(0.1)
            return None

    return MockWebSocketServer()


@pytest.fixture
def mock_git_repo(tmp_path):
    """Create mock git repository."""
    import subprocess

    repo_path = tmp_path / "test_repo"
    repo_path.mkdir()

    # Initialize repo
    subprocess.run(["git", "init"], cwd=repo_path, check=True)
    subprocess.run(["git", "config", "user.name", "Test Bot"], cwd=repo_path, check=True)
    subprocess.run(["git", "config", "user.email", "test@example.com"], cwd=repo_path, check=True)

    # Create initial commit
    (repo_path / "README.md").write_text("# Test Repository\n")
    (repo_path / "specs").mkdir()
    (repo_path / "specs" / "rfe.md").write_text("# RFE Template\n")

    subprocess.run(["git", "add", "."], cwd=repo_path, check=True)
    subprocess.run(["git", "commit", "-m", "Initial commit"], cwd=repo_path, check=True)

    return repo_path


# ============================================================================
# Marker Registration
# ============================================================================

def pytest_configure(config):
    """Register custom markers."""
    config.addinivalue_line("markers", "contract: Contract/interface tests")
    config.addinivalue_line("markers", "integration: Integration tests")
    config.addinivalue_line("markers", "e2e: End-to-end tests")
    config.addinivalue_line("markers", "websocket: WebSocket streaming tests")
    config.addinivalue_line("markers", "k8s: Kubernetes-related tests")
```

---

## 3. Contract Testing Implementation

### 3.1 tests/contract/test_adapter_contract.py

```python
"""Test LangGraphAdapter implements RunnerAdapter contract."""

import pytest
import inspect
from typing import Dict, Any
from runner_shell.core.context import RunnerContext
from runner_shell.core.protocol import MessageType


class TestLangGraphAdapterContract:
    """Verify LangGraphAdapter interface compliance."""

    def test_class_exists(self):
        """LangGraphAdapter class must exist."""
        from langgraph_runner.adapter import LangGraphAdapter
        assert LangGraphAdapter is not None

    def test_has_initialize_method(self):
        """Adapter must have async initialize(context: RunnerContext) method."""
        from langgraph_runner.adapter import LangGraphAdapter

        adapter = LangGraphAdapter()
        assert hasattr(adapter, "initialize"), "Missing initialize method"
        assert callable(adapter.initialize), "initialize is not callable"
        assert inspect.iscoroutinefunction(
            adapter.initialize
        ), "initialize must be async"

    def test_has_run_method(self):
        """Adapter must have async run() method."""
        from langgraph_runner.adapter import LangGraphAdapter

        adapter = LangGraphAdapter()
        assert hasattr(adapter, "run"), "Missing run method"
        assert callable(adapter.run), "run is not callable"
        assert inspect.iscoroutinefunction(adapter.run), "run must be async"

    def test_has_handle_message_method(self):
        """Adapter must have async handle_message(message: Dict) method."""
        from langgraph_runner.adapter import LangGraphAdapter

        adapter = LangGraphAdapter()
        assert hasattr(adapter, "handle_message"), "Missing handle_message method"
        assert callable(adapter.handle_message), "handle_message is not callable"
        assert inspect.iscoroutinefunction(
            adapter.handle_message
        ), "handle_message must be async"

    def test_initialize_signature(self):
        """Verify initialize(context: RunnerContext) signature."""
        from langgraph_runner.adapter import LangGraphAdapter

        sig = inspect.signature(LangGraphAdapter.initialize)
        params = list(sig.parameters.keys())

        assert "self" in params, "Missing self parameter"
        assert "context" in params, "Missing context parameter"

        # Check type annotation
        context_param = sig.parameters["context"]
        assert (
            context_param.annotation == RunnerContext
        ), f"context parameter should be RunnerContext, got {context_param.annotation}"

    def test_run_signature(self):
        """Verify run() signature and return type."""
        from langgraph_runner.adapter import LangGraphAdapter

        sig = inspect.signature(LangGraphAdapter.run)
        params = list(sig.parameters.keys())

        assert params == ["self"], f"run() should only have self parameter, got {params}"

        # Return type should be Dict
        return_annotation = str(sig.return_annotation)
        assert (
            "Dict" in return_annotation or "dict" in return_annotation
        ), f"run() should return Dict, got {sig.return_annotation}"

    def test_handle_message_signature(self):
        """Verify handle_message(message: Dict) signature."""
        from langgraph_runner.adapter import LangGraphAdapter

        sig = inspect.signature(LangGraphAdapter.handle_message)
        params = list(sig.parameters.keys())

        assert "self" in params, "Missing self parameter"
        assert "message" in params, "Missing message parameter"

        # Check type annotation
        message_param = sig.parameters["message"]
        assert (
            "Dict" in str(message_param.annotation)
            or "dict" in str(message_param.annotation)
        ), f"message parameter should be Dict, got {message_param.annotation}"

    @pytest.mark.asyncio
    async def test_initialize_stores_context(self, runner_context):
        """Verify initialize() stores context reference."""
        from langgraph_runner.adapter import LangGraphAdapter

        adapter = LangGraphAdapter()
        await adapter.initialize(runner_context)

        assert adapter.context is not None, "Context not stored"
        assert adapter.context.session_id == runner_context.session_id
        assert adapter.context.workspace_path == runner_context.workspace_path

    @pytest.mark.asyncio
    async def test_run_returns_dict_with_required_keys(
        self, runner_context, mock_runner_shell
    ):
        """Verify run() returns dict with success/result/error keys."""
        from langgraph_runner.adapter import LangGraphAdapter

        adapter = LangGraphAdapter()
        adapter.shell = mock_runner_shell
        await adapter.initialize(runner_context)

        result = await adapter.run()

        assert isinstance(result, dict), f"run() should return dict, got {type(result)}"
        assert "success" in result, "Result missing 'success' key"
        assert isinstance(result["success"], bool), "'success' should be boolean"

        # Error case should have 'error' key
        if not result["success"]:
            assert "error" in result, "Failed result missing 'error' key"

    @pytest.mark.asyncio
    async def test_handle_message_accepts_dict(self, runner_context, mock_runner_shell):
        """Verify handle_message accepts Dict parameter."""
        from langgraph_runner.adapter import LangGraphAdapter

        adapter = LangGraphAdapter()
        adapter.shell = mock_runner_shell
        await adapter.initialize(runner_context)

        message = {"type": "user_message", "content": "Test prompt"}

        # Should not raise exception
        await adapter.handle_message(message)

    def test_websoket_message_protocol_compliance(self):
        """Verify adapter uses correct WebSocket message types."""
        from langgraph_runner.adapter import LangGraphAdapter
        import ast

        # Read source code
        source = inspect.getsource(LangGraphAdapter)

        # Check for use of MessageType enum
        assert "MessageType" in source, "Should use MessageType enum for messages"

        # Check for expected message types
        expected_types = [
            "SYSTEM_MESSAGE",
            "AGENT_MESSAGE",
            "USER_MESSAGE",
            "AGENT_RUNNING",
        ]

        for msg_type in expected_types:
            assert (
                msg_type in source
            ), f"Should use MessageType.{msg_type} in adapter implementation"
```

---

## 4. Unit Testing Examples

### 4.1 tests/unit/test_adapter_methods.py

```python
"""Unit tests for LangGraphAdapter methods."""

import pytest
import json
from pathlib import Path
from unittest.mock import AsyncMock, Mock, patch


@pytest.mark.asyncio
class TestLangGraphAdapterInitialize:
    """Test LangGraphAdapter.initialize() method."""

    async def test_initialize_with_valid_context(self, runner_context, mock_runner_shell):
        """Test successful initialization with valid context."""
        from langgraph_runner.adapter import LangGraphAdapter

        adapter = LangGraphAdapter()
        adapter.shell = mock_runner_shell

        await adapter.initialize(runner_context)

        assert adapter.context == runner_context
        assert adapter.context.session_id == "test-session-123456"
        assert Path(adapter.context.workspace_path).exists()

    async def test_initialize_sets_working_directory(self, runner_context):
        """Test initialize changes working directory to workspace."""
        from langgraph_runner.adapter import LangGraphAdapter
        import os

        original_cwd = os.getcwd()
        adapter = LangGraphAdapter()

        await adapter.initialize(runner_context)

        assert os.getcwd() == runner_context.workspace_path

        # Cleanup
        os.chdir(original_cwd)

    async def test_initialize_clones_repositories(
        self, runner_context, mock_runner_shell, monkeypatch
    ):
        """Test initialize clones repositories from REPOS_JSON."""
        from langgraph_runner.adapter import LangGraphAdapter

        repos_json = json.dumps(
            [
                {
                    "name": "test-repo",
                    "input": {
                        "url": "https://github.com/test/repo",
                        "branch": "main",
                    },
                }
            ]
        )

        runner_context.environment["REPOS_JSON"] = repos_json

        # Mock git clone
        mock_run_cmd = AsyncMock()
        adapter = LangGraphAdapter()
        adapter.shell = mock_runner_shell
        adapter._run_cmd = mock_run_cmd

        await adapter.initialize(runner_context)

        # Verify git clone was called
        # (Actual implementation would verify this)

    async def test_initialize_handles_missing_repositories(
        self, runner_context, mock_runner_shell, monkeypatch
    ):
        """Test initialize handles gracefully when repo clone fails."""
        from langgraph_runner.adapter import LangGraphAdapter

        repos_json = json.dumps(
            [
                {
                    "name": "missing-repo",
                    "input": {
                        "url": "https://github.com/nonexistent/repo",
                        "branch": "main",
                    },
                }
            ]
        )

        runner_context.environment["REPOS_JSON"] = repos_json

        adapter = LangGraphAdapter()
        adapter.shell = mock_runner_shell

        # Mock failed git clone
        async def mock_run_cmd(*args, **kwargs):
            raise RuntimeError("Repository not found")

        adapter._run_cmd = mock_run_cmd

        with pytest.raises(RuntimeError, match="repository"):
            await adapter.initialize(runner_context)


@pytest.mark.asyncio
class TestLangGraphAdapterRun:
    """Test LangGraphAdapter.run() method."""

    async def test_run_with_simple_prompt(self, runner_context, mock_runner_shell):
        """Test run executes with simple prompt."""
        from langgraph_runner.adapter import LangGraphAdapter

        runner_context.environment["PROMPT"] = "Create a simple specification"

        adapter = LangGraphAdapter()
        adapter.shell = mock_runner_shell
        await adapter.initialize(runner_context)

        # Mock LangGraph execution
        with patch.object(adapter, "_run_langgraph_workflow", new_callable=AsyncMock) as mock_workflow:
            mock_workflow.return_value = {"output": "Sample specification"}

            result = await adapter.run()

            assert result["success"] is True
            mock_workflow.assert_called_once()

    async def test_run_returns_error_on_api_failure(self, runner_context, mock_runner_shell):
        """Test run returns error dict on API failure."""
        from langgraph_runner.adapter import LangGraphAdapter

        runner_context.environment["ANTHROPIC_API_KEY"] = "invalid-key"
        runner_context.environment["PROMPT"] = "Test prompt"

        adapter = LangGraphAdapter()
        adapter.shell = mock_runner_shell
        await adapter.initialize(runner_context)

        # Mock API failure
        with patch.object(
            adapter, "_run_langgraph_workflow", new_callable=AsyncMock
        ) as mock_workflow:
            mock_workflow.side_effect = RuntimeError("Authentication failed")

            result = await adapter.run()

            assert result["success"] is False
            assert "error" in result
            assert "authentication" in result["error"].lower()

    async def test_run_sends_progress_messages(self, runner_context, mock_runner_shell):
        """Test run sends progress messages to WebSocket."""
        from langgraph_runner.adapter import LangGraphAdapter
        from runner_shell.core.protocol import MessageType

        runner_context.environment["PROMPT"] = "Test prompt"

        adapter = LangGraphAdapter()
        adapter.shell = mock_runner_shell
        await adapter.initialize(runner_context)

        with patch.object(adapter, "_run_langgraph_workflow", new_callable=AsyncMock):
            await adapter.run()

            # Verify messages sent to WebSocket
            calls = mock_runner_shell._send_message.call_args_list

            # Should have at least one message call
            assert len(calls) > 0

            # Check message types
            message_types = [call[0][0] for call in calls]
            assert MessageType.SYSTEM_MESSAGE in message_types


@pytest.mark.asyncio
class TestLangGraphAdapterMessageHandling:
    """Test LangGraphAdapter.handle_message() method."""

    async def test_handle_user_message(self, runner_context, mock_runner_shell):
        """Test handling user_message in interactive mode."""
        from langgraph_runner.adapter import LangGraphAdapter

        adapter = LangGraphAdapter()
        adapter.shell = mock_runner_shell
        await adapter.initialize(runner_context)

        message = {
            "type": "user_message",
            "content": "Continue with the implementation"
        }

        # Should not raise
        await adapter.handle_message(message)

        # Message should be queued for processing
        # (Implementation specific)

    async def test_handle_interrupt_message(self, runner_context, mock_runner_shell):
        """Test handling interrupt message."""
        from langgraph_runner.adapter import LangGraphAdapter

        adapter = LangGraphAdapter()
        adapter.shell = mock_runner_shell
        await adapter.initialize(runner_context)

        message = {"type": "interrupt"}

        await adapter.handle_message(message)

        # Interrupt should be processed
        # (Implementation specific)

    async def test_handle_end_session_message(self, runner_context, mock_runner_shell):
        """Test handling end_session message."""
        from langgraph_runner.adapter import LangGraphAdapter

        adapter = LangGraphAdapter()
        adapter.shell = mock_runner_shell
        await adapter.initialize(runner_context)

        message = {"type": "end_session"}

        await adapter.handle_message(message)

        # Session should be terminated
        # (Implementation specific)

    async def test_handle_unknown_message_type(self, runner_context, mock_runner_shell):
        """Test handling unknown message type gracefully."""
        from langgraph_runner.adapter import LangGraphAdapter

        adapter = LangGraphAdapter()
        adapter.shell = mock_runner_shell
        await adapter.initialize(runner_context)

        message = {"type": "unknown_type", "data": "test"}

        # Should handle gracefully without raising
        await adapter.handle_message(message)
```

---

## 5. Integration Testing Examples

### 5.1 tests/integration/test_operator_integration.py

```python
"""Integration tests for operator and runner selection."""

import pytest
import yaml
import subprocess
from pathlib import Path


@pytest.mark.integration
@pytest.mark.k8s
class TestOperatorRunnerSelection:
    """Test operator correctly selects runner image based on runnerType."""

    @pytest.fixture
    def k8s_namespace(self):
        """Create temporary Kubernetes namespace for testing."""
        import uuid
        ns_name = f"test-ns-{str(uuid.uuid4())[:8]}"

        # Create namespace
        subprocess.run(
            ["kubectl", "create", "namespace", ns_name],
            check=True,
            capture_output=True
        )

        yield ns_name

        # Cleanup
        subprocess.run(
            ["kubectl", "delete", "namespace", ns_name],
            capture_output=True
        )

    def test_operator_selects_langgraph_image(self, k8s_namespace):
        """Verify operator uses AMBIENT_LANGGRAPH_RUNNER_IMAGE for langgraph runnerType."""
        cr_yaml = {
            "apiVersion": "ambient-code.io/v1",
            "kind": "AgenticSession",
            "metadata": {
                "name": "test-langgraph",
                "namespace": k8s_namespace,
            },
            "spec": {
                "prompt": "Create a test specification",
                "runnerType": "langgraph",
                "timeout": 300,
                "llmSettings": {
                    "model": "claude-3-5-sonnet-20241022",
                    "temperature": 0.7,
                },
                "repos": [
                    {
                        "input": {
                            "url": "https://github.com/test/repo",
                            "branch": "main",
                        }
                    }
                ],
            },
        }

        # Apply CR
        cr_yaml_str = yaml.dump(cr_yaml)
        result = subprocess.run(
            ["kubectl", "apply", "-f", "-"],
            input=cr_yaml_str.encode(),
            capture_output=True,
            text=True,
            cwd=k8s_namespace,
        )
        assert result.returncode == 0, f"Failed to apply CR: {result.stderr}"

        # Wait for Job creation (with timeout)
        import time
        max_retries = 30
        for i in range(max_retries):
            result = subprocess.run(
                [
                    "kubectl",
                    "get",
                    "job",
                    "-n",
                    k8s_namespace,
                    "-o",
                    "yaml",
                    "-l",
                    "session=test-langgraph",
                ],
                capture_output=True,
                text=True,
            )

            if result.returncode == 0 and result.stdout.strip():
                job = yaml.safe_load(result.stdout)
                if job.get("items"):
                    # Verify image selection
                    job_spec = job["items"][0]["spec"]["template"]["spec"]
                    image = job_spec["containers"][0]["image"]

                    # Should use LangGraph image (value depends on environment setup)
                    assert "langgraph-runner" in image.lower() or "langgraph" in image.lower(), \
                        f"Expected LangGraph image, got: {image}"

                    return

            time.sleep(1)

        pytest.fail("Job was not created within timeout")

    def test_operator_selects_claude_code_image(self, k8s_namespace):
        """Verify operator uses AMBIENT_CLAUDE_RUNNER_IMAGE for claude-code runnerType."""
        cr_yaml = {
            "apiVersion": "ambient-code.io/v1",
            "kind": "AgenticSession",
            "metadata": {
                "name": "test-claude-code",
                "namespace": k8s_namespace,
            },
            "spec": {
                "prompt": "Create a test specification",
                "runnerType": "claude-code",
                "timeout": 300,
                "llmSettings": {
                    "model": "claude-3-5-sonnet-20241022",
                    "temperature": 0.7,
                },
            },
        }

        # Apply CR and verify image
        cr_yaml_str = yaml.dump(cr_yaml)
        subprocess.run(
            ["kubectl", "apply", "-f", "-"],
            input=cr_yaml_str.encode(),
            capture_output=True,
            check=True,
        )

        # Wait for Job and verify image is Claude Code
        # (Similar pattern to above)

    def test_operator_defaults_to_claude_code(self, k8s_namespace):
        """Verify operator defaults to Claude Code when runnerType not specified."""
        cr_yaml = {
            "apiVersion": "ambient-code.io/v1",
            "kind": "AgenticSession",
            "metadata": {
                "name": "test-default",
                "namespace": k8s_namespace,
            },
            "spec": {
                "prompt": "Create a test specification",
                # runnerType NOT specified
                "timeout": 300,
            },
        }

        # Apply CR and verify default image is Claude Code
        # (Similar pattern to above)
```

---

## 6. Running Tests in CI/CD

### 6.1 GitHub Actions Workflow Snippet

```yaml
# .github/workflows/langgraph-tests.yml

name: LangGraph Runner Tests

on:
  push:
    branches: [main, ambient-langgraph-runner]
    paths:
      - 'components/runners/langgraph-runner/**'
      - 'tests/**'

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.11']

    steps:
      - uses: actions/checkout@v3

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v4
        with:
          python-version: ${{ matrix.python-version }}

      - name: Cache pip packages
        uses: actions/cache@v3
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements*.txt') }}

      - name: Install dependencies
        working-directory: components/runners/langgraph-runner
        run: |
          pip install -r requirements.txt
          pip install -r requirements-test.txt

      - name: Lint with Black
        working-directory: components/runners/langgraph-runner
        run: black --check langgraph_runner tests

      - name: Type check with mypy
        working-directory: components/runners/langgraph-runner
        run: mypy langgraph_runner

      - name: Run unit tests
        working-directory: components/runners/langgraph-runner
        run: |
          pytest tests/unit/ tests/contract/ -v --cov=langgraph_runner \
            --cov-report=xml --cov-report=term

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          files: ./components/runners/langgraph-runner/coverage.xml

      - name: Generate HTML coverage report
        if: always()
        working-directory: components/runners/langgraph-runner
        run: coverage html

      - name: Archive coverage report
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: coverage-report
          path: components/runners/langgraph-runner/htmlcov/
```

---

## 7. Quick Command Reference

```bash
# Run all tests
pytest tests/ -v

# Run only unit tests
pytest tests/unit/ -v

# Run only contract tests
pytest tests/contract/ -v

# Run with coverage
pytest tests/ --cov=langgraph_runner --cov-report=html

# Run specific test class
pytest tests/unit/test_adapter_methods.py::TestLangGraphAdapterRun -v

# Run specific test method
pytest tests/unit/test_adapter_methods.py::TestLangGraphAdapterRun::test_run_with_simple_prompt -v

# Run with markers
pytest tests/ -m "asyncio and not integration" -v

# Run with detailed output
pytest tests/ -vv --tb=long --capture=no

# Run with strict markers (fail if marker not found)
pytest tests/ -m "unit" --strict-markers -v

# Run with custom timeout
pytest tests/ --timeout=60 -v

# Parallel execution
pytest tests/ -n auto

# Only failed tests from last run
pytest tests/ --lf -v

# Profile test execution
pytest tests/ --durations=10

# Debug first failure and stop
pytest tests/ -x -vv
```

---

**Next Steps:**
1. Set up local test environment using this guide
2. Implement LangGraphAdapter according to contract specifications
3. Run contract tests to verify interface compliance
4. Add unit tests for each adapter method
5. Set up Kind cluster for integration testing
6. Verify CI/CD pipeline passes all test suites

