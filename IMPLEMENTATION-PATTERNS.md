# Implementation Patterns: LangGraph Runner for RFE Workflow

**Author:** Stella (Staff Engineer)
**Date:** 2025-11-04
**Purpose:** Concrete code patterns and best practices for implementing LangGraph runner

This document complements the technical architecture with hands-on implementation guidance, focusing on patterns that ensure the LangGraph runner produces outputs equivalent to the Claude Code runner.

---

## Pattern 1: SpecKit Template Loading

### Problem
LangGraph runner must load SpecKit templates from `.specify/templates/` directory and use them identically to Claude Code runner.

### Implementation Pattern

```python
# /components/runners/langgraph-runner/template_loader.py

from pathlib import Path
from typing import Dict, Optional
import logging

logger = logging.getLogger(__name__)


class SpecKitTemplateLoader:
    """
    Load SpecKit templates for RFE workflow.

    Compatible with SpecKit template structure:
    .specify/templates/
      spec-template.md
      plan-template.md
      tasks-template.md
    """

    def __init__(self, workspace_path: Path):
        self.workspace_path = workspace_path
        self.templates_dir = workspace_path / ".specify" / "templates"

    def load_template(self, template_name: str) -> str:
        """
        Load template file and return content as system prompt.

        Args:
            template_name: Template filename (e.g., "spec-template.md")

        Returns:
            Template content as string

        Raises:
            FileNotFoundError: If template doesn't exist
        """
        template_path = self.templates_dir / template_name

        if not template_path.exists():
            raise FileNotFoundError(
                f"SpecKit template not found: {template_path}\n"
                f"Ensure repositories are seeded before running RFE workflow.\n"
                f"Available templates: {list(self.list_templates().keys())}"
            )

        logger.info(f"Loading SpecKit template: {template_name}")
        content = template_path.read_text(encoding="utf-8")

        return content

    def list_templates(self) -> Dict[str, Path]:
        """List all available templates in .specify/templates/"""
        if not self.templates_dir.exists():
            return {}

        return {
            f.name: f
            for f in self.templates_dir.glob("*.md")
        }

    def validate_templates(self) -> bool:
        """
        Validate that all required RFE templates exist.

        Returns:
            True if all templates present, False otherwise
        """
        required_templates = [
            "spec-template.md",
            "plan-template.md",
            "tasks-template.md",
        ]

        missing = []
        for template in required_templates:
            if not (self.templates_dir / template).exists():
                missing.append(template)

        if missing:
            logger.error(f"Missing required templates: {missing}")
            return False

        logger.info("All required RFE templates present")
        return True
```

**Usage in RFE graph:**
```python
# In rfe_graph.py

async def specify_node(state: RFEState) -> RFEState:
    loader = SpecKitTemplateLoader(Path(state["workspace_path"]))

    # Validate templates before starting
    if not loader.validate_templates():
        raise RuntimeError("Missing required SpecKit templates")

    # Load template as system prompt
    spec_template = loader.load_template("spec-template.md")

    # Call Claude API with template
    spec_content = await generate_with_claude(
        system_prompt=spec_template,
        user_prompt=state["prompt"]
    )

    return {**state, "spec_content": spec_content}
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

        logger.info(f"Initialized SQLite checkpointer at {self.db_path}")

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

## Pattern 3: Streaming Output with Chunking

### Problem
Large RFE outputs (spec.md, plan.md, tasks.md) can overwhelm WebSocket. Need to chunk and stream efficiently.

### Implementation Pattern

```python
# /components/runners/langgraph-runner/streaming.py

from typing import Any
from runner_shell.core.protocol import MessageType, PartialInfo
import json
import asyncio

class StreamingOutputHandler:
    """
    Handles streaming of RFE outputs to WebSocket.

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

    async def stream_file_write(self, file_path: str, content: str):
        """
        Stream file write notification with content preview.

        Useful for showing spec.md, plan.md, tasks.md being written.
        """
        await self.shell._send_message(
            MessageType.SYSTEM_MESSAGE,
            {
                "type": "file_written",
                "file_path": file_path,
                "size_bytes": len(content.encode('utf-8')),
                "preview": content[:500] + ("..." if len(content) > 500 else ""),
            }
        )
```

**Usage in RFE graph:**
```python
async def specify_node(state: RFEState) -> RFEState:
    # ... generate spec_content

    # Write spec.md to workspace
    spec_path = Path(state["workspace_path"]) / "spec.md"
    spec_path.write_text(spec_content)

    # Stream notification
    streaming_handler = state.get("streaming_handler")
    if streaming_handler:
        await streaming_handler.stream_file_write("spec.md", spec_content)

    return {**state, "spec_content": spec_content}
```

---

## Pattern 4: Observability & Metrics

### Problem
Need visibility into RFE graph execution for debugging and performance monitoring.

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
class RFEWorkflowMetrics:
    """Aggregates metrics for RFE workflow execution."""
    thread_id: str
    start_time: datetime
    end_time: Optional[datetime] = None
    nodes_executed: List[NodeExecution] = field(default_factory=list)
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
            "checkpoints_saved": self.checkpoints_saved,
            "total_output_kb": round(self.total_output_bytes / 1024, 2),
            "nodes": [
                {
                    "name": n.node_name,
                    "duration_ms": n.duration_ms,
                    "success": n.success,
                }
                for n in self.nodes_executed
            ],
        }


class ObservabilityManager:
    """Tracks and reports RFE workflow execution metrics."""

    def __init__(self, shell):
        self.shell = shell
        self.metrics: Optional[RFEWorkflowMetrics] = None
        self.current_node: Optional[NodeExecution] = None

    def start_execution(self, thread_id: str):
        """Start tracking a new execution."""
        self.metrics = RFEWorkflowMetrics(
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
```

---

## Pattern 5: Error Recovery & Retries

### Problem
Node failures (API timeouts, template issues) should be recoverable without restarting entire workflow.

### Implementation Pattern

```python
# /components/runners/langgraph-runner/error_handling.py

import asyncio
from typing import Callable
from functools import wraps
import logging

logger = logging.getLogger(__name__)


def with_retry(
    max_attempts: int = 3,
    backoff_base: float = 2.0,
    exceptions: tuple = (Exception,)
):
    """
    Decorator for node functions to add automatic retry logic.

    Usage:
        @with_retry(max_attempts=3)
        async def specify_node(state):
            # node implementation
    """
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        async def wrapper(*args, **kwargs):
            last_exception = None

            for attempt in range(max_attempts):
                try:
                    return await func(*args, **kwargs)

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
            raise RuntimeError(
                f"Node {func.__name__} failed after {max_attempts} attempts: {last_exception}"
            ) from last_exception

        return wrapper
    return decorator


# Example usage in RFE graph:

@with_retry(max_attempts=3, backoff_base=2.0)
async def specify_node(state: RFEState) -> RFEState:
    """Generate specification with automatic retry on Claude API failures."""
    from template_loader import SpecKitTemplateLoader

    loader = SpecKitTemplateLoader(Path(state["workspace_path"]))
    spec_template = loader.load_template("spec-template.md")

    # This may fail due to API rate limits, network issues, etc.
    # Decorator will retry up to 3 times with exponential backoff
    spec_content = await generate_with_claude(
        system_prompt=spec_template,
        user_prompt=state["prompt"]
    )

    spec_path = Path(state["workspace_path"]) / "spec.md"
    spec_path.write_text(spec_content)

    return {**state, "spec_content": spec_content}
```

---

## Pattern 6: Configuration Management

### Problem
Checkpointer settings, retry attempts, pruning policies need to be configurable.

### Implementation Pattern

```python
# /components/runners/langgraph-runner/config_manager.py

from pathlib import Path
from typing import Any
import os


class RunnerConfig:
    """Configuration for LangGraph runner (RFE workflow)."""

    def __init__(self):
        # Checkpoint settings
        self.checkpoint_type = os.getenv("CHECKPOINT_TYPE", "sqlite")
        self.max_checkpoints_per_thread = int(os.getenv("MAX_CHECKPOINTS_PER_THREAD", "10"))
        self.prune_interval_hours = int(os.getenv("PRUNE_INTERVAL_HOURS", "24"))

        # Execution settings
        self.recursion_limit = int(os.getenv("RECURSION_LIMIT", "100"))
        self.timeout_seconds = int(os.getenv("TIMEOUT_SECONDS", "3600"))
        self.retry_attempts = int(os.getenv("RETRY_ATTEMPTS", "3"))

        # Observability
        self.track_metrics = os.getenv("TRACK_METRICS", "true").lower() == "true"
        self.log_node_outputs = os.getenv("LOG_NODE_OUTPUTS", "false").lower() == "true"

    def to_dict(self) -> dict:
        return {
            "checkpoint": {
                "type": self.checkpoint_type,
                "max_checkpoints_per_thread": self.max_checkpoints_per_thread,
                "prune_interval_hours": self.prune_interval_hours,
            },
            "execution": {
                "recursion_limit": self.recursion_limit,
                "timeout_seconds": self.timeout_seconds,
                "retry_attempts": self.retry_attempts,
            },
            "observability": {
                "track_metrics": self.track_metrics,
                "log_node_outputs": self.log_node_outputs,
            }
        }
```

**Usage in adapter:**
```python
class LangGraphAdapter:
    async def initialize(self, context: RunnerContext):
        # Load configuration
        self.config = RunnerConfig()

        # Use configuration
        self.managed_checkpointer = ManagedCheckpointer(
            db_path=checkpoint_path,
            max_checkpoints_per_thread=self.config.max_checkpoints_per_thread,
            prune_interval_hours=self.config.prune_interval_hours,
        )
```

---

## Summary: Key Takeaways

These patterns address common challenges in LangGraph RFE runner implementation:

1. **SpecKit Compatibility**: Template loading pattern ensures identical inputs to Claude API
2. **Resource Management**: Automatic checkpoint pruning prevents PVC exhaustion
3. **Performance**: Streaming with chunking handles large spec/plan/tasks files gracefully
4. **Operations**: Metrics and observability enable debugging in production
5. **Reliability**: Retry logic makes RFE workflow resilient to transient API failures
6. **Flexibility**: Configuration management allows tuning without code changes

**Implementation priority:**
1. Start with Pattern 1 (Template Loading) - core functionality
2. Add Pattern 2 (Checkpointer) - prevents operational issues
3. Layer in Pattern 3 (Streaming) - user experience
4. Enhance with Patterns 4-6 as needed

**Testing each pattern:**
- Unit tests for individual components
- Integration tests with actual RFE workflow
- Comparison tests: LangGraph output vs Claude Code output
- Performance tests: checkpoint I/O, streaming latency

**Next steps:**
1. Prototype Pattern 1 + 2 in sandbox
2. Validate output equivalence with Claude Code runner
3. Benchmark checkpoint performance
4. Document configuration options

---

**Prepared by:** Stella (Staff Engineer)
**Contact:** stella@ambient-code.io
**Date:** 2025-11-04
