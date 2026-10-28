---
type: "Architecture"
title: "Callbacks: Observability and Event Streaming"
description: "Event-driven callback system for monitoring LLM execution, streaming tokens, tracing with LangSmith, and custom instrumentation through handler interfaces and manager patterns."
tags: ["callbacks", "observability", "streaming", "event-driven", "tracing", "instrumentation"]
verified:
  - by: openwiki/0.5.0
    at: 2026-10-28T08:19:50.216Z
sources:
  - id: openwiki-source-96c73d2c05223b0b46abdbe9
    resource: repo://libs/core/langchain_core/callbacks/__init__.py
  - id: openwiki-source-c9313cf42f0120d86b20245f
    resource: repo://libs/core/langchain_core/callbacks/base.py
  - id: openwiki-source-1c233fccf5a66b84d0045366
    resource: repo://libs/core/langchain_core/callbacks/manager.py
  - id: openwiki-source-55497dabc655f803e40dc13a
    resource: repo://libs/core/langchain_core/callbacks/stdout.py
  - id: openwiki-source-2685400b7962e4c90cefe9df
    resource: repo://libs/core/langchain_core/callbacks/usage.py
  - id: openwiki-source-bfd8b1aa6ad00852a2e99762
    resource: repo://libs/core/langchain_core/tracers/context.py
  - id: openwiki-source-79eb321fa44e679f85ad5c48
    resource: repo://libs/core/langchain_core/tracers/langchain.py
generated: { by: "openwiki/0.5.0", at: "2026-10-28T08:19:50.216Z" }
---

## Overview

LangChain's callback system is an event-driven instrumentation layer that fires at key points in the execution lifecycle of LLMs, chains, tools, and retrievers. It enables observability, token streaming, custom instrumentation, and integration with external tracing systems like LangSmith.

The system consists of three layers:

- **Handlers** (`BaseCallbackHandler`, `AsyncCallbackHandler`) that respond to specific events
- **Managers** (`CallbackManager`, `AsyncCallbackManager`) that coordinate handler dispatch and propagate context
- **Run managers** (e.g., `CallbackManagerForLLMRun`) that bind managers to specific executions and enable parent-child hierarchies

## Core Concepts

### Event Lifecycle

Every component type in LangChain fires a standard sequence of events:

| Component | Start | End | Error | Streaming |
|-----------|-------|-----|-------|-----------|
| **LLM** (non-chat) | `on_llm_start` | `on_llm_end` | `on_llm_error` | `on_llm_new_token` |
| **Chat Model** | `on_chat_model_start` | `on_llm_end` | `on_llm_error` | `on_llm_new_token`, `on_stream_event` |
| **Chain** | `on_chain_start` | `on_chain_end` | `on_chain_error` | — |
| **Tool** | `on_tool_start` | `on_tool_end` | `on_tool_error` | — |
| **Retriever** | `on_retriever_start` | `on_retriever_end` | `on_retriever_error` | — |
| **Agent** | `on_chain_start` | `on_chain_end` | `on_chain_error` | `on_agent_action`, `on_agent_finish` |

Each event carries common context parameters:

- **`run_id`** (UUID): Unique identifier for the current execution
- **`parent_run_id`** (UUID | None): ID of the parent run in nested executions
- **`tags`** (list[str] | None): Tags propagated from parent for filtering and metadata correlation
- **`metadata`** (dict[str, Any] | None): Custom key-value metadata propagated through the execution tree

### Handler Interface

`BaseCallbackHandler` defines the core callback interface through mixins, with optional methods that implementations override:

```python
from langchain_core.callbacks import BaseCallbackHandler
from uuid import UUID
from typing import Any

class MyHandler(BaseCallbackHandler):
    """Custom callback handler for observability."""
    
    def on_llm_start(
        self,
        serialized: dict[str, Any],
        prompts: list[str],
        *,
        run_id: UUID,
        parent_run_id: UUID | None = None,
        tags: list[str] | None = None,
        metadata: dict[str, Any] | None = None,
        **kwargs: Any,
    ) -> None:
        """Called when an LLM starts execution."""
        print(f"LLM invoked with {len(prompts)} prompt(s)")
    
    def on_llm_end(self, response, **kwargs: Any) -> None:
        """Called when an LLM finishes successfully."""
        print(f"LLM produced {len(response.generations[0])} completions")
    
    def on_llm_error(self, error: BaseException, **kwargs: Any) -> None:
        """Called when an LLM raises an error."""
        print(f"LLM failed: {type(error).__name__}")
```

**Handler properties:**

- **`raise_error`** (bool, default False): Whether to raise exceptions from callback methods
- **`run_inline`** (bool, default False): Whether to run async handlers synchronously in the calling thread vs. a thread pool executor
- **`ignore_*` properties** (e.g., `ignore_llm`, `ignore_chain`): Allow handlers to filter which event types they receive

### Async Handlers

`AsyncCallbackHandler` extends `BaseCallbackHandler` with async method signatures to support non-blocking execution in async contexts:

```python
from langchain_core.callbacks import AsyncCallbackHandler

class AsyncMyHandler(AsyncCallbackHandler):
    async def on_llm_start(
        self,
        serialized: dict[str, Any],
        prompts: list[str],
        *,
        run_id: UUID,
        **kwargs: Any,
    ) -> None:
        """Async handler for LLM start."""
        await log_to_database(serialized, prompts)
    
    async def on_llm_end(self, response, **kwargs: Any) -> None:
        """Async handler for LLM end."""
        await update_metrics(response)
```

Handlers can mix sync and async methods in the same class for compatibility with both sync and async calling contexts.

### Chat Model Fallback Pattern

When a handler's `on_chat_model_start` raises `NotImplementedError`, the callback system automatically falls back to `on_llm_start` with message lists converted to prompt strings. This enables handlers to support chat models through the legacy LLM interface:

```python
class LegacyHandler(BaseCallbackHandler):
    def on_chat_model_start(
        self,
        serialized: dict[str, Any],
        messages: list[list[BaseMessage]],
        **kwargs: Any,
    ) -> None:
        # Raising NotImplementedError triggers fallback to on_llm_start
        raise NotImplementedError
    
    def on_llm_start(self, serialized, prompts, **kwargs: Any) -> None:
        # This will be called instead for chat models
        print(f"Chat model called with prompts: {prompts}")
```

**Important:** When implementing `on_chat_model_start`, always declare `serialized` and `messages` as explicit positional arguments (not `*args`), as documented in the method signature. This is required for the fallback mechanism to work correctly.

## Callback Managers

Managers coordinate handler dispatch across potentially multiple handlers and manage context propagation (tags, metadata, run IDs) through execution hierarchies.

### Synchronous Manager: CallbackManager

`CallbackManager` is the synchronous coordinator. It dispatches events to all registered handlers and returns specialized run managers bound to individual component invocations:

```python
from langchain_core.callbacks import BaseCallbackHandler, CallbackManager
from uuid import UUID

handler = MyHandler()
manager = CallbackManager(handlers=[handler])

# Start an LLM execution
llm_managers = manager.on_llm_start(
    serialized={"name": "my-llm"},
    prompts=["What is AI?"],
    tags=["user-query"],
    metadata={"user_id": "123"}
)

# Each returned manager tracks one LLM run
for llm_manager in llm_managers:
    # ... invoke LLM ...
    llm_manager.on_llm_end(response)
```

**Start method return types:**

- `on_llm_start()` → `list[CallbackManagerForLLMRun]` (one per prompt)
- `on_chat_model_start()` → `list[CallbackManagerForLLMRun]` (one per message list)
- `on_chain_start()` → `CallbackManagerForChainRun` (single manager)
- `on_tool_start()` → `CallbackManagerForToolRun` (single manager)
- `on_retriever_start()` → `CallbackManagerForRetrieverRun` (single manager)

### Asynchronous Manager: AsyncCallbackManager

`AsyncCallbackManager` is the async variant. It dispatches events to handlers concurrently and returns async run managers:

```python
from langchain_core.callbacks import AsyncCallbackManager

async def main():
    manager = AsyncCallbackManager(handlers=[handler])
    
    # Async dispatch with concurrency
    llm_managers = await manager.on_chat_model_start(
        serialized={"name": "gpt-4"},
        messages=[[message1, message2]],
        tags=["chat"]
    )
    
    for mgr in llm_managers:
        # ... async LLM execution ...
        await mgr.on_llm_end(response)
```

**Concurrency:** Handlers with `run_inline=False` are scheduled concurrently via `asyncio.gather`; those with `run_inline=True` run sequentially.

### Event Dispatch

Two internal functions handle dispatch:

- **`handle_event`** (sync): Dispatches events to handlers, collects coroutines from async handlers invoked synchronously, and executes them in a thread pool to avoid blocking the event loop.

- **`ahandle_event`** (async): Dispatches to async and sync handlers concurrently, running inline handlers sequentially before concurrent ones.

Both functions respect `ignore_*` properties to filter handlers for specific event types.

## Run Managers and Hierarchy

Run managers bind a manager to a specific execution run, enabling tracking of individual invocations and parent-child relationships.

### Base Run Manager

`BaseRunManager` stores the execution context:

```python
class BaseRunManager:
    run_id: UUID                              # Unique ID for this run
    handlers: list[BaseCallbackHandler]       # All handlers for this run
    inheritable_handlers: list[...]           # Handlers to pass to children
    parent_run_id: UUID | None                # ID of parent run
    tags: list[str]                           # Non-inheritable tags
    inheritable_tags: list[str]               # Tags for children
    metadata: dict[str, Any]                  # Non-inheritable metadata
    inheritable_metadata: dict[str, Any]      # Metadata for children
```

### Parent Run Managers and Child Context

`ParentRunManager` (and its async variant `AsyncParentRunManager`) support the creation of child managers that inherit context:

```python
# Inside a chain execution
parent_manager: CallbackManagerForChainRun = ...

# Create a child manager for a nested tool invocation
child_manager = parent_manager.get_child(tag="my-tool")

# Child inherits parent's inheritable_handlers, inheritable_tags, and inheritable_metadata
tool.invoke(input, {"callbacks": child_manager})
```

The hierarchy enables:
- Automatic propagation of tracing context through nested executions
- Parent-child relationships in traces
- Logical grouping of related operations

### Specialized Run Managers

Each component type has a specialized run manager:

- `CallbackManagerForLLMRun`: For LLM invocations; supports token streaming via `on_llm_new_token`
- `CallbackManagerForChainRun`: For chains; supports `get_child()` for nested runs
- `CallbackManagerForToolRun`: For tools
- `CallbackManagerForRetrieverRun`: For retrievers

## Streaming Support

Token-level streaming is supported for LLMs through the `on_llm_new_token` callback:

```python
class StreamingHandler(BaseCallbackHandler):
    def on_llm_new_token(
        self,
        token: str | list[str | dict[str, Any]],
        *,
        chunk: GenerationChunk | ChatGenerationChunk | None = None,
        run_id: UUID,
        **kwargs: Any,
    ) -> None:
        """Called for each token generated during streaming."""
        print(f"Token: {token}")

# Use with streaming LLMs
from langchain_core.callbacks import StreamingStdOutCallbackHandler

llm.invoke(prompt, {"callbacks": [StreamingStdOutCallbackHandler()]})
```

The `chunk` parameter provides detailed information about the generated output when available.

Protocol-level event streaming (v3) is supported through `on_stream_event` for structured chat model events:

```python
class EventStreamHandler(BaseCallbackHandler):
    def on_stream_event(
        self,
        event: MessagesData,  # Protocol event
        *,
        run_id: UUID,
        **kwargs: Any,
    ) -> None:
        """Called for each protocol event (message-start, content-block-delta, etc.)."""
        print(f"Event: {event}")
```

## Custom Events

Applications can emit custom instrumentation events using `dispatch_custom_event` (sync) and `adispatch_custom_event` (async). Handlers implement `on_custom_event` to react:

```python
from langchain_core.callbacks import dispatch_custom_event
from langchain_core.callbacks import BaseCallbackHandler

class CustomEventHandler(BaseCallbackHandler):
    def on_custom_event(
        self,
        name: str,
        data: Any,
        *,
        run_id: UUID,
        tags: list[str] | None = None,
        metadata: dict[str, Any] | None = None,
        **kwargs: Any,
    ) -> None:
        """Handle a custom application event."""
        print(f"Custom event '{name}': {data}")

# Dispatch from a RunnableLambda or tool
def my_tool(input: str, config: RunnableConfig):
    # ... do work ...
    dispatch_custom_event("my_event", {"result": "success"}, config=config)
    return output
```

Custom events must be dispatched from within an active run (e.g., inside a tool or RunnableLambda) and require a parent run ID context.

## LangSmith Integration

LangChain integrates with LangSmith for production tracing through the `LangChainTracer` callback handler:

### Automatic Activation

Tracing is automatically activated when `LANGCHAIN_TRACING_V2=true` environment variable is set:

```python
import os
os.environ["LANGCHAIN_TRACING_V2"] = "true"

# Traces are now sent to LangSmith automatically
chain.invoke({"input": "test"})
```

### Explicit Activation

Use the `tracing_v2_enabled()` context manager for explicit control:

```python
from langchain_core.tracers.context import tracing_v2_enabled

with tracing_v2_enabled(project_name="my-project") as tracer:
    result = chain.invoke({"input": "test"})
    run_url = tracer.get_run_url()  # Get the LangSmith trace URL
```

### Grouped Tracing

Use `trace_as_chain_group` and `atrace_as_chain_group` to group multiple independent operations as children of a parent run:

```python
from langchain_core.callbacks.manager import trace_as_chain_group

with trace_as_chain_group("my_workflow", inputs={"task": "demo"}) as manager:
    result1 = llm.invoke(prompt1, {"callbacks": manager})
    result2 = tool.invoke(input2, {"callbacks": manager})
    # Both runs appear as children of the same trace
```

## Built-in Handlers

LangChain provides several production-ready handlers:

### StdOutCallbackHandler

Prints chain execution to stdout:

```python
from langchain_core.callbacks import StdOutCallbackHandler

handler = StdOutCallbackHandler()
chain.invoke(input, {"callbacks": [handler]})
# Outputs: > Entering new ... chain...
#          > Finished chain.
```

### StreamingStdOutCallbackHandler

Streams individual tokens to stdout as they are generated:

```python
from langchain_core.callbacks import StreamingStdOutCallbackHandler

llm.invoke(prompt, {"callbacks": [StreamingStdOutCallbackHandler()]})
# Outputs tokens in real-time
```

### UsageMetadataCallbackHandler

Tracks token usage (input/output tokens, cost) from LLM responses:

```python
from langchain_core.callbacks import UsageMetadataCallbackHandler

handler = UsageMetadataCallbackHandler()
llm.invoke(prompt, {"callbacks": [handler]})
print(handler.get_total_token_count())  # Total tokens used
print(handler.get_total_cost())          # Estimated cost
```

### FileCallbackHandler

Logs execution traces to a file.

## Manager Configuration

The `CallbackManager.configure()` class method aggregates callbacks from multiple sources with environment variable integration:

```python
from langchain_core.callbacks import CallbackManager

manager = CallbackManager.configure(
    inheritable_callbacks=[handler1],        # Passed to child runs
    local_callbacks=[handler2],              # Not passed to children
    inheritable_tags=["production"],        # Tags for child runs
    local_tags=["debug"],                   # Tags for this run only
    inheritable_metadata={"project": "x"},  # Metadata for child runs
    local_metadata={"request_id": "123"},   # Metadata for this run only
    verbose=True,                           # Enable verbose mode
)
```

This method is typically called by runnables and chains to set up callbacks with proper inheritance semantics.

## Testing and Debugging

### Run Collection

Collect all traces in memory for testing without LangSmith:

```python
from langchain_core.tracers.context import collect_runs

with collect_runs() as run_collector:
    chain.invoke({"input": "test"})
    
# Access traced runs
for run in run_collector.traced_runs:
    print(f"Run: {run.name}")
    print(f"Output: {run.outputs}")
```

This is useful for testing execution flow and verifying correct event ordering without sending data to an external service.

## Error Handling

The callback system is resilient to handler errors:

- Exceptions in handlers are logged by default (not raised)
- Set `raise_error=True` on a handler to propagate its exceptions
- The `ignore_*` properties prevent handlers from receiving unwanted events, reducing error surface
- Async handlers invoked from sync contexts have exceptions logged and swallowed to avoid deadlocking the event loop

## Extension Points

To extend the callback system:

1. **Create a custom handler:** Subclass `BaseCallbackHandler` or `AsyncCallbackHandler` and override event methods
2. **Filter event types:** Use the `ignore_*` properties to optimize handler execution
3. **Emit custom events:** Use `dispatch_custom_event` or `adispatch_custom_event` from application code
4. **Integrate external systems:** Extend `BaseCallbackHandler` to send events to monitoring, logging, or analytics services
5. **Implement parent-child hierarchies:** Use run managers' `get_child()` method to enable automatic context propagation

Handlers receive complete execution context (run IDs, tags, metadata) enabling flexible instrumentation strategies.
