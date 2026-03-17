# Observability

**Goal:** Instrument agents with OpenTelemetry and Pydantic Logfire to trace model calls, tool executions, retries, and errors. Use the Logfire MCP server to query traces and diagnose issues.

**Steps:**

1. Install `logfire` and configure it in the application.
2. Set `instrument=True` on agents or use `logfire.instrument_pydantic_ai()` globally.
3. Enable the Logfire MCP server in `mcp.json` for trace querying.
4. Monitor token usage, retries, errors, and latency.

## Instrumenting Agents

### Per-Agent Instrumentation

Set `instrument=True` on any agent that should emit traces:

```python
from pydantic_ai import Agent

agent = Agent(
    'gemini-2.5-flash',
    deps_type=MyDeps,
    output_type=MyOutput,
    instrument=True,
)
```

This emits OTel spans for every model call, tool execution, and retry on that agent.

### Global Instrumentation

Instrument all agents and MCP connections at once:

```python
import logfire

logfire.configure(service_name='my-agent-app')
logfire.instrument_pydantic_ai()  # all agents, globally
logfire.instrument_mcp()          # all MCP client/server calls
```

Place this at application startup, before any agent runs. The `logfire run` CLI command can also auto-detect and instrument PydanticAI automatically.

### Minimal Setup

```python
"""
Instrumented agent with Logfire.
    pip install pydantic-ai logfire
    export LOGFIRE_TOKEN='your-logfire-write-token'
"""
import logfire
from pydantic_ai import Agent

logfire.configure()

agent = Agent(
    'gemini-2.5-flash',
    instructions='You are a helpful assistant.',
    instrument=True,
)

result = agent.run_sync('What is the capital of the United Kingdom?')
print(result.output)
```

Running this sends traces to Logfire automatically. View them at https://logfire.pydantic.dev.

## What Gets Traced

PydanticAI emits spans following OpenTelemetry semantic conventions:

| Span | What it captures |
|------|-----------------|
| Agent run | Full lifecycle of a single `agent.run()` call |
| Model request | Each LLM API call with model, tokens, and latency |
| Tool call | Each tool execution with name, arguments, and result |
| Retry | Each retry attempt with the reason and retry count |
| Output validation | Schema validation of the agent's structured output |
| MCP request | Each MCP tool call when using `logfire.instrument_mcp()` |

## What to Monitor

Track these metrics to catch problems early:

| Metric | Why it matters |
|--------|---------------|
| Token usage per run | Cost control; detect prompt bloat |
| Cost per agent run | Budget tracking across models |
| Tool call frequency | Spot tools being called unnecessarily |
| Tool call latency | Find slow external dependencies |
| Retry rates | High retries mean bad prompts or tool errors |
| Output validation failures | The model is producing invalid structured output |
| End-to-end latency | User-facing response time |
| Error rates by exception type | Surface recurring failures |

## Logfire MCP Server

The Logfire MCP server gives Kiro direct access to trace data. Enable it in `mcp.json` to query exceptions, run SQL against traces, and generate links to the Logfire UI.

### Configuration

In `mcp.json`:

```json
{
  "logfire": {
    "command": "uvx",
    "args": ["logfire-mcp@latest"],
    "env": {
      "LOGFIRE_READ_TOKEN": "${LOGFIRE_READ_TOKEN}"
    },
    "disabled": true
  }
}
```

To enable: set `disabled` to `false` and provide a read token. Generate one at:
https://logfire.pydantic.dev/-/redirect/latest-project/settings/read-tokens

Store the token in an environment variable, not in the config file.

### Available Tools

The Logfire MCP server exposes these tools:

| Tool | Purpose |
|------|---------|
| `find_exceptions` | Get exception counts grouped by file |
| `find_exceptions_in_file` | Get detailed trace info for exceptions in a specific file |
| `arbitrary_query` | Run SQL queries against the OTel records table |
| `get_logfire_records_schema` | View the schema to help write queries |
| `logfire_link` | Generate a direct link to view a trace in the Logfire UI |

### Example Queries

Find recent exceptions in an agent file:

```json
{
  "name": "find_exceptions_in_file",
  "arguments": {
    "filepath": "agents/sql_analyst.py",
    "age": 60
  }
}
```

Query tool call latency over the last hour:

```json
{
  "name": "arbitrary_query",
  "arguments": {
    "query": "SELECT attributes->>'gen_ai.tool.name' as tool, avg(duration) as avg_ms FROM records WHERE span_name LIKE '%tool%' GROUP BY tool ORDER BY avg_ms DESC",
    "age": 60
  }
}
```

## Alternative Backends

PydanticAI emits standard OpenTelemetry data. Logfire is the recommended backend, but traces can be sent to any OTel-compatible system:

- Jaeger
- Datadog
- Grafana Tempo
- Honeycomb
- AWS X-Ray

Configure the OTel exporter in `logfire.configure()` or use the standard `OTEL_EXPORTER_*` environment variables.

## Best Practices

- Instrument every production agent with `instrument=True` or `logfire.instrument_pydantic_ai()`
- Call `logfire.configure()` once at application startup
- Use `service_name` to distinguish between different applications sending traces
- Tag traces with user or tenant context through deps for per-user observability
- Alert on spikes in retry rates, validation failures, or cost per run
- Use the Logfire MCP server to debug failures without leaving the IDE
- Keep the Logfire read token in environment variables, never in code or config files
- Use `logfire.instrument_mcp()` when agents consume MCP tools to get full distributed traces
