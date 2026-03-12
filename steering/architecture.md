# PydanticAI Architecture Principles

## Type Safety First
- Every agent MUST declare `deps_type` and `output_type` explicitly
- Use `Agent[MyDeps, MyOutput]` typing; try to avoid instantiating a bare `Agent()` with no generics
- Define output schemas as Pydantic `BaseModel` with `Field(description=...)` on every field
- Use `RunContext[MyDeps]` in all tool function signatures that need context
- Prefer `@dataclass` for dependency containers and graph state; use `BaseModel` for output schemas
- If the types don't line up, fix them before running; PydanticAI's type safety catches errors at write-time

## Agent Design Patterns
- Agents are stateless and designed to be module-level singletons (global scope)
- One agent per responsibility; don't build a mega-agent that does everything
- Use `instructions=` for persistent behavioral guidance the LLM should always follow
- Use `system_prompt=` for behavioral guidance injected at the start of every inference and to retain the system prompts used in previous requests.
- Use `@agent.system_prompt` decorator for dynamic prompts that depend on runtime deps
- Set `retries=` at the agent level for model-level retries on failure
- Set `output_retries=` separately to control how many times the agent retries on output validation failure
- Use `model_settings=` for temperature, max_tokens, and other model-specific knobs
- Always set `instrument=True` in production agents for observability
- Use `result.output`, not `result.data`

## Dependency Injection
- Define deps as `@dataclass` classes with typed fields for each service
- Common deps: database connections, HTTP clients, API keys, user context, feature flags
- Pass deps to `agent.run(prompt, deps=my_deps)` at call time
- Tools and system prompts access deps through `ctx.deps` via `RunContext`
- Never reach for module-level globals or environment variables inside tools
- For testing, swap real deps with test doubles; the injection makes this trivial

## Structured Output Design
- Use `BaseModel` for all agent outputs that feed into downstream code or APIs
- Add `Field(description=...)` to every field; the LLM uses these to understand what to produce
- Use `Literal` types and `Annotated` for constrained fields
- For multiple possible output shapes, use discriminated unions
- Set `output_retries` (defaults to agent's `retries` value) so the agent can self-correct
- Validate business rules in `@agent.output_validator` decorators
- Integer cents for money: `amount_cents: int = Field(description="Amount in cents, e.g. 1999 for $19.99")`
- Never use floats for monetary values

## Output Functions

Output functions are an alternative to `BaseModel` outputs when you need to validate,
transform, or hand off the model's response as part of ending the run.

- Set `output_type` to a callable (or list of callables) instead of a type
- The model is forced to call one of the output functions to end the run; the return value becomes `result.output`
- Output function arguments are validated by Pydantic, just like tool arguments
- Raise `ModelRetry("message")` inside the function to send the model back with a helpful error
- Output functions can optionally take `RunContext` as the first argument for access to deps and context
- Prefer output functions over `@agent.output_validator` when validation logic differs per output type — cleaner than `isinstance` branching inside a single validator
- Don't also register an output function as a `@agent.tool`; it confuses the model about which to call

Example use cases: running a SQL query and retrying on error, handing off to a sub-agent,
or post-processing text before it becomes the final output.

@agent.output_validator is still the right choice when you have one output type and need
async IO in your validation (e.g. a database EXPLAIN check).

### Output Modes

By default, PydanticAI uses tool calls to elicit structured output (Tool Output mode).
You can override this per agent using marker classes on `output_type`.

- `ToolOutput(MyModel, name="...", description="...")` — default mode; customize tool name/description
  without changing behaviour. Use when you need to disambiguate multiple output types for the model.
- `NativeOutput([Foo, Bar])` — uses the model's native structured output / JSON Schema response format.
  More reliable on models that support it, but Gemini cannot use tools at the same time as `NativeOutput`.
- `PromptedOutput([Foo, Bar])` — injects the JSON schema into the instructions and parses the plain-text
  response. Works with any model, but is the least reliable; use only when the model lacks tool calling
  or native structured output support.

Choose based on your model provider:
- OpenAI/Anthropic: `ToolOutput` (default) or `NativeOutput` both work well
- Gemini: use `ToolOutput` (default) if you also need tools in the same run; use `NativeOutput` for
  output-only agents
- Weak or local models: `PromptedOutput` as a fallback
```

---

### Small addition to **Error Handling**

Add one bullet:
```
- In output validators and output functions, check `ctx.partial_output` before running side effects or
  IO-heavy validation; during streaming these are called on every partial chunk, not just the final output

## Tool Design
- `@agent.tool` for tools that need `RunContext` (access to deps, retry info, message history)
- `@agent.tool_plain` for pure functions that only need their declared parameters
- Tool docstrings ARE the tool description sent to the LLM; write them clearly and concisely
- Parameter descriptions are extracted from docstrings; use Google-style format:
  ```python
  @agent.tool
  async def get_user(ctx: RunContext[MyDeps], user_id: str) -> UserProfile:
      """Fetch a user profile by their unique ID.

      Args:
          user_id: The unique identifier of the user to look up.
      """
  ```
- Return typed values from tools, not raw strings, when the output has structure
- Raise `ModelRetry("helpful message")` on recoverable errors; the agent retries with your message as context
- Never return error strings like `"Error: user not found"`; that confuses the LLM
- Keep tools focused: one clear action per tool
- Avoid tools with more than 5 parameters; decompose complex inputs into a Pydantic model
- Use `prepare` functions on tools if you need to conditionally include/exclude tools per run

## When to Use Multiple Agents
- Start with a single agent. Only split into multiple agents when you have a clear reason.
- A single agent with good tools handles most use cases. Don't add complexity for its own sake.
- Split into multiple agents when:
  - **Different responsibilities need different instructions.** A code reviewer and a code fixer need fundamentally different system prompts. Cramming both into one agent dilutes the quality of each.
  - **You need different models per task.** Use a strong model for complex reasoning (reviewing code) and a cheaper model for straightforward work (formatting a report). Multi-agent lets you mix models.
  - **You want independent testing and evals.** If you can't write a clear eval for a single agent because it does too many things, that's a sign to split. Each agent should be independently testable.
  - **You want reusable components.** If your reviewer agent is useful in other workflows beyond the current pipeline, make it a standalone agent that gets called via delegation.
  - **Context isolation matters.** Each agent gets its own conversation context. This prevents a long tool-calling history from one task polluting the context window for another task.
- Don't split when:
  - A single agent with 3-5 tools can handle the job
  - The "agents" would just be passing data through with no real reasoning at each step
  - You're splitting because it feels architecturally clean but each agent only has one tool

## Multi-Agent Patterns
PydanticAI supports three patterns. Choose the simplest one that works.

### Agent Delegation
One agent calls another from within a tool. The delegate runs, returns, and the caller continues.
- Best for: pipelines where Agent A needs Agent B's output to continue reasoning
- The orchestrator agent owns the tools that call delegate agents
- Always pass `ctx.usage` to the delegate: `result = await agent_b.run(prompt, usage=ctx.usage, deps=...)`
- The delegate can use a different model than the caller
- Share state between agents through deps, not by stuffing everything into the prompt
- Example: code review pipeline (orchestrator → reviewer → fixer → summariser)

### Programmatic Hand-off
Application code runs agents sequentially. No agent calls another directly.
- Best for: workflows where deterministic logic or a human decides what runs next
- Pass `message_history` between runs to maintain conversation context
- Agents don't need matching `deps_type` in this pattern
- Simpler to debug because the flow is explicit Python code, not emergent LLM behaviour
- Example: triage agent classifies a ticket, then application code routes to the right specialist

### Graph-Based Control Flow
Use `pydantic-graph` for state machines with branching, looping, or conditional transitions.
- Best for: complex workflows with 3+ agents and non-linear paths
- Define each agent call as a graph node
- State flows through `GraphRunContext`
- Supports persistence for long-running or interruptible workflows
- Generates Mermaid diagrams for documentation
- Example: data pipeline with validate → transform → enrich → report, with retry loops on validation failure

### Choosing a Pattern
- One agent can handle it → don't use multi-agent
- Linear pipeline, agents need each other's output → delegation
- Branching logic decided by code or a human → programmatic hand-off
- Complex state machine with loops or conditional paths → graph
- When in doubt, start with delegation. You can always refactor to a graph later.

## Conversation Management
- A single `agent.run()` call can involve multiple model request/response cycles (tool calls)
- Use `message_history=result.new_messages()` to chain runs in a multi-turn conversation
- For new conversations, omit `message_history` to start fresh
- Prune old messages from history to manage token usage in long conversations
- Store message history externally (database, Redis) for persistence across sessions

## Graph Workflows
- Define graph state as a `@dataclass` with typed fields for all shared data
- Each node is a `@dataclass` inheriting from `BaseNode[StateT]`
- The `run()` method's return type annotation defines which nodes can come next (edges)
- Return `End[T]` to signal the graph is done with a final result
- Use `GraphRunContext[StateT]` to access and mutate state inside nodes
- Use state persistence (`FullStatePersistence`) for interruptible or long-running workflows
- Generate Mermaid diagrams with `graph.mermaid_code()` for documentation and debugging
- For simpler graphs, the beta `GraphBuilder` API uses `@g.step` decorators instead of classes

## Error Handling
- `ModelRetry` in tools: signals a recoverable error; the LLM gets your message and retries
- `UnexpectedModelBehavior`: raised when the model produces invalid responses beyond retry limits
- Agent-level `retries` controls model-level retries (e.g., bad JSON, refused responses)
- `output_retries` controls output validation retries (e.g., invalid field values)
- Always provide helpful, actionable retry messages: "User ID must be a UUID, got '{value}'"
- Use `usage_limits` to cap runaway agents: set `request_limit`, `request_tokens_limit`, `total_tokens_limit`

## Evals Strategy
- Create an eval dataset for every agent, alongside the agent code, not as an afterthought
- Use `pydantic_evals.Dataset` with `Case` objects defining inputs and expected outputs
- Store datasets as YAML files in `evals/datasets/` for version control
- Use built-in evaluators for deterministic checks (exact match, contains, schema valid)
- Use `LLMJudge` evaluators for subjective quality assessment
- Run evals in CI; set an accuracy threshold and fail the build if it drops
- Track eval scores over time with Logfire integration
- Test edge cases: empty inputs, malformed data, adversarial prompts, very long inputs

## Testing Strategy
- Use `TestModel` for fast, deterministic, zero-cost unit tests
- Use `FunctionModel` when you need custom response logic in tests
- Test tool functions independently (they're just Python async functions)
- Test output models with direct Pydantic instantiation and validation
- Mock deps using standard `unittest.mock` or `pytest` fixtures
- Reserve actual LLM calls for eval runs, not unit tests
- Test multi-agent flows end-to-end with `TestModel` to verify delegation/hand-off wiring

## Observability
- Set `instrument=True` on every production agent (or configure globally)
- PydanticAI emits OpenTelemetry spans for model calls, tool executions, and retries
- Use Pydantic Logfire for the best integrated experience, or any OTel-compatible backend
- Monitor: token usage, cost per run, tool call frequency, retry rates, validation failures, latency
- Tag traces with user/tenant context via deps for per-user observability
- Alert on anomalies: sudden spikes in retries, cost per run, or validation failure rates

## Deployment
- Expose agents as FastAPI endpoints for HTTP access
- Use `agent.run_stream()` with SSE for real-time streaming to chat UIs
- AG-UI and Vercel AI SDK event streams are supported for frontend integration
- Use `fasta2a` to expose agents as A2A-compatible services
- For long-running workflows, use durable execution (Temporal, DBOS, or Prefect)
- Pin model versions in production configs; don't rely on aliases like "latest"
- Set `usage_limits` in production to prevent runaway costs

## Security
- Never put API keys in agent instructions or system prompts
- Pass secrets through deps, sourced from environment variables or a secrets manager
- Use human-in-the-loop approval for tools that modify external state (payments, database writes, emails)
- Validate and sanitize all user inputs before passing them as agent prompts
- Use `allowed_domains` on `WebFetchTool` to prevent SSRF-style attacks
- Log all tool calls with audit context for compliance
- Rate limit agent runs per user/tenant in your API layer

## Dependency Management
- Core: `pydantic-ai` (includes `pydantic-graph`)
- Slim: `pydantic-ai-slim[openai,anthropic,...]` for specific model providers only
- Evals: `pydantic-evals` for the eval framework
- Observability: `logfire` for Pydantic Logfire integration
- Pin versions in `pyproject.toml` or `requirements.txt`
- Use `pydantic-ai-slim` in production to minimize dependency footprint
