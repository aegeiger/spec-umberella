# Implementation Patterns: LangGraph Runner Integration

**Author:** Stella (Staff Engineer)
**Date:** 2025-11-04
**Purpose:** Concrete code patterns and best practices for implementing LangGraph runner

This document complements the technical architecture with hands-on implementation guidance, focusing on patterns that have proven effective in similar ML/AI systems.

---

## Pattern 1: Graph Loading & Validation

### Problem
Graph definitions are executable Python code. We need to load them safely and validate structure before execution.

### Implementation Pattern

```python
# /components/runners/langgraph-runner/graph_loader.py

import ast
import inspect
from pathlib import Path
from typing import Optional, Set
from langgraph.graph import StateGraph


class GraphLoader:
    """Secure loader for LangGraph definitions with validation."""

    # Allowed imports for graph definitions (whitelist approach)
    ALLOWED_MODULES = {
        'langgraph.graph',
        'langgraph.prebuilt',
        'langchain_core.messages',
        'langchain_anthropic',
        'typing',
        'pydantic',
    }

    def __init__(self, workspace_path: Path):
        self.workspace_path = workspace_path

    def load_graph(self, graph_file: Path) -> StateGraph:
        """
        Load and validate graph definition from Python file.

        Security checks:
        - Validates Python syntax
        - Checks for dangerous operations (exec, eval, __import__)
        - Ensures only whitelisted imports
        - Validates graph structure (has nodes, edges)
        """
        if not graph_file.exists():
            raise FileNotFoundError(f"Graph definition not found: {graph_file}")

        # 1. Parse AST and validate structure
        with open(graph_file, 'r') as f:
            source = f.read()

        try:
            tree = ast.parse(source)
        except SyntaxError as e:
            raise ValueError(f"Invalid Python syntax in graph definition: {e}")

        # 2. Security validation
        self._validate_ast_security(tree, graph_file)

        # 3. Execute in restricted namespace
        namespace = self._create_safe_namespace()
        exec(compile(tree, graph_file, 'exec'), namespace)

        # 4. Extract graph from namespace
        if 'graph' not in namespace:
            raise ValueError(
                "Graph definition must define a 'graph' variable of type StateGraph"
            )

        graph = namespace['graph']
        if not isinstance(graph, StateGraph):
            raise TypeError(
                f"'graph' variable must be StateGraph, got {type(graph)}"
            )

        # 5. Validate graph structure
        self._validate_graph_structure(graph)

        return graph

    def _validate_ast_security(self, tree: ast.AST, filename: Path):
        """Check AST for dangerous operations."""
        dangerous_nodes = []

        for node in ast.walk(tree):
            # Block eval/exec
            if isinstance(node, ast.Call):
                if isinstance(node.func, ast.Name):
                    if node.func.id in ('eval', 'exec', 'compile', '__import__'):
                        dangerous_nodes.append(
                            f"Dangerous function call: {node.func.id} at line {node.lineno}"
                        )

            # Block import of disallowed modules
            elif isinstance(node, ast.Import):
                for alias in node.names:
                    if not self._is_allowed_module(alias.name):
                        dangerous_nodes.append(
                            f"Disallowed import: {alias.name} at line {node.lineno}"
                        )

            elif isinstance(node, ast.ImportFrom):
                if node.module and not self._is_allowed_module(node.module):
                    dangerous_nodes.append(
                        f"Disallowed import from: {node.module} at line {node.lineno}"
                    )

        if dangerous_nodes:
            raise SecurityError(
                f"Graph definition contains dangerous operations:\n" +
                "\n".join(dangerous_nodes)
            )

    def _is_allowed_module(self, module_name: str) -> bool:
        """Check if module is in whitelist."""
        # Allow exact matches or submodules
        return any(
            module_name == allowed or module_name.startswith(allowed + '.')
            for allowed in self.ALLOWED_MODULES
        )

    def _create_safe_namespace(self) -> dict:
        """Create restricted execution namespace."""
        # Start with minimal builtins
        safe_builtins = {
            'print': print,
            'len': len,
            'str': str,
            'int': int,
            'float': float,
            'bool': bool,
            'list': list,
            'dict': dict,
            'tuple': tuple,
            'set': set,
            'range': range,
            'enumerate': enumerate,
            'zip': zip,
        }

        return {
            '__builtins__': safe_builtins,
            '__name__': '__main__',
            '__file__': str(self.workspace_path / 'graph.py'),
        }

    def _validate_graph_structure(self, graph: StateGraph):
        """Validate graph has required structure."""
        if not hasattr(graph, 'nodes') or not graph.nodes:
            raise ValueError("Graph must have at least one node")

        if not hasattr(graph, 'edges') or not graph.edges:
            raise ValueError("Graph must have at least one edge")

        # Check for entry point
        if not hasattr(graph, '_entry_point') or graph._entry_point is None:
            raise ValueError("Graph must have an entry point (use graph.set_entry_point())")


class SecurityError(Exception):
    """Raised when graph definition contains security issues."""
    pass
```

**Usage in adapter:**
```python
from graph_loader import GraphLoader

async def _load_graph(self, graph_path: Path) -> StateGraph:
    loader = GraphLoader(self.context.workspace_path)
    return loader.load_graph(graph_path)
```

---

## Pattern 2: Checkpoint Management with Pruning

### Problem
Checkpoints accumulate over time, filling PVC. Need automatic pruning while preserving recent state.

### Implementation Pattern

```python
# /components/runners/langgraph-runner/checkpointer.py

import asyncio
import logging
from pathlib import Path
from datetime import datetime, timedelta
from typing import Optional
from langgraph.checkpoint.sqlite import AsyncSqliteSaver
import aiosqlite

logger = logging.getLogger(__name__)


class ManagedCheckpointer:
    """
    Wrapper around LangGraph checkpointer with automatic pruning.

    Features:
    - Prunes old checkpoints to save disk space
    - Keeps last N checkpoints per thread
    - Async checkpoint operations for better performance
    """

    def __init__(
        self,
        db_path: Path,
        max_checkpoints_per_thread: int = 10,
        prune_interval_hours: int = 24,
    ):
        self.db_path = db_path
        self.max_checkpoints = max_checkpoints_per_thread
        self.prune_interval = timedelta(hours=prune_interval_hours)
        self.checkpointer: Optional[AsyncSqliteSaver] = None
        self._prune_task: Optional[asyncio.Task] = None

    async def initialize(self) -> AsyncSqliteSaver:
        """Initialize checkpointer with pruning task."""
        self.db_path.parent.mkdir(parents=True, exist_ok=True)

        # Create checkpointer with WAL mode for better concurrency
        self.checkpointer = await AsyncSqliteSaver.from_conn_string(
            f"sqlite:///{self.db_path}?journal_mode=WAL"
        )

        # Start background pruning task
        self._prune_task = asyncio.create_task(self._prune_loop())

        return self.checkpointer

    async def close(self):
        """Clean up resources."""
        if self._prune_task:
            self._prune_task.cancel()
            try:
                await self._prune_task
            except asyncio.CancelledError:
                pass

    async def _prune_loop(self):
        """Background task to periodically prune old checkpoints."""
        while True:
            try:
                await asyncio.sleep(self.prune_interval.total_seconds())
                await self.prune_old_checkpoints()
            except asyncio.CancelledError:
                break
            except Exception as e:
                logger.error(f"Checkpoint pruning failed: {e}")

    async def prune_old_checkpoints(self):
        """
        Remove old checkpoints, keeping only the most recent N per thread.

        This prevents unbounded growth of checkpoint database.
        """
        logger.info(f"Starting checkpoint pruning (keep {self.max_checkpoints} per thread)")

        async with aiosqlite.connect(self.db_path) as db:
            # Get all threads
            async with db.execute(
                "SELECT DISTINCT thread_id FROM checkpoints"
            ) as cursor:
                threads = [row[0] async for row in cursor]

            total_deleted = 0

            for thread_id in threads:
                # Get checkpoints for this thread, ordered by timestamp (newest first)
                async with db.execute(
                    """
                    SELECT checkpoint_id, timestamp
                    FROM checkpoints
                    WHERE thread_id = ?
                    ORDER BY timestamp DESC
                    """,
                    (thread_id,)
                ) as cursor:
                    checkpoints = [(row[0], row[1]) async for row in cursor]

                # Keep only the most recent N checkpoints
                if len(checkpoints) > self.max_checkpoints:
                    to_delete = checkpoints[self.max_checkpoints:]
                    delete_ids = [cp[0] for cp in to_delete]

                    await db.execute(
                        f"""
                        DELETE FROM checkpoints
                        WHERE checkpoint_id IN ({','.join('?' * len(delete_ids))})
                        """,
                        delete_ids
                    )

                    deleted_count = len(delete_ids)
                    total_deleted += deleted_count
                    logger.info(
                        f"Pruned {deleted_count} old checkpoints for thread {thread_id}"
                    )

            await db.commit()

        logger.info(f"Checkpoint pruning complete: removed {total_deleted} checkpoints")

    async def get_checkpoint_stats(self) -> dict:
        """Get statistics about checkpoint storage."""
        async with aiosqlite.connect(self.db_path) as db:
            # Count checkpoints per thread
            async with db.execute(
                """
                SELECT thread_id, COUNT(*) as count, MAX(timestamp) as latest
                FROM checkpoints
                GROUP BY thread_id
                """
            ) as cursor:
                threads = {}
                async for row in cursor:
                    threads[row[0]] = {
                        "count": row[1],
                        "latest_timestamp": row[2],
                    }

            # Get database file size
            db_size_mb = self.db_path.stat().st_size / (1024 * 1024)

            return {
                "threads": threads,
                "total_threads": len(threads),
                "db_size_mb": round(db_size_mb, 2),
            }
```

**Usage in adapter:**
```python
async def _setup_checkpointer(self) -> AsyncSqliteSaver:
    checkpoint_dir = Path(self.context.workspace_path).parent / ".langgraph"
    db_path = checkpoint_dir / "checkpoints.db"

    # Create managed checkpointer with auto-pruning
    self.managed_checkpointer = ManagedCheckpointer(
        db_path=db_path,
        max_checkpoints_per_thread=10,  # Keep last 10 states
        prune_interval_hours=24,        # Prune daily
    )

    return await self.managed_checkpointer.initialize()
```

---

## Pattern 3: Human-in-the-Loop with Timeout

### Problem
Graph may wait indefinitely for user input. Need timeout and fallback behavior.

### Implementation Pattern

```python
# /components/runners/langgraph-runner/hitl_manager.py

import asyncio
from typing import Optional, Any
from dataclasses import dataclass
from datetime import datetime


@dataclass
class PendingInput:
    """Represents a pending user input request."""
    node_name: str
    prompt: str
    created_at: datetime
    timeout_seconds: int = 3600  # 1 hour default


class HumanInTheLoopManager:
    """
    Manages human-in-the-loop interactions with timeout handling.

    Pattern:
    1. Graph hits interrupt node
    2. Manager sends WAITING_FOR_INPUT message
    3. Wait for user input with timeout
    4. Resume graph or abort if timeout
    """

    def __init__(self, shell):
        self.shell = shell
        self.pending_input: Optional[PendingInput] = None
        self.input_queue: asyncio.Queue = asyncio.Queue()

    async def request_input(
        self,
        node_name: str,
        prompt: str,
        timeout_seconds: int = 3600,
    ) -> Optional[str]:
        """
        Request user input with timeout.

        Returns:
            User input string if received, None if timeout
        """
        # Record pending request
        self.pending_input = PendingInput(
            node_name=node_name,
            prompt=prompt,
            created_at=datetime.utcnow(),
            timeout_seconds=timeout_seconds,
        )

        # Send waiting message to UI
        await self.shell._send_message(
            MessageType.WAITING_FOR_INPUT,
            {
                "node": node_name,
                "prompt": prompt,
                "timeout_seconds": timeout_seconds,
            }
        )

        try:
            # Wait for input with timeout
            user_input = await asyncio.wait_for(
                self.input_queue.get(),
                timeout=timeout_seconds
            )

            self.pending_input = None
            return user_input

        except asyncio.TimeoutError:
            # Timeout - send timeout notification
            await self.shell._send_message(
                MessageType.SYSTEM_MESSAGE,
                {
                    "message": f"Input timeout after {timeout_seconds}s - continuing with default",
                    "level": "warning",
                }
            )

            self.pending_input = None
            return None

    def submit_input(self, user_input: str):
        """Submit user input (called from handle_message)."""
        self.input_queue.put_nowait(user_input)

    def has_pending_input(self) -> bool:
        """Check if there's a pending input request."""
        return self.pending_input is not None

    def get_pending_prompt(self) -> Optional[str]:
        """Get the current pending prompt."""
        if self.pending_input:
            return self.pending_input.prompt
        return None
```

**Integration in adapter:**
```python
class LangGraphAdapter:
    def __init__(self):
        self.hitl_manager = None

    async def initialize(self, context: RunnerContext):
        # ... existing init
        self.hitl_manager = HumanInTheLoopManager(self.shell)

    async def _handle_graph_event(self, event: dict):
        if event["event"] == "on_interrupt":
            # Graph hit interrupt node
            node_name = event.get("name", "unknown")
            interrupt_data = event.get("data", {})

            # Request user input with timeout
            user_input = await self.hitl_manager.request_input(
                node_name=node_name,
                prompt=interrupt_data.get("prompt", "Please provide input"),
                timeout_seconds=3600,
            )

            if user_input is None:
                # Timeout - use default or abort
                raise InterruptTimeout(f"No input received for {node_name}")

            # Resume graph with user input
            # (LangGraph handles this via checkpoint update)

    async def handle_message(self, message: dict):
        msg_type = message.get('type', '')

        if msg_type == 'user_message':
            payload = message.get('payload', {})
            content = payload.get('content', '')

            # Submit to HITL manager if waiting for input
            if self.hitl_manager.has_pending_input():
                self.hitl_manager.submit_input(content)
```

---

## Pattern 4: Streaming Output with Chunking

### Problem
Large graph outputs can overwhelm WebSocket. Need to chunk and stream efficiently.

### Implementation Pattern

```python
# /components/runners/langgraph-runner/streaming.py

from typing import AsyncIterator, Any
from runner_shell.core.protocol import MessageType, PartialInfo
import json


class StreamingOutputHandler:
    """
    Handles streaming of large graph outputs to WebSocket.

    Features:
    - Chunks large outputs to avoid WebSocket limits
    - Tracks sequence numbers for reassembly
    - Handles backpressure
    """

    MAX_CHUNK_SIZE = 8192  # 8KB chunks

    def __init__(self, shell):
        self.shell = shell
        self.stream_seq = 0

    async def stream_output(
        self,
        output: Any,
        output_type: str = "node_output"
    ):
        """
        Stream output to WebSocket, chunking if necessary.

        Args:
            output: Output data (can be dict, list, string)
            output_type: Type of output (for UI rendering)
        """
        # Serialize output
        if isinstance(output, (dict, list)):
            output_str = json.dumps(output, indent=2)
        else:
            output_str = str(output)

        # Check if chunking needed
        if len(output_str) <= self.MAX_CHUNK_SIZE:
            # Small output - send as single message
            await self.shell._send_message(
                MessageType.AGENT_MESSAGE,
                {
                    "type": output_type,
                    "output": output_str,
                }
            )
        else:
            # Large output - chunk it
            await self._stream_chunked(output_str, output_type)

    async def _stream_chunked(self, content: str, output_type: str):
        """Stream content in chunks using MESSAGE_PARTIAL."""
        self.stream_seq += 1
        stream_id = f"stream-{self.stream_seq}"

        chunks = [
            content[i:i + self.MAX_CHUNK_SIZE]
            for i in range(0, len(content), self.MAX_CHUNK_SIZE)
        ]

        total_chunks = len(chunks)

        for index, chunk in enumerate(chunks):
            partial = PartialInfo(
                id=stream_id,
                index=index,
                total=total_chunks,
                data=chunk,
            )

            await self.shell._send_message(
                MessageType.MESSAGE_PARTIAL,
                {
                    "type": output_type,
                    "stream_id": stream_id,
                },
                partial=partial,
            )

            # Small delay to avoid overwhelming WebSocket
            if index < total_chunks - 1:
                await asyncio.sleep(0.01)
```

**Usage in adapter:**
```python
class LangGraphAdapter:
    def __init__(self):
        self.streaming_handler = None

    async def initialize(self, context: RunnerContext):
        # ... existing init
        self.streaming_handler = StreamingOutputHandler(self.shell)

    async def _handle_graph_event(self, event: dict):
        if event["event"] == "on_chain_end":
            output = event.get("data", {}).get("output", {})

            # Stream output (automatically chunks if large)
            await self.streaming_handler.stream_output(
                output,
                output_type="node_output"
            )
```

---

## Pattern 5: Observability & Metrics

### Problem
Need visibility into graph execution for debugging and performance monitoring.

### Implementation Pattern

```python
# /components/runners/langgraph-runner/observability.py

import time
from typing import Dict, List, Optional
from dataclasses import dataclass, field
from datetime import datetime


@dataclass
class NodeExecution:
    """Records a single node execution."""
    node_name: str
    start_time: float
    end_time: Optional[float] = None
    success: bool = True
    error: Optional[str] = None
    output_size_bytes: int = 0

    @property
    def duration_ms(self) -> Optional[float]:
        if self.end_time:
            return (self.end_time - self.start_time) * 1000
        return None


@dataclass
class GraphExecutionMetrics:
    """Aggregates metrics for entire graph execution."""
    thread_id: str
    start_time: datetime
    end_time: Optional[datetime] = None
    nodes_executed: List[NodeExecution] = field(default_factory=list)
    interrupts: int = 0
    checkpoints_saved: int = 0
    total_output_bytes: int = 0

    def add_node_execution(self, node: NodeExecution):
        self.nodes_executed.append(node)
        self.total_output_bytes += node.output_size_bytes

    def to_dict(self) -> dict:
        """Convert to dict for logging/reporting."""
        return {
            "thread_id": self.thread_id,
            "start_time": self.start_time.isoformat(),
            "end_time": self.end_time.isoformat() if self.end_time else None,
            "total_nodes": len(self.nodes_executed),
            "successful_nodes": sum(1 for n in self.nodes_executed if n.success),
            "failed_nodes": sum(1 for n in self.nodes_executed if not n.success),
            "total_duration_ms": sum(
                n.duration_ms for n in self.nodes_executed if n.duration_ms
            ),
            "interrupts": self.interrupts,
            "checkpoints_saved": self.checkpoints_saved,
            "total_output_kb": round(self.total_output_bytes / 1024, 2),
        }


class ObservabilityManager:
    """Tracks and reports graph execution metrics."""

    def __init__(self, shell):
        self.shell = shell
        self.metrics: Optional[GraphExecutionMetrics] = None
        self.current_node: Optional[NodeExecution] = None

    def start_execution(self, thread_id: str):
        """Start tracking a new execution."""
        self.metrics = GraphExecutionMetrics(
            thread_id=thread_id,
            start_time=datetime.utcnow(),
        )

    def start_node(self, node_name: str):
        """Record start of node execution."""
        self.current_node = NodeExecution(
            node_name=node_name,
            start_time=time.time(),
        )

    def end_node(self, output: Any = None, error: Optional[str] = None):
        """Record end of node execution."""
        if not self.current_node:
            return

        self.current_node.end_time = time.time()

        if error:
            self.current_node.success = False
            self.current_node.error = error
        elif output is not None:
            # Estimate output size
            output_str = str(output)
            self.current_node.output_size_bytes = len(output_str.encode('utf-8'))

        self.metrics.add_node_execution(self.current_node)
        self.current_node = None

    def record_interrupt(self):
        """Record human-in-the-loop interrupt."""
        if self.metrics:
            self.metrics.interrupts += 1

    def record_checkpoint(self):
        """Record checkpoint save."""
        if self.metrics:
            self.metrics.checkpoints_saved += 1

    async def end_execution(self):
        """Finalize metrics and report to backend."""
        if not self.metrics:
            return

        self.metrics.end_time = datetime.utcnow()

        # Send metrics as system message
        await self.shell._send_message(
            MessageType.SYSTEM_MESSAGE,
            {
                "type": "execution_metrics",
                "metrics": self.metrics.to_dict(),
            }
        )
```

**Integration in adapter:**
```python
class LangGraphAdapter:
    def __init__(self):
        self.observability = None

    async def initialize(self, context: RunnerContext):
        # ... existing init
        self.observability = ObservabilityManager(self.shell)

    async def run(self) -> Dict[str, Any]:
        self.observability.start_execution(self.thread_id)

        try:
            # ... graph execution
            async for event in self.graph.astream_events(...):
                await self._handle_graph_event(event)

            return {"success": True}

        finally:
            await self.observability.end_execution()

    async def _handle_graph_event(self, event: dict):
        event_type = event.get("event")

        if event_type == "on_chain_start":
            node_name = event.get("name", "")
            self.observability.start_node(node_name)

        elif event_type == "on_chain_end":
            output = event.get("data", {}).get("output")
            self.observability.end_node(output=output)

        elif event_type == "on_chain_error":
            error = str(event.get("data", {}).get("error", "Unknown error"))
            self.observability.end_node(error=error)

        elif event_type == "on_interrupt":
            self.observability.record_interrupt()
```

---

## Pattern 6: Error Recovery & Retries

### Problem
Node failures should be recoverable without restarting entire graph.

### Implementation Pattern

```python
# /components/runners/langgraph-runner/error_handling.py

import asyncio
from typing import Callable, Any, Optional
from functools import wraps


class RetryableNodeError(Exception):
    """Raised when a node execution fails but can be retried."""
    pass


class FatalNodeError(Exception):
    """Raised when a node execution fails and cannot be retried."""
    pass


def with_retry(
    max_attempts: int = 3,
    backoff_base: float = 2.0,
    exceptions: tuple = (Exception,)
):
    """
    Decorator for node functions to add automatic retry logic.

    Usage:
        @with_retry(max_attempts=3)
        async def my_node(state):
            # node implementation
    """
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        async def wrapper(*args, **kwargs):
            last_exception = None

            for attempt in range(max_attempts):
                try:
                    return await func(*args, **kwargs)

                except FatalNodeError:
                    # Don't retry fatal errors
                    raise

                except exceptions as e:
                    last_exception = e

                    if attempt < max_attempts - 1:
                        # Calculate backoff
                        delay = backoff_base ** attempt
                        logger.warning(
                            f"Node {func.__name__} failed (attempt {attempt + 1}/{max_attempts}), "
                            f"retrying in {delay}s: {e}"
                        )
                        await asyncio.sleep(delay)
                    else:
                        logger.error(
                            f"Node {func.__name__} failed after {max_attempts} attempts: {e}"
                        )

            # All retries exhausted
            raise RetryableNodeError(
                f"Node failed after {max_attempts} attempts: {last_exception}"
            ) from last_exception

        return wrapper
    return decorator


# Example usage in graph definition:

from langgraph.graph import StateGraph
from typing import TypedDict

class AgentState(TypedDict):
    messages: list
    retry_count: int


@with_retry(max_attempts=3)
async def api_call_node(state: AgentState) -> AgentState:
    """Node that makes external API call with automatic retry."""
    # Simulate API call that might fail
    response = await external_api.call(state["messages"])

    return {
        **state,
        "messages": state["messages"] + [response]
    }


def create_graph() -> StateGraph:
    graph = StateGraph(AgentState)

    # Add nodes with built-in retry logic
    graph.add_node("api_call", api_call_node)
    graph.add_node("process_response", process_node)

    graph.set_entry_point("api_call")
    graph.add_edge("api_call", "process_response")

    return graph


graph = create_graph()
```

---

## Pattern 7: Configuration Management

### Problem
Graph behavior needs to be configurable without changing code.

### Implementation Pattern

```python
# /components/runners/langgraph-runner/config_manager.py

from pathlib import Path
from typing import Dict, Any, Optional
import yaml


class GraphConfig:
    """
    Configuration for graph execution.

    Loaded from .langgraph-config.yaml in workspace.
    """

    def __init__(self, workspace_path: Path):
        self.workspace_path = workspace_path
        self.config: Dict[str, Any] = {}
        self._load_config()

    def _load_config(self):
        """Load configuration from workspace."""
        config_file = self.workspace_path / ".langgraph-config.yaml"

        if config_file.exists():
            with open(config_file, 'r') as f:
                self.config = yaml.safe_load(f) or {}
        else:
            # Use defaults
            self.config = self._default_config()

    def _default_config(self) -> dict:
        """Default configuration."""
        return {
            "checkpointer": {
                "type": "sqlite",
                "max_checkpoints_per_thread": 10,
                "prune_interval_hours": 24,
            },
            "execution": {
                "recursion_limit": 100,
                "timeout_seconds": 3600,
                "retry_attempts": 3,
            },
            "human_in_the_loop": {
                "enabled": True,
                "input_timeout_seconds": 3600,
            },
            "observability": {
                "track_metrics": True,
                "log_node_outputs": False,
            }
        }

    def get(self, key_path: str, default: Any = None) -> Any:
        """
        Get config value by dot-separated path.

        Example:
            config.get("checkpointer.type")  # Returns "sqlite"
        """
        keys = key_path.split('.')
        value = self.config

        for key in keys:
            if isinstance(value, dict) and key in value:
                value = value[key]
            else:
                return default

        return value


# Example usage in adapter:

class LangGraphAdapter:
    async def initialize(self, context: RunnerContext):
        # Load configuration
        self.config = GraphConfig(Path(context.workspace_path))

        # Use configuration
        checkpointer_type = self.config.get("checkpointer.type", "sqlite")
        recursion_limit = self.config.get("execution.recursion_limit", 100)
        hitl_enabled = self.config.get("human_in_the_loop.enabled", True)

        # Configure components based on config
        if checkpointer_type == "sqlite":
            self.checkpointer = await self._setup_sqlite_checkpointer()
        elif checkpointer_type == "postgres":
            self.checkpointer = await self._setup_postgres_checkpointer()
```

**Example workspace config:**
```yaml
# .langgraph-config.yaml (in workspace root)

checkpointer:
  type: sqlite
  max_checkpoints_per_thread: 5
  prune_interval_hours: 12

execution:
  recursion_limit: 50
  timeout_seconds: 7200  # 2 hours
  retry_attempts: 2

human_in_the_loop:
  enabled: true
  input_timeout_seconds: 1800  # 30 minutes

observability:
  track_metrics: true
  log_node_outputs: true  # Debug mode
```

---

## Summary: Key Takeaways

These patterns address common challenges in LangGraph integration:

1. **Security First**: Graph loading with AST validation prevents code injection
2. **Resource Management**: Automatic checkpoint pruning prevents PVC exhaustion
3. **User Experience**: Timeout handling prevents indefinite waits
4. **Performance**: Streaming with chunking handles large outputs gracefully
5. **Operations**: Metrics and observability enable debugging in production
6. **Reliability**: Retry logic makes graphs resilient to transient failures
7. **Flexibility**: Configuration management allows tuning without code changes

**Implementation priority:**
1. Start with Pattern 1 (Graph Loading) - security critical
2. Add Pattern 2 (Checkpointer) - prevents operational issues
3. Layer in Pattern 3 (HITL) - core functionality
4. Enhance with Patterns 4-7 as needed

**Testing each pattern:**
- Unit tests for individual components
- Integration tests with mock graphs
- Load tests for performance characteristics
- Security tests for graph validation

**Next steps:**
1. Review patterns with security team
2. Prototype Pattern 1 + 2 in sandbox
3. Benchmark checkpoint performance
4. Document configuration schema
