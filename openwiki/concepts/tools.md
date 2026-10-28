---
type: "Agent Integration Layer"
title: "Tools: Agent Actions and Provider Integration"
description: "Structured interfaces for language models to invoke external functions, with schema generation, validation, error handling, and runtime injection of framework context."
tags: ["tools", "agents", "schema-inference", "runnable-integration", "structured-output"]
verified:
  - by: openwiki/0.5.0
    at: 2026-10-28T08:19:50.216Z
sources:
  - id: openwiki-source-c52037e7b642f7ac5a7642a8
    resource: repo://libs/core/langchain_core/language_models/chat_models.py
  - id: openwiki-source-4ff475d7b00540f962384251
    resource: repo://libs/core/langchain_core/tools/base.py
  - id: openwiki-source-9c422fcb5ac12738f17d1cd1
    resource: repo://libs/core/langchain_core/tools/convert.py
  - id: openwiki-source-1ab4436ccb637ddf41e35732
    resource: repo://libs/core/langchain_core/tools/render.py
  - id: openwiki-source-80e84f93417c922f44011393
    resource: repo://libs/core/langchain_core/tools/simple.py
  - id: openwiki-source-b816e651a5890bde13cf8013
    resource: repo://libs/core/langchain_core/tools/structured.py
generated: { by: "openwiki/0.5.0", at: "2026-10-28T08:19:50.216Z" }
---

## Overview

Tools extend language models with the ability to interact with external systems and accomplish specialized tasks. In LangChain, tools are structured components that agents select and invoke to fulfill user objectives. Each tool provides a JSON schema describing its inputs, a description guiding model selection, and execution logic supporting both sync and async operation.

A tool workflow follows three phases:

1. **Schema Definition & Model Binding**: Tools define input requirements via JSON schemas so models can generate well-formed invocations
2. **Tool Calling**: The language model generates structured tool calls referencing tool names and arguments
3. **Execution & Response**: The tool validates inputs, executes logic, applies error handling, and returns results wrapped in a `ToolMessage` to continue the agent loop

## Tool Foundations

### BaseTool: Universal Interface

`BaseTool` is an abstract `RunnableSerializable` that accepts string, dict, or `ToolCall` inputs and returns typed output. All LangChain tools inherit from `BaseTool` and must implement the abstract `_run` method for sync execution and optionally `_arun` for async.

Core responsibilities:

- **Input schema management**: Store schema as Pydantic `BaseModel`, dict (JSON Schema), or auto-inferred from function signatures
- **Input validation & parsing**: Convert user input (string or dict) to validated arguments, applying Pydantic defaults and filtering framework arguments
- **Execution orchestration**: Call `_run` or `_arun` with callback context, handle exceptions per configuration, and format output
- **Output wrapping**: Convert tool result to `ToolMessage` when invoked with a tool call ID, or return raw output otherwise
- **Runtime injection**: Supply framework-injected arguments (`run_manager`, `RunnableConfig`) and user-injected arguments (`InjectedToolArg`) at execution time

Core properties:

- **name**: Unique identifier used by models and in tool call references
- **description**: Explains the tool's purpose, parameters, and usage; guides model selection
- **args_schema**: Pydantic `BaseModel` or JSON schema dict defining input arguments. When omitted, inferred from function signature
- **return_direct**: Boolean flag controlling whether the agent loop terminates after this tool executes (typically for final-answer tools)
- **response_format**: Controls output wrapping—`'content'` (default, treats output as message content) or `'content_and_artifact'` (expects tuple of content and artifact)

Error handling configuration:

- **handle_tool_error**: Governs how `ToolException` is handled: `False` (re-raise), `True` (return exception message), string (return custom message), or callable (custom handler receiving the exception)
- **handle_validation_error**: Governs how input validation errors are handled: `False` (re-raise), `True` (return generic error), string (custom message), or callable (custom handler)

Additional fields:

- **callbacks**: Lifecycle event handlers for observability
- **tags** / **metadata**: Labels for tracking and instrumentation
- **extras**: Provider-specific configuration (e.g., Anthropic `cache_control`, `defer_loading`)

### The @tool Decorator

The `@tool` decorator is the most concise way to create tools from Python functions. It supports multiple usage patterns:

```python
# Bare decorator - infers schema from function signature
@tool
def search(query: str) -> str:
    """Search for the query."""
    return f"Results for {query}"

# With custom name
@tool("search_documents")
def my_search(q: str) -> str:
    """Search documents."""
    return q

# With configuration
@tool(description="Advanced search", return_direct=True)
def advanced_search(q: str, limit: int = 10) -> str:
    """Search with advanced options."""
    return f"Searched {q} with limit {limit}"

# With explicit schema
@tool(args_schema=MySchema)
def configured_tool(x: int) -> str:
    """Tool with explicit schema."""
    return str(x)

# Async tools
@tool
async def async_search(query: str) -> str:
    """Async search implementation."""
    await asyncio.sleep(0.1)
    return f"Async results for {query}"

# With Google-style docstring parsing
@tool(parse_docstring=True)
def documented_tool(query: str, limit: int) -> str:
    """Search for documents.
    
    Args:
        query: The search query
        limit: Maximum results to return
    """
    return f"Found {limit} results for {query}"
```

The decorator creates either a `StructuredTool` (when schema inference is enabled) or a simple `Tool` (for single string inputs when `infer_schema=False`).

### StructuredTool

`StructuredTool` wraps a function or coroutine and enforces a Pydantic schema. It accepts multiple named arguments, validates them against the schema, and passes them to the wrapped function.

```python
from langchain_core.tools import StructuredTool
from pydantic import BaseModel

class SearchSchema(BaseModel):
    query: str
    limit: int = 10

def search_fn(query: str, limit: int) -> str:
    return f"Searched {query} with limit {limit}"

tool = StructuredTool.from_function(
    search_fn,
    name="search",
    description="Search for documents",
    args_schema=SearchSchema
)
```

### Simple Tool (Tool)

`Tool` is a lightweight variant for single string inputs and string outputs. Used when `infer_schema=False` or for backwards compatibility.

### Class-Based Tools

Custom tools can extend `BaseTool` directly:

```python
from langchain_core.tools import BaseTool
from pydantic import BaseModel

class MySchema(BaseModel):
    x: int

class CustomTool(BaseTool):
    name: str = "custom_tool"
    description: str = "Does something custom"
    args_schema: type[BaseModel] = MySchema
    
    def _run(self, **kwargs) -> str:
        # Sync implementation
        return "result"
    
    async def _arun(self, **kwargs) -> str:
        # Async implementation
        return "result"
```

## Schema Generation and Inference

### Automatic Schema Inference

When a tool is created with `infer_schema=True` (the default), the schema is automatically derived from the function signature and type annotations:

```python
@tool
def process(value: int, flag: bool = False) -> str:
    """Process a value with optional flag."""
    return f"Processed {value}"
```

The decorator generates a Pydantic model with `value` (required) and `flag` (optional, default False). The tool's JSON schema becomes `model_json_schema()` of this model.

The `create_schema_from_function` utility does the heavy lifting:

1. Extracts function signature and type hints (including `Annotated` metadata for field descriptions)
2. Uses Pydantic's `validate_arguments` to build an initial model
3. Filters out framework arguments (`run_manager`, `callbacks`, `self`/`cls`)
4. Filters out injected arguments annotated with `InjectedToolArg` or `InjectedToolCallId`
5. Applies docstring parameter descriptions if `parse_docstring=True`
6. Creates a Pydantic subset model with only user-supplied fields

### Manual Schema Definition

Pass an explicit Pydantic model as `args_schema`:

```python
from pydantic import BaseModel, Field

class InputSchema(BaseModel):
    query: str = Field(..., description="Search query")
    limit: int = Field(default=10, description="Result limit")

@tool(args_schema=InputSchema)
def search(query: str, limit: int) -> str:
    return f"Searched {query} with limit {limit}"
```

### JSON Schema Format

For non-Pydantic use cases, pass a JSON schema dict:

```python
tool = BaseTool(
    name="my_tool",
    description="Tool description",
    args_schema={
        "type": "object",
        "properties": {
            "param": {"type": "string"}
        },
        "required": ["param"]
    }
)
```

### Parsing Google-Style Docstrings

When `parse_docstring=True`, parameter descriptions are extracted from Google-style docstrings and merged into the schema:

```python
@tool(parse_docstring=True)
def search(query: str, limit: int = 10) -> str:
    """Search for documents.
    
    Args:
        query: The search query string
        limit: Maximum number of results
    """
    return query
```

The resulting schema includes both the parameter types and the documented descriptions.

## Runtime Argument Injection

Tool arguments can be marked for framework injection rather than user input. These arguments are excluded from the tool schema sent to models but injected at execution time.

### InjectedToolArg

Mark an argument with `InjectedToolArg` to exclude it from the model-facing schema:

```python
from typing import Annotated
from langchain_core.tools import tool, InjectedToolArg

@tool
def my_tool(
    query: str,
    injected_value: Annotated[str, InjectedToolArg]
) -> str:
    """Tool with injected argument."""
    return f"{query} + {injected_value}"
```

The `injected_value` parameter does not appear in the tool's JSON schema and must be supplied at call time via the tool execution framework.

### InjectedToolCallId

Receive the tool call ID (generated by the model):

```python
from typing import Annotated
from langchain_core.messages import ToolMessage
from langchain_core.tools import tool, InjectedToolCallId

@tool
def my_tool(
    x: int,
    tool_call_id: Annotated[str, InjectedToolCallId]
) -> ToolMessage:
    """Tool that receives its call ID."""
    return ToolMessage(str(x), tool_call_id=tool_call_id, name="my_tool")
```

## Tool Calling Workflow

### Schema Binding to Models

When tools are bound to a chat model via `bind_tools()`, the model receives each tool's JSON schema:

```python
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4")
tools = [search_tool, calculator_tool]
bound_model = model.bind_tools(tools)
```

The `bind_tools` method on `BaseChatModel` returns a new `Runnable` that instructs the model to generate `ToolCall` objects in its response. Models format tool calls as structured objects containing name, arguments, and a unique ID.

### Tool Call Structure

A tool call is a structured object containing:

- **name**: The tool identifier (matches tool instance name)
- **args**: Argument dict (or single string for simple tools)
- **tool_call_id** (or **id**): Unique ID generated by the model (typically UUID or provider-specific string)

The framework recognizes tool calls via `isinstance(value, dict) and value.get("type") == "tool_call"`.

### Input Validation and Parsing

When a tool is invoked, the `_parse_input` method validates input against the tool's `args_schema`:

1. If the input is a dict, validate it against the Pydantic schema (if present)
2. Apply Pydantic defaults for fields with explicit defaults
3. Inject runtime-supplied arguments (`tool_call_id` if the schema expects it via `InjectedToolCallId`)
4. Filter out framework arguments (`run_manager`, `callbacks`) and injected args from the final kwargs passed to the wrapped function

The `_to_args_and_kwargs` method converts the validated input to positional/keyword arguments suitable for the tool's `_run` method.

### Execution and Error Handling

The `run()` method (or async `arun()`) orchestrates tool execution:

1. **Configure callbacks**: Set up callback managers for lifecycle events
2. **Emit start event**: Fire `on_tool_start` for observability
3. **Validate input**: Use `_parse_input` to validate and parse the input dict
4. **Convert to args/kwargs**: Use `_to_args_and_kwargs` to prepare function arguments
5. **Inject runtime context**: If the `_run` signature accepts `run_manager` or `config`, inject these
6. **Execute**: Call `_run` (or `_arun` for async) within config context
7. **Handle errors**: Catch `ToolException` and `ValidationError`, applying the configured error handlers
8. **Format output**: Wrap result in `ToolMessage` (if `tool_call_id` is provided) using `_format_output`
9. **Emit end/error event**: Fire `on_tool_end` or `on_tool_error` for observability

```python
try:
    result = tool.run(
        {"query": "test"},
        tool_call_id="call_123"
    )
    # result is a ToolMessage if tool_call_id was provided
except ToolException as e:
    # Handled per handle_tool_error
    pass
```

### Output Formatting

Tools support two response formats:

**content** (default): Tool output becomes the `ToolMessage` content.

**content_and_artifact**: Tool output is a `(content, artifact)` tuple; both are stored in the `ToolMessage`.

```python
@tool(response_format="content_and_artifact")
def structured_output() -> tuple[str, dict]:
    """Returns content and artifact."""
    return "summary", {"detailed": "data"}
```

The `_format_output` utility:
- Detects if output is a `ToolMessage` or `ToolOutputMixin` (returned as-is)
- Detects if output is a list of `ToolOutputMixin` instances (returned as-is)
- Normalizes content to string or list of message content blocks
- Wraps in `ToolMessage` with provided `tool_call_id`, `name`, and `status`

## Rendering Tools for Display

Tools can be rendered as text for display in prompts or logs:

```python
from langchain_core.tools import render_text_description, render_text_description_and_args

# Render just names and descriptions
text = render_text_description([search_tool, calc_tool])
# Output:
# search - Search for information
# calculator - Perform math calculations

# Render with argument schemas
detailed = render_text_description_and_args([search_tool])
# Output:
# search - Search for information, args: {"query": {"type": "string"}}
```

## Tool Schema Lifecycle and Caching

### Tool Call Schema Memoization

For performance, the `tool_call_schema` property (the schema sent to models, excluding injected arguments) is memoized per tool instance. This avoids regenerating Pydantic subset models and JSON schemas on every agent invocation.

The memoization is cleared when `name`, `description`, or `args_schema` is reassigned:

```python
tool = search_tool
schema1 = tool.tool_call_schema  # Generated and cached

tool.description = "Updated description"
# Memo cleared; next access regenerates

schema2 = tool.tool_call_schema  # New schema generated and cached
```

The subset model class is cached in `_tool_call_schema_memo` and its `model_json_schema()` method is patched to cache the generated dict. The patch avoids Pydantic's lack of per-class caching, which would otherwise cause agent loops to pay full schema generation cost on every model invocation.

### Pickling

Tools can be pickled for serialization and deployment. Memoized schemas (dynamically created Pydantic classes) are dropped during pickling and rebuilt lazily on next access.

## Advanced Patterns

### Custom Error Handling

Define custom logic for handling tool errors and validation errors:

```python
def handle_search_error(e: ToolException) -> str:
    if "timeout" in str(e).lower():
        return "Search timed out. Please try a simpler query."
    return f"Search failed: {e}"

@tool(handle_tool_error=handle_search_error)
def search(query: str) -> str:
    if len(query) > 100:
        raise ToolException("Query too long")
    return f"Found: {query}"
```

### Tool Composition in Agents

Agents select tools based on descriptions and execute them in sequence:

```python
from langchain_core.agents import create_react_agent

tools = [search_tool, calculator_tool, weather_tool]
agent = create_react_agent(model, tools, prompt)
result = agent.invoke({"input": "What is the square root of 144?"})
```

Each tool call produces a `ToolMessage` that becomes input context for the next model invocation.

### Tool Metadata and Tags

Tools support metadata for tracking and instrumentation:

```python
@tool(tags=["search", "documentation"])
def search(query: str) -> str:
    """Search documentation."""
    return query

tool.metadata = {
    "version": "1.0",
    "category": "search",
    "provider": "langchain"
}
```

### Provider-Specific Extras

Pass provider-specific configuration via `extras`:

```python
@tool(extras={"cache_control": {"type": "ephemeral"}})
def cached_search(query: str) -> str:
    """Search with caching."""
    return query
```

Chat model implementations read the `extras` field when constructing provider payloads and include provider-specific configuration in tool definitions.

## Key Responsibilities and Design Principles

**Schema Management**: Tools maintain input schema as Pydantic models, JSON dicts, or auto-inferred from signatures. Schemas are exposed in model-facing and internal forms, with injected arguments filtered from the former.

**Input Validation**: Validate user-supplied arguments against schemas, apply Pydantic defaults, handle validation errors per configuration, and filter framework arguments before passing to the wrapped function.

**Execution**: Run tool logic synchronously or asynchronously, catch exceptions, apply error handlers per configuration, and format output with optional wrapping in `ToolMessage`.

**Callback Integration**: Emit lifecycle events (`on_tool_start`, `on_tool_end`, `on_tool_error`) for observability, debugging, and instrumentation.

**Runnable Protocol**: Implement `invoke` and `ainvoke` for pipeline composition, enabling tools to be used in LangChain's composable runtime system.

**Model Integration**: Expose schemas in provider formats (OpenAI, Anthropic, etc.) via `bind_tools` for structured tool calling and agent execution.

**Runtime Injection**: Support framework-injected arguments (callbacks, config, run managers) and user-injected arguments (via `InjectedToolArg`) without requiring them in the model-facing schema.
