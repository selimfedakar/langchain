# Files

- [Agent Execution: From Request to Response](agent-execution.md) - End-to-end control flow for agent execution, middleware composition, state management, and error handling.
- [Composability: Building Modular Systems](composability.md) - Composition patterns for building complex applications from simple, reusable components. Covers Runnable piping, component interfaces, and composability boundaries.
- [Middleware: Composable Agent Behaviors](middleware.md) - Middleware architecture for composing agent behavior through lifecycle hooks and request interception. Covers interface, execution model, state management, built-in implementations, and composition patterns for retries, validation, security, and control flow.
- [Model Initialization: Universal Model Loading](model-initialization.md) - The init_chat_model factory function and provider routing system. Covers model identifier syntax, provider discovery, fallback logic, and model profile integration for loading any supported LLM.
- [Structured Output: Type-Safe Results from Language Models](structured-output.md) - Patterns for obtaining structured, validated output from language models using Pydantic schemas, JSON schemas, and output parsers. Bind schemas to chat models with with_structured_output() or parse raw model responses using BaseOutputParser implementations.
