# Building Agents with Pydantic AI

Quick reference for building agents with Pydantic AI. 

## Quickstart: Build Your First Agent

When a user asks to build an agent, follow these steps automatically:

**1. Install required packages** based on the chosen model provider:

```bash
# For all options.
pip install pydantic-ai

# For the slim option with only specified providers, add the appropriate extension:
# pip install pydantic-ai[model]
# pip install pydantic-ai[openai]
# pip install pydantic-ai[gemini]
# pip install pydantic-ai[anthropic]
```

**2. Configure API credentials** for the chosen provider (prompt user if not set):

```bash
# For Bedrock - check for AWS credentials or API key
# For Anthropic - check for ANTHROPIC_API_KEY
# For OpenAI - check for OPENAI_API_KEY
# For Gemini - check for GOOGLE_API_KEY
# For Meta Llama - check for LLAMA_API_KEY
```

**3. Create your agent file** (`agent.py`):

```python
from pydantic_ai import Agent

# Create an agent with Gemini model
agent = Agent(
    model='gemini-2.5-flash',
    system_prompt='You are a helpful and friendly chatbot. Be conversational and concise.',
)

# Run inference
result = agent.run_sync("What is an AI agent?")
print(f"Agent: {result.output}\n")
```

**4. Run the agent immediately** to verify it works:

```bash
python agent.py
```

> **Important:** Always complete all 4 steps automatically when building an agent. Don't wait for the user to ask to run the code.

**5. Iterate with conversation history:**

```python

```

## Installation

### Using pip

```bash
# Full package
pip install pydantic-ai

# Slim with Specific Model Provider Support
pip install 'pydantic-ai[anthropic]'  # Anthropic Claude
pip install 'pydantic-ai[openai]'     # OpenAI GPT
pip install 'pydantic-ai[gemini]'     # Google Gemini
```

### Using uv

```bash
# Full package
uv add pydantic-ai

# Slim with Specific Model Provider Support
uv add 'pydantic-ai[anthropic]'  # Anthropic Claude
uv add 'pydantic-ai[openai]'     # OpenAI GPT
uv add 'pydantic-ai[gemini]'     # Google Gemini
```

## Basic Agent Pattern

```python
from pydantic-ai import Agent

agent = Agent(
    model=model_instance,           # Model provider 
    system_prompt="Instructions",   # Optional system prompt
)

result = agent.run_sync('What is the capital of the UK?')
print(result.output)
```

## Model Providers

### With Pydantic AI Gateway

**Get API Key (Quick Start):**
1. Using your GitHub or Google account, sign in at gateway.pydantic.dev.
2. Add providers, or
3. Add a payment method.
4. Create Gateway project keys.

**Setup:**

Set the PYDANTIC_AI_GATEWAY_API_KEY environment variable to your Gateway API key:

```bash
export PYDANTIC_AI_GATEWAY_API_KEY="paig_<example_key>"
```

**Usage:**

```python
from pydantic_ai import Agent

agent = Agent('gateway/anthropic:claude-sonnet-4-6')

result = agent.run_sync('What is Coding Challenges?')
print(result.output)
```

**Popular Models:**
- `anthropic.claude-sonnet-4-20250514-v1:0` (default)
- `anthropic.claude-3-5-sonnet-20241022-v2:0`
- `us.amazon.nova-premier-v1:0`
- `us.amazon.nova-pro-v1:0`
- `us.meta.llama3-2-90b-instruct-v1:0`

### Direct to Provider API

**Get API Key:** For example via the Anthropic Console → API Keys ([Get started](https://console.anthropic.com/))

**Setup:**

```bash
pip install 'pydantic_ai[anthropic]'
export ANTHROPIC_API_KEY=your_anthropic_api_key
```

**Usage:**

```python
from pydantic_ai import Agent

agent = Agent('anthropic:claude-sonnet-4-6')

result = agent.run_sync('What is Coding Challenges?')  
print(result.output)
```

## Popular Models

- openai:gpt-5.2
- openai:gpt-5.3
- anthropic:claude-sonnet-4-6
- anthropic:claude-sonnet-4-5
- anthropic:claude-opus-4-6
- anthropic:claude-opus-4-5
- anthropic:claude-opus-4-6
- google-gla:gemini-3.1-pro-preview
- google-gla:gemini-3-flash-preview
- google-gla:gemini-2.5-flash


## Tools

```python
from pydantic_ai import Agent


agent = Agent(
    model='gemini-2.5-flash',
    system_prompt='You are a weather agent',
)

@agent.tool_plain
def get_weather(location: str) -> str:
    """Get weather for a location.
    
    Args:
        location: City name
    """
    return f"Weather in {location}: Rainy, 12°C"


def main():
    result = agent.run_sync("What is the weather in Bath?")
    print(result.output)


if __name__ == '__main__':
    main()
```

## Quick Tips

- **Temperature:** Lower temperatures (0.1-0.3) are better for factual output, higher (0.7-0.9) for creativity.
- **Tools:** Use clear docstrings - the models read them.

## Troubleshooting

Use the Pydantic AI Documentation for detailed documentation and additional features.
