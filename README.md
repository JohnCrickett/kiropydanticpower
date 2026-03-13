# Kiro Power for Paydantic AI

There are two ways to install this power in Kiro.

**From GitHub:**
1. Open the Powers panel (click the Ghosty icon with the lightning bolt 👻⚡ in the sidebar)
2. Click "Install Custom Power" → "Import power from GitHub" and paste the repo URL (https://github.com/JohnCrickett/kiropydanticpower)
3. The power must have a valid `POWER.md` file in the repository root

**From a local folder:**
For powers you have cloned locally and install from the local path using the "Import power from a folder" option.

**Once installed:**

To use the power ask Kiro to help create an agent with Pydantic AI, for example:

## Creating a Chatbot
```
I want to create a chatbot using Pydantic AI and gemini 2.5 flash.
```

## Creating a Structure Output Agent
```
I want to build an AI agent using pydantic. The agent should handle support requests from customers and return structured data.
```

## Creating a Support Agent with Evals

```text 
I want to write an agent to handle support tickets. It should parse user request in unstructured text and generate structured output. 

Use Pydantic AI and I want evals to check it works correctly.

User tickets should have a serveity from 1 (highest) to 5 (lowest).
Tickets should have a title, category, summary and full description. The full description should be what the user provided.
```
