# Use Structured Outputs

**Goal:** Get validated, structured data from your agent.

**Steps:**
1. Define structured data types that derive from `BaseModel`.
2. Create the agent and set the `output_type` to the value of the structured data type or types to use. This can be a single type or a list of types.
3. Invoke the agent as normal.

For example, consider a customer support agent.

```python
from pydantic import BaseModel, Field
from pydantic_ai import Agent

# Define your output structure
class SupportTicket(BaseModel):
    summary: str = Field(description='Brief summary of the issue.')
    description: str = Field(description="The description of the problem that the user provided.")
    priority: int = Field(description='Priority level 1-5.', ge=1, le=5)
    category: str = Field(description='Support ticket category.')

# Create agent with output type
agent = Agent(
    'gemini-2.5-flash',
    output_type=SupportTicket,
    instructions='Analyse support queries and create structured tickets.',
)

# Run and get structured output
result = agent.run_sync("My account is locked and I cannot log in! I haven't changed my password!")
print(result.output)
```

**Benefits:**
- Type-safe output.
- Automatic validation via Pydantic.
- IDE autocomplete support.
- Runtime guarantees.
