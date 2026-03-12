---
name: "pydanticai"
displayName: "Build an agent with Pydantic AI"
description: "Build AI agents with Pydantic AI using Bedrock, Anthropic, OpenAI, Gemini, or Llama models"
keywords: ["agents", "ai", "llm", "bedrock", "anthropic", "openai", "gemini", "pydantic", "pydanticai", "tools"]
author: "John Crickett"
---

# Pydantic AI Agent Power

## Onboarding

### Prerequisites

- Python 3.10 or higher
- Basic understanding of async/await in Python
- Familiarity with type hints (recommended but not required)
- API keys for your chosen LLM provider (OpenAI, Anthropic, Google, etc.)

### Installation

#### Standard Installation

Before installing any dependencies check whether the user is using virtualenv or uv. To detect uv use within the project, look for a uv.lock file. If there is not uv.lock look for a directory matching the pattern *venv. If in doubt ask the user.

If the project uses uv install PydanticAI as follows:

```bash
uv add pydantic-ai
```

Otherwise use `pip`:

```bash
pip install pydantic-ai
```

#### Slim Installation (Specific Models)

Note if the user knows which model they're going to use and wants to avoid installing extra, unneeded packages, you can use the pydantic-ai-slim package. Details of the options are provided here: https://ai.pydantic.dev/install/#slim-install 

Essentially the commands become: `uv add pydantic-ai[model]` where model is replaced with the desired model, so for bedrock it becomes" `uv add pydantic-ai[bedrock]`.

### Basic Configuration

Set up your API key as an environment variable:

```bash
# For OpenAI
export OPENAI_API_KEY='your-api-key-here'

# For Anthropic
export ANTHROPIC_API_KEY='your-api-key-here'

# For Google
export GEMINI_API_KEY='your-api-key-here'
```

### Verification

Create a simple test file to verify installation:

```python
from pydantic_ai import Agent

agent = Agent('openai:gpt-4')
result = agent.run_sync('Hello, world!')
print(result.output)
```

Run it (if using Python directly):
```bash
python test_agent.py
```

Run it (if using uv):
```bash
uv run test_agent.py
```

If you see a response from the model, you're ready to go!

## Overview

The PydanticAI Agent Builder power provides tools, patterns, and best practices for building robust AI agents using the PydanticAI framework. It covers everything from single-agent applications to complex multi-agent workflows, with an emphasis on type safety, testability, and production readiness.

PydanticAI brings the "FastAPI feeling" to agent development. This power steers Kiro to use the framework correctly from day one, so you don't have to iterate through docs and trial-and-error yourself.


## Available Steering Files

Always read the architecture steering file before writing any PydanticAI code:
Call action "readSteering" with powerName="pydanticai-builder", steeringFile="architecture.md"

-  **architecture.md** (read always) — Core principles: type safety, agent design, dependency injection, structured outputs, tool design, multi-agent patterns, error handling, evals, observability, deployment, and security.
- **getting-started** - Complete guide for building agents with Pydantic AI, covering: installation, tools, and troubleshooting.
- **basic-chatbot** - A guide to building a basic chatbot.
- **data-analysis-agent** - A guide to building a data analysys agent.
- **tool-calling-agent** - A guide to building an agent that makes tool calls.
- **multi-agent** - A guide to building a multi-agent solution.
- **evals** — How to evaluate your agents with pydantic-evals, with a complete worked example against the SQL agent

## MCP Servers

- **fetch**: HTTP requests for external API integration
- **logfire**: exposes your OpenTelemetry data in Logfire

# Pydantic AI Best Practices

## Schema & Type Safety
### Do:
- Define a `BaseModel` output type for every agent so the LLM is contractually bound to return structured, validated data.
- Nest models and use typed lists for complex outputs.
- Use `Field(description=...)` annotations to give the LLM clear hints about what each field should contain.
- Use `output_type` instead of the removed `result_type` for structured output

### Don't:
- Accept raw string or `dict` outputs from agents in production code.
- Reuse one flat model for multiple unrelated agent outputs.

## Dependency Injection
### Do:
- Use `deps_type` and `RunContext` to pass database connections, config, and business logic into agents and tools.
- Swap real dependencies for mocks in tests and evals using the same injection system.

### Don't:
- Rely on globals or module-level state inside tool functions.
- Hardcode external connections directly inside tools.

## Tools
### Do:
- Use `@agent.tool` or `@agent.tool_plain` decorators and let Pydantic AI auto-generate JSON schemas from your type hints and docstrings.
- Keep tools focused and single-purpose, with clear, descriptive names and docstrings.

### Don't:
- Overload a single tool with multiple responsibilities; the LLM picks tools by description, so vague tools cause poor routing.
- Skip docstrings; they directly influence how the model decides to use the tool.

## Validation
### Do:
- Validate at every stage: request payloads, retrieved documents, prompt inputs, and LLM outputs.
- Fail fast; stop downstream work as soon as an early validation fails.
- Pass validated objects downstream rather than re-parsing raw data.

### Don't:
- Validate only at the final output stage.
- Parse the same payload more than once.

## Observability
### Do:
- Integrate with Pydantic Logfire or any OTel-compatible platform for tracing, debugging, and cost tracking.
- Track validation error rates per model and prompt version; frequent failures in specific fields signal schema or prompt issues.

### Don't:
- Log entire payloads if compliance or cost is a concern; log summaries instead.
- Skip tracing in production; LLM behaviour is harder to debug without it.

## Testing & Evals
### Do:
- Use Pydantic Evals to build datasets, run evaluations, and track model performance over time.
- Aim for high test coverage, especially around validation logic and tool outputs.

### Don't:
- Test only the happy path; LLMs will eventually return unexpected structures.
- Treat evals as a one-time exercise; run them continuously as models and prompts evolve.

## Agent Design
### Do:
- Use Human-in-the-Loop tool approval for actions that are expensive or irreversible.
- Use durable execution (e.g. Temporal integration) for long-running workflows so agents resume after failures.
- Treat agent development like regular software engineering; standard principles still apply.

### Don't:
- Let agents take autonomous, high-stakes actions without a confirmation step.
- Assume an agent will always complete in a single run for complex, multi-step workflows.

## Model Agnosticism
### Do:
- Keep agent code model-agnostic; Pydantic AI supports OpenAI, Anthropic, Gemini, and many others without rewriting your logic.
- Abstract model selection behind config so you can swap providers without touching agent code.

### Don't:
- Hardcode provider-specific behaviour or response formats into your agents.
- Couple your business logic to a single model's quirks.
