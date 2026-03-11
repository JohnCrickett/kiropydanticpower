# Create a Basic Agent

**Goal:** Build your first AI agent with simple instructions

**Steps:**

1. Import pydantic_ai.
2. Create the Agent instance.
3. Define the core logic of a chat bot, a loop, some input and invoking the inference.

**Example:**

```python
"""
Simple chatbot using Pydantic AI with Google Gemini.

Before running, set your API key:
    export GEMINI_API_KEY='your-api-key-here'
"""

from pydantic_ai import Agent

# Create an agent with Gemini model
agent = Agent(
    'gemini-2.5-flash',
    system_prompt='You are a helpful and friendly chatbot. Be conversational and concise.',
)


def main():
    """Run the chatbot in a loop."""
    print("Chatbot started! Type 'quit' or 'exit' to stop.\n")
    
    while True:
        user_input = input("You: ").strip()
        
        if user_input.lower() in ['quit', 'exit']:
            print("Goodbye!")
            break
        
        if not user_input:
            continue
        
        try:
            result = agent.run_sync(user_input)
            print(f"Bot: {result.output}\n")
        except Exception as e:
            print(f"Error: {e}\n")


if __name__ == '__main__':
    main()
```

**Key Concepts:**
- `Agent()` - Creates a new agent instance
- `instructions` - Static instructions that guide the agent's behavior
- `run_sync()` - Runs the agent synchronously and returns a result
- `result.output` - The final output from the agent
