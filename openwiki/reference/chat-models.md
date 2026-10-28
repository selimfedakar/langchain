---
type: Reference
title: Chat Models
description: Core abstractions and APIs for building conversational AI with LangChain, including BaseChatModel implementation contract, streaming architecture, tool calling, structured output, and integration patterns.
tags: [chat-model, llm, api-reference, messaging, streaming, tool-calling]
verified:
  - by: openwiki/0.5.0
    at: 2026-10-28T08:19:50.216Z
sources:
  - id: openwiki-source-c52037e7b642f7ac5a7642a8
    resource: repo://libs/core/langchain_core/language_models/chat_models.py
generated: { by: "openwiki/0.5.0", at: "2026-10-28T08:19:50.216Z" }
---

# Chat Models

Chat models form the central abstraction for conversational AI in LangChain. They accept sequences of messages and return AI-generated responses, exposing both streaming and batching capabilities. This reference documents the core `BaseChatModel` class, its implementation contract, API methods, lifecycle, and integration patterns.

## Overview: BaseChatModel Architecture

`BaseChatModel` (in `langchain_core.language_models.chat_models`) serves as the abstract base class for all chat model implementations. It extends `BaseLanguageModel` to provide:

1. **Unified Message Interface**: Accepts lists of `BaseMessage` objects and returns `AIMessage` responses, enabling consistent API across providers.
2. **Runnable Integration**: Implements the `Runnable` protocol, allowing chat models to be composed into chains and workflows.
3. **Streaming Support**: Provides both legacy V1 callback-based streaming and protocol-event-based V2/V3 streaming.
4. **Tool Calling and Structured Output**: Enables binding tools to models and enforcing schema-based responses.
5. **Generation Lifecycle**: Manages caching, rate limiting, callback dispatch, and error handling.

## Implementation Contract

### Required Overrides

Custom chat model implementations **must** provide:

#### `_generate(messages, stop, run_manager, **kwargs) → ChatResult`

The core synchronous generation method. Called by `invoke()`, `generate()`, and streaming operations.

**Parameters:**
- `messages: list[BaseMessage]` - Normalized input messages (see Message Conversion below).
- `stop: list[str] | None` - Optional stop sequences to terminate generation.
- `run_manager: CallbackManagerForLLMRun | None` - Callback manager for lifecycle events.
- `**kwargs` - Provider-specific parameters (temperature, max_tokens, etc.).

**Returns:** `ChatResult` containing:
- `generations: list[ChatGeneration]` - One generation per message list (typically one).
- `llm_output: dict[str, Any] | None` - Provider-specific output (token counts, metadata).

**Example:**
```python
def _generate(self, messages, stop=None, run_manager=None, **kwargs):
    # Convert messages to provider format
    # Call provider API
    # Extract AIMessage and metadata
    return ChatResult(generations=[ChatGeneration(message=ai_msg)])
```

#### `_llm_type` Property

A string identifier for logging and serialization (e.g., `"openai"`, `"anthropic"`, `"ollama"`). Used in traces and model discovery.

```python
@property
def _llm_type(self) -> str:
    return "openai"
```

### Optional Overrides

Implementations may override these for optimized paths:

#### `_stream(messages, stop, run_manager, **kwargs) → Iterator[ChatGenerationChunk]`

Yields streaming chunks incrementally. If not provided, `stream()` falls back to calling `invoke()` and wrapping the result.

**Yields:** `ChatGenerationChunk` objects with incremental content and metadata.

#### `_agenerate(messages, stop, run_manager, **kwargs) → ChatResult`

Native asynchronous generation. If not provided, `_generate()` runs in a thread executor.

#### `_astream(messages, stop, run_manager, **kwargs) → AsyncIterator[ChatGenerationChunk]`

Native asynchronous streaming. If not provided, `_stream()` runs in a thread executor.

#### `_identifying_params` Property

Provider-specific parameters for tracing and debugging (optional, defaults to empty dict).

```python
@property
def _identifying_params(self) -> dict[str, Any]:
    return {"model": self.model, "temperature": self.temperature}
```

#### `bind_tools(tools, tool_choice=None, **kwargs) → Runnable`

Converts and binds tools to the model. Base implementation raises `NotImplementedError`; providers override to encode tools into API format.

#### `with_structured_output(schema, include_raw=False, **kwargs) → Runnable`

Wraps the model to enforce schema-based responses. Providers may override for native schema support.

## Input and Message Conversion

### Input Types (`LanguageModelInput`)

Chat models accept three input forms:

1. **String**: Converted to `StringPromptValue` → `HumanMessage`.
   ```python
   model.invoke("What is the capital of France?")
   ```

2. **Message List**: List of `BaseMessage` or message-like tuples/dicts, converted to `ChatPromptValue`.
   ```python
   from langchain_core.messages import HumanMessage, SystemMessage
   
   model.invoke([
       SystemMessage(content="You are a helpful assistant."),
       HumanMessage(content="Hello!")
   ])
   ```

3. **PromptValue**: Existing `ChatPromptValue` or `StringPromptValue` passed through unchanged.

### Message Types

Standard message classes in `langchain_core.messages`:

| Type | Role | Usage |
|------|------|-------|
| `HumanMessage` | User input | Questions, instructions, user-side content. |
| `AIMessage` | Model response | Returned by chat models; may contain `tool_calls` and `additional_kwargs`. |
| `SystemMessage` | Model instructions | Guides behavior; provider-specific handling. |
| `ToolMessage` | Tool result | Contains output from tool execution; requires `tool_call_id`. |
| `FunctionMessage` | Legacy tool result | OpenAI-specific; largely superseded by `ToolMessage`. |

### Message Content

Message content can be:
- **String**: Simple text (`"Hello world"`).
- **List of Content Blocks**: Multimodal support for images, audio, files, and text.
  ```python
  from langchain_core.messages import HumanMessage
  
  msg = HumanMessage(content=[
      {"type": "text", "text": "What's in this image?"},
      {"type": "image_url", "image_url": {"url": "https://..."}}
  ])
  ```

### Message Normalization

Input messages are converted to `BaseMessage` objects via `_convert_input()`, then normalized via `_normalize_messages()` to ensure consistent internal representation. This includes:
- Converting message-like tuples/dicts to proper `BaseMessage` instances.
- Extracting and preserving metadata fields (tool calls, usage, response metadata).
- Provider-specific transformation (e.g., OpenAI role/content format).

## Core API Methods

### Single-Shot Invocation

#### `invoke(input, config=None, *, stop=None, **kwargs) → AIMessage`

Synchronous single call. Blocks until response received.

```python
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o")
response = model.invoke("What is LangChain?")
print(response.content)
```

**Returns:** `AIMessage` with content and optional metadata.

#### `ainvoke(input, config=None, *, stop=None, **kwargs) → AIMessage`

Asynchronous variant. Defaults to running `invoke()` in a thread executor unless overridden.

```python
response = await model.ainvoke("What is LangChain?")
```

### Streaming

#### `stream(input, config=None, *, stop=None, **kwargs) → Iterator[AIMessageChunk]`

Yields message chunks as they arrive from the model.

**Behavior:**
- If `_stream()` not implemented, falls back to `invoke()` and yields result.
- Dispatches `on_llm_new_token()` callbacks for real-time processing.
- Attaches metadata and run ID to each chunk.

```python
for chunk in model.stream("Tell me a story"):
    print(chunk.content, end="", flush=True)
```

#### `astream(input, config=None, *, stop=None, **kwargs) → AsyncIterator[AIMessageChunk]`

Asynchronous streaming. Defaults to async executor unless overridden.

```python
async for chunk in model.astream("Tell me a story"):
    print(chunk.content, end="", flush=True)
```

#### `stream_events(input, config=None, *, version="v2", **kwargs) → Iterator[StreamEvent] | ChatModelStream`

Streams structured events for LangGraph and tracing integration.

**Versions:**
- `version="v1"`: Legacy callback-based events.
- `version="v2"` (default): Protocol events with richer structure.
- `version="v3"`: Content-block-centric streaming (beta) returning `ChatModelStream` with typed projections (`.text`, `.reasoning`, `.tool_calls`, `.output`).

```python
async for event in model.astream_events("Solve 2+2", version="v2"):
    print(event["event"])  # "on_chat_model_start", "on_chat_model_stream", "on_chat_model_end"
```

### Batch Generation

#### `generate(messages, stop=None, callbacks=None, *, tags=None, metadata=None, run_name=None, run_id=None, **kwargs) → LLMResult`

Processes multiple message sequences in a batch. Callbacks are invoked per batch with aggregated metadata.

```python
result = model.generate([
    [HumanMessage(content="What is 2+2?")],
    [HumanMessage(content="What is 3+3?")]
])

for generation_list in result.generations:
    print(generation_list[0].message.content)
```

**Returns:** `LLMResult` with:
- `generations: list[list[ChatGeneration]]` - One list per input message sequence.
- `llm_output: dict[str, Any] | None` - Combined metadata (usually `None`).
- `run: list[RunInfo]` - Run IDs if callbacks were used.

#### `agenerate(messages, stop=None, callbacks=None, *, tags=None, metadata=None, run_name=None, run_id=None, **kwargs) → LLMResult`

Asynchronous batch generation. Concurrently processes each message list.

```python
result = await model.agenerate([
    [HumanMessage(content="What is 2+2?")],
    [HumanMessage(content="What is 3+3?")]
])
```

#### `generate_prompt(prompts, stop=None, callbacks=None, **kwargs) → LLMResult`

Converts `PromptValue` objects to messages, then calls `generate()`.

#### `agenerate_prompt(prompts, stop=None, callbacks=None, **kwargs) → LLMResult`

Async variant of `generate_prompt()`.

### Batch Convenience Methods (Runnable Protocol)

#### `batch(inputs, config=None, *, return_exceptions=False, **kwargs) → list[AIMessage]`

Runs `invoke()` concurrently on multiple inputs.

```python
responses = model.batch(["What is 2+2?", "What is 3+3?"])
```

#### `abatch(inputs, config=None, *, return_exceptions=False, **kwargs) → list[AIMessage]`

Async concurrent invocation.

```python
responses = await model.abatch(["What is 2+2?", "What is 3+3?"])
```

#### `batch_as_completed(inputs, config=None, *, return_exceptions=False, **kwargs) → Iterator[tuple[int, AIMessage | Exception]]`

Returns results as they complete, preserving input order via indices.

```python
for index, response in model.batch_as_completed(inputs):
    print(f"Input {index}: {response.content}")
```

## Streaming Architecture

### V1 Streaming (Legacy Callback-Based)

Triggered when:
- A `_StreamingCallbackHandler` is attached to run manager.
- `stream=True` is explicitly passed.
- Instance-level `streaming=True` attribute is set.

**Flow:**
1. `stream()` / `astream()` calls `_stream()` / `_astream()`.
2. Each chunk is yielded incrementally.
3. `on_llm_new_token()` callback fires for each chunk.
4. `on_llm_end()` fires after all chunks received or on error.

**Handler interface:**
```python
class MyStreamingHandler(BaseCallbackHandler):
    def on_llm_new_token(self, token: str, **kwargs):
        print(token, end="", flush=True)
```

### V2 Streaming (Protocol Events)

Triggered when a `_V2StreamingCallbackHandler` is attached. Provides structured events for LangGraph and tracing.

**Flow:**
1. `_generate_with_cache()` checks for V2 handlers.
2. Calls `_iter_v2_events()` / `_aiter_v2_events()`.
3. Prefers native `_stream_chat_model_events()` / `_astream_chat_model_events()` if provided.
4. Otherwise, bridges `_stream()` chunks via `chunks_to_events()`.
5. Each event fires `on_stream_event()`.

**Events:** `MessagesData` protocol events structured as content blocks with type, index, and incremental content.

### V3 Streaming (Content-Block-Centric, Beta)

Accessed via `stream_events(version="v3")` or `astream_events(version="v3")`.

**Returns:** `ChatModelStream` or `AsyncChatModelStream` exposing:
- `.text` - Accumulated text content.
- `.reasoning` - Reasoning content (if supported by model).
- `.tool_calls` - Parsed tool calls.
- `.output` - Final assembled `AIMessage`.
- `.done` - Whether stream is complete.

```python
stream = model.stream_events("Solve this: 2+2", version="v3")
for chunk in stream.text:
    print(chunk, end="", flush=True)
final_msg = stream.output
```

### Streaming Disabling

The `disable_streaming` field controls when streaming is suppressed:

| Value | Behavior |
|-------|----------|
| `True` | Always bypass streaming; use non-streaming `invoke()`. |
| `"tool_calling"` | Bypass streaming only when `tools` are bound. |
| `False` (default) | Use streaming if available and requested. |

```python
model = ChatOpenAI(..., disable_streaming="tool_calling")
# stream() works normally
response = model.stream("Hello")
# but tool calling uses non-streaming
response = model.bind_tools(tools).stream("Use a tool")
```

## Tool Calling

### Binding Tools

#### `bind_tools(tools, tool_choice=None, **kwargs) → Runnable[LanguageModelInput, AIMessage]`

Converts and binds tool schemas to the model, returning a wrapped `Runnable` that enforces tool calling.

**Tool Specification:**

Tools can be:
- `BaseTool` instances (have `.invoke()` and `args_schema`).
- Pydantic `BaseModel` classes (with docstrings and field descriptions).
- Plain Python functions (analyzed for signature and docstring).
- Dictionary tool schemas (raw OpenAI/Anthropic format).

```python
from langchain_core.tools import tool
from pydantic import BaseModel

@tool
def get_weather(location: str) -> str:
    """Get weather for a location."""
    return f"Sunny in {location}"

class SearchInput(BaseModel):
    """Search input."""
    query: str
    max_results: int = 10

bound = model.bind_tools([get_weather, SearchInput])
```

**Parameters:**
- `tools: Sequence[...]` - Tools to bind.
- `tool_choice: str | None` - Tool selection hint (`"any"`, specific name, or `None` for optional).
- `**kwargs` - Provider-specific options.

**Returns:** `Runnable` that produces `AIMessage` with `tool_calls` when tools are used.

### Parsing Tool Calls

**Response Format:**

When bound tools are called, the returned `AIMessage` contains:

```python
response = bound_model.invoke("Get the weather in Paris")

# response.tool_calls is a list of ToolCall objects:
for call in response.tool_calls:
    print(call.id)      # Unique ID for this call
    print(call.name)    # Tool name
    print(call.args)    # Dict of arguments
    print(call.type)    # Always "tool_call"
```

**Output Parsers:**

`langchain_core.output_parsers.openai_tools` provides parsers:

```python
from langchain_core.output_parsers import JsonOutputToolsParser, PydanticToolsParser

# Extract as dicts
parser = JsonOutputToolsParser()
tool_calls = parser.invoke(response)

# Extract and validate as Pydantic
parser = PydanticToolsParser(tools=[MyTool])
validated = parser.invoke(response)
```

## Structured Output

### `with_structured_output(schema, include_raw=False, **kwargs) → Runnable`

Wraps a model to enforce schema-based responses. The returned `Runnable` guarantees output matching the schema.

**Schema Types:**

- **Pydantic `BaseModel`**: Output is validated instance.
  ```python
  from pydantic import BaseModel
  
  class Answer(BaseModel):
      answer: str
      confidence: float
  
  structured = model.with_structured_output(Answer)
  response = structured.invoke("Is the Earth round?")
  assert isinstance(response, Answer)
  ```

- **TypedDict**: Output is dict conforming to schema.
  ```python
  from typing_extensions import TypedDict
  
  class Answer(TypedDict):
      answer: str
      confidence: float
  
  structured = model.with_structured_output(Answer)
  response = structured.invoke("Is the Earth round?")
  assert isinstance(response, dict)
  ```

- **JSON Schema dictionary**: Output is dict matching schema.
  ```python
  schema = {
      "type": "object",
      "properties": {
          "answer": {"type": "string"},
          "confidence": {"type": "number"}
      },
      "required": ["answer"]
  }
  
  structured = model.with_structured_output(schema)
  ```

- **OpenAI function schema**: Output is dict.

**Implementation:**

The base implementation:
1. Binds schema as a single tool with `tool_choice="any"`.
2. Invokes the bound model (tool calling guaranteed).
3. Parses tool arguments via `JsonOutputToolsParser` or `PydanticToolsParser`.
4. Returns parsed output (or `{"raw": msg, "parsed": obj, "parsing_error": e}` if `include_raw=True`).

**Parameters:**
- `schema: dict | type` - Output schema.
- `include_raw: bool` - If `True`, return `{"raw": message, "parsed": output, "parsing_error": exception}`. Otherwise return parsed output only.
- `**kwargs` - Provider-specific structured output options.

**Example:**
```python
from pydantic import BaseModel

class PersonInfo(BaseModel):
    name: str
    age: int
    hobbies: list[str]

extractor = model.with_structured_output(PersonInfo)
person = extractor.invoke("Extract info: John, age 30, likes coding and chess")

print(person.name)     # "John"
print(person.age)      # 30
print(person.hobbies)  # ["coding", "chess"]
```

## Response Metadata and AIMessage Structure

### AIMessage Fields

Generated `AIMessage` objects carry:

| Field | Type | Description |
|-------|------|-------------|
| `content` | `str \| list[ContentBlock]` | Primary response text or multimodal blocks. Empty string when only tool calls present. |
| `tool_calls` | `list[ToolCall]` | Tool invocations if model performed tool use. |
| `additional_kwargs` | `dict[str, Any]` | Provider-specific fields (e.g., `tool_calls`, `refusal`, `stop_reason`). |
| `response_metadata` | `dict[str, Any]` | Response headers, token usage, finish reason, logprobs—extracted from API response. |
| `usage_metadata` | `UsageMetadata \| None` | Normalized usage with `input_tokens`, `output_tokens`, `total_tokens`, and optional cost breakdown. |
| `id` | `str \| None` | Unique message identifier (run ID prefixed with `"lc-"`). |

### Content Blocks

Multimodal content supports:
- **text**: `{"type": "text", "text": "..."}`
- **image_url**: `{"type": "image_url", "image_url": {"url": "..."}}`
- **image**: Base64 image data (v0 format).
- **tool_call**: `{"type": "tool_call", "name": "...", "id": "...", "args": {...}}`
- **reasoning**: `{"type": "reasoning", "reasoning": "..."}`
- **audio**: Audio data blocks.
- **file**: File or document blocks.

### Output Version

The `output_version` field controls multimodal content storage:

| Value | Behavior |
|-------|----------|
| `None` / `"v0"` (default) | Provider-specific format in `content`; `content_blocks` property lazily parses to standard format. |
| `"v1"` | Standard format written to `content` directly, making `content_blocks` and `content` equivalent. |

Set globally via `LC_OUTPUT_VERSION` environment variable or per-model:
```python
model = ChatOpenAI(..., output_version="v1")
```

## Caching and Lifecycle

### Cache Lookup

Before calling `_generate()`, `_generate_with_cache()` checks the cache:

1. **Serialization**: Message inputs and call parameters are serialized into a cache key via `_get_llm_string()`.
2. **Lookup**: Query configured cache (instance, global, or none).
3. **Hit**: Return cached `ChatGeneration` objects, zero out token cost, and replay V2 events if handler attached.
4. **Miss**: Apply rate limiting and proceed to generation.

```python
from langchain_core.caches import InMemoryCache

cache = InMemoryCache()
model = ChatOpenAI(..., cache=cache)

response1 = model.invoke("Hello")  # Cache miss; calls API
response2 = model.invoke("Hello")  # Cache hit; no API call
```

### Rate Limiting

If `rate_limiter` (a `BaseRateLimiter`) is configured, `acquire()` / `aacquire()` is called before generation to enforce request pacing at the model level.

```python
from langchain_core.rate_limiters import InMemoryRateLimiter

limiter = InMemoryRateLimiter(requests_per_second=1)
model = ChatOpenAI(..., rate_limiter=limiter)
```

### Callback Lifecycle

Standard callback events fire during generation:

| Event | Fired | Handler Method |
|-------|-------|-----------------|
| `on_chat_model_start` | Before API call | `on_chat_model_start(serialized, input_list, ...)` |
| `on_llm_new_token` | Per streaming chunk (V1) | `on_llm_new_token(token, ...)` |
| `on_stream_event` | Per protocol event (V2/V3) | `on_stream_event(event, ...)` |
| `on_llm_end` | After generation | `on_llm_end(result, ...)` |
| `on_llm_error` | On exception | `on_llm_error(error, ...)` |

```python
from langchain_core.callbacks import BaseCallbackHandler

class MyCallback(BaseCallbackHandler):
    def on_chat_model_start(self, serialized, input_list, **kwargs):
        print(f"Model called with {len(input_list)} inputs")
    
    def on_llm_end(self, response, **kwargs):
        print(f"Generated {len(response.generations)} generations")

model.invoke("Hello", config={"callbacks": [MyCallback()]})
```

## Configuration and Composition

### Binding Configuration

#### `bind(**kwargs) → Runnable`

Binds runtime kwargs to the model, returning a `_ChatModelBinding` that preserves chat-model-specific overloads (e.g., `stream_events(version="v3")`).

```python
bound = model.bind(temperature=0.5, top_p=0.9)
response = bound.invoke("Be concise")
```

### Composable Methods (Declarative)

These return wrapped `Runnable` objects for declarative pipeline construction:

| Method | Returns | Purpose |
|--------|---------|---------|
| `bind_tools(...)` | `Runnable[..., AIMessage]` | Enable tool calling. |
| `with_structured_output(...)` | `Runnable[..., dict \| BaseModel]` | Enforce schema. |
| `with_retry(...)` | `Runnable` | Automatic retry on failure. |
| `with_fallbacks(...)` | `Runnable` | Fallback to alternative models. |

```python
pipeline = (
    model
    .bind_tools(tools)
    .with_retry(stop_after_attempt=3)
    .with_fallbacks([backup_model])
)
response = pipeline.invoke("Use a tool")
```

### Configurable Models

#### `configurable_fields(*fields) → Runnable`

Mark init parameters as runtime-configurable.

```python
from langchain_core.runnables import ConfigurableField

model = ChatOpenAI(
    ...
).configurable_fields(
    model=ConfigurableField(name="model"),
    temperature=ConfigurableField(name="temp")
)

response = model.invoke(
    "Hello",
    config={"configurable": {"model": "gpt-4", "temp": 0.7}}
)
```

#### `configurable_alternatives(*fields) → Runnable`

Swap models at runtime via config.

```python
model = ChatOpenAI(...).configurable_alternatives(
    default_key="openai",
    anthropic=ChatAnthropic(...)
)

response = model.invoke(
    "Hello",
    config={"configurable": {"which": "anthropic"}}
)
```

### Universal Factory: `init_chat_model()`

The `init_chat_model()` function (in `langchain_v1.langchain.chat_models.base`) provides provider-agnostic model instantiation:

```python
from langchain.chat_models import init_chat_model

# Prefixed form (provider inferred from prefix)
model = init_chat_model("openai:gpt-4")
model = init_chat_model("anthropic:claude-opus")

# Bare model name (provider inferred via heuristics)
model = init_chat_model("gpt-4")  # Inferred as OpenAI
model = init_chat_model("claude-3-sonnet")  # Inferred as Anthropic

# Explicit provider
model = init_chat_model("gpt-4", model_provider="openai")
```

**Supported Providers:**
- `openai` (gpt-4, gpt-4o, o1, etc.)
- `anthropic` (claude-3-opus, claude-3-sonnet, etc.)
- `azure_openai`, `azure_ai`
- `google_vertexai`, `google_genai`, `google_anthropic_vertex`
- `mistralai`, `groq`, `deepseek`, `xai`, `cohere`, `perplexity`, `upstage`
- `bedrock`, `bedrock_converse`, `anthropic_bedrock`
- `ollama`, `together`, `fireworks`, `huggingface`, `ibm`, `nvidia`, `litellm`, `openrouter`, `baseten`, `meta`, `langsmith`

**Configurable Models:**

```python
# All parameters configurable at runtime
model = init_chat_model(
    "openai:gpt-4",
    configurable_fields="any"
)

response = model.invoke(
    "Hello",
    config={
        "configurable": {
            "model": "gpt-4o",
            "temperature": 0.8,
            "api_key": "sk-..."
        }
    }
)
```

## Provider Integration

### Message Conversion

Providers implement message conversion to adapt `BaseMessage` to API format:

**OpenAI Example:**
```python
# LangChain message
HumanMessage(content="What's the weather?")

# Converted to OpenAI format
{"role": "user", "content": "What's the weather?"}
```

**Anthropic Example:**
```python
# LangChain message with multimodal content
HumanMessage(content=[
    {"type": "text", "text": "Describe this:"},
    {"type": "image_url", "image_url": {"url": "..."}}
])

# Converted to Anthropic format
{
    "role": "user",
    "content": [
        {"type": "text", "text": "Describe this:"},
        {"type": "image", "source": {"type": "url", "url": "..."}}
    ]
}
```

### Model Profiles

The optional `profile: ModelProfile` field exposes model capabilities:

```python
model = ChatOpenAI(...)
if model.profile and "tool_calling" in model.profile.get("abilities", []):
    bound = model.bind_tools(tools)
```

Profiles include context window sizes, supported modalities, tool calling availability, structured output support, and cost information.

```python
def _resolve_model_profile(self) -> ModelProfile | None:
    """Return model metadata or None if unavailable."""
    # Called automatically during initialization
    # Override in subclasses to load provider-specific profiles
    return None
```

### Key Provider Patterns

**All providers:**
1. Extend `BaseChatModel` (or intermediate base like `BaseChatOpenAI`).
2. Implement `_generate()` (required) and optionally `_stream()`, `_agenerate()`, `_astream()`.
3. Override `bind_tools()` to convert schemas to provider format.
4. Optionally override `with_structured_output()` for native support.
5. Implement `_resolve_model_profile()` to load model capabilities.

## Error Handling

### Generation Errors

Errors during generation are caught, formatted into `ChatGeneration` objects with error metadata, and dispatched to `on_llm_error()`:

```python
from langchain_core.callbacks import BaseCallbackHandler

class ErrorHandler(BaseCallbackHandler):
    def on_llm_error(self, error, response=None, **kwargs):
        print(f"Generation failed: {error}")
        if response and response.generations:
            print(f"Error metadata: {response.generations[0].generation_info}")

model.invoke("Hello", config={"callbacks": [ErrorHandler()]})
```

The `_generate_response_from_error()` utility extracts HTTP response, headers, and status codes from exceptions for tracing.

### Streaming Errors

Errors during streaming:
1. Merge accumulated chunks into a partial `ChatGeneration`.
2. Dispatch both chunks and error metadata to callbacks.
3. Re-raise exception.

## Performance Configuration

### Disabling Features

Optimize by disabling unused features:

```python
# Disable caching
model = ChatOpenAI(..., cache=False)

# Disable streaming entirely
model = ChatOpenAI(..., disable_streaming=True)

# Disable streaming only for tool calling
model = ChatOpenAI(..., disable_streaming="tool_calling")
```

### Async and Concurrency

Prefer async APIs for concurrent workloads:

```python
# Concurrent invocation
import asyncio

results = await asyncio.gather(
    model.ainvoke("Hello"),
    model.ainvoke("Hi"),
    model.ainvoke("Hey")
)

# Batch with concurrency control
results = await model.abatch(inputs, config={"max_concurrency": 5})
```

## Testing and Mocking

### FakeChatModel

For testing without API calls:

```python
from langchain_core.language_models.fake_chat_models import FakeChatModel

fake = FakeChatModel()
response = fake.invoke("Hello")
```

Supports tool calling, streaming, and structured output for integration testing.

## Summary

`BaseChatModel` provides a unified abstraction for conversational AI, abstracting provider differences while enabling streaming, tool calling, structured output, caching, and composable pipelines. Implementations inherit comprehensive lifecycle management, callback dispatch, and Runnable integration—requiring only `_generate()` and `_llm_type` for basic functionality, with optional performance enhancements via streaming and async overrides.
