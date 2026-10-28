# Files

- [Callbacks: Observability and Event Streaming](callbacks.md) - Event-driven callback system for monitoring LLM execution, streaming tokens, tracing with LangSmith, and custom instrumentation through handler interfaces and manager patterns.
- [Chat Models: LLM Integration Points](chat-models.md)
- [Messages: LLM Communication Protocol](messages.md) - Message types and structures—the data contract between application code and language models. Covers base message types, role semantics, tool-related message types, and provider-specific conversions.
- [Prompts: Templating and Composability](prompts.md) - Prompt templates define message sequences and variable substitution patterns for language models. Covers template syntax, PromptValue routing, runnable integration, and few-shot learning with dynamic example selection.
- [Runnables: Core Execution Abstraction](runnables.md) - Runnable is the foundational execution abstraction in LangChain. It defines a unified, composable interface for synchronous, asynchronous, batch, and streaming execution. Every LangChain component—prompts, models, parsers, retrievers, tools—is a Runnable, enabling declarative composition and uniform control flow.
- [Tools: Agent Actions and Provider Integration](tools.md) - Structured interfaces for language models to invoke external functions, with schema generation, validation, error handling, and runtime injection of framework context.
