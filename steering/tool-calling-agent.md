# Tool Calling Agent 

**Goal:** Enable your agent to call tools (functions) to retrieve information or carry out actions.

**Steps:**

1. Create the agent.
2. Create any tools required.
3. Invoke the agent. 

```python
"""
Coding assistant agent with file tools.
Set your API key before running:
    export GEMINI_API_KEY='your-api-key-here'
"""
import os
from pydantic_ai import Agent, ModelRetry

INSTRUCTIONS = """
You are a helpful coding assistant.
Your job is to help the user write high quality software.
High quality software is the simplest code that meets the requirements.

Best Practices:
 - Keep functions small with a single responsibility
 - Implement proper error handling with appropriate exceptions
"""

agent = Agent(
    'gemini-2.5-flash',
    instructions=INSTRUCTIONS,
)


@agent.tool_plain
def list_files(path: str, recursive: bool = False) -> list[str]:
    """List files in a directory.

    Args:
        path: The directory path to list files from.
        recursive: If True, lists files in subdirectories too.
    """
    if not os.path.isdir(path):
        raise ModelRetry(f"'{path}' is not a valid directory. Check the path and try again.")

    if recursive:
        return [
            os.path.join(root, f)
            for root, _, files in os.walk(path)
            for f in files
        ]
    return [
        os.path.join(path, f)
        for f in os.listdir(path)
        if os.path.isfile(os.path.join(path, f))
    ]


@agent.tool_plain
def search_files(query: str, path: str, recursive: bool = False) -> list[str]:
    """Search for files containing a specific string.

    Args:
        query: The string to search for.
        path: The directory path to search in.
        recursive: If True, searches subdirectories too.
    """
    if not os.path.isdir(path):
        raise ModelRetry(f"'{path}' is not a valid directory. Check the path and try again.")

    matching = []
    for root, _, files in os.walk(path):
        for name in files:
            file_path = os.path.join(root, name)
            try:
                with open(file_path, 'r', encoding='utf-8', errors='ignore') as f:
                    if query in f.read():
                        matching.append(file_path)
            except Exception:
                continue
        if not recursive:
            break
    return matching


@agent.tool_plain
def create_file(path: str, content: str = "") -> str:
    """Create a new file with the given content.

    Args:
        path: The file path to create (e.g., "src/my_module.py").
        content: The content to write. Defaults to empty.
    """
    directory = os.path.dirname(path)
    if directory:
        os.makedirs(directory, exist_ok=True)

    with open(path, 'w', encoding='utf-8') as f:
        f.write(content)
    return f"Created {path} ({len(content)} bytes)"


@agent.tool_plain
def read_file(path: str, start_line: int = 0, end_line: int = 4000) -> str:
    """Read a file and return its contents.

    Args:
        path: The file path to read.
        start_line: First line to include (0-indexed).
        end_line: Last line to include. Capped at 4000.
    """
    end_line = min(end_line, 4000)
    try:
        with open(path, encoding='utf-8') as f:
            lines = f.readlines()[start_line:end_line]
    except FileNotFoundError:
        raise ModelRetry(f"File '{path}' not found. Check the path and try again.")
    return ''.join(lines)


def main():
    """Run the coding assistant in a chat loop."""
    print("Coding assistant started! Type 'quit' to stop.\n")

    result = agent.run_sync("Prepare to write some code with the user.")

    while True:
        prompt = input("You: ").strip()
        if prompt.lower() in ['quit', 'q', 'exit']:
            break
        if not prompt:
            continue

        result = agent.run_sync(prompt, message_history=result.new_messages())
        print(f"Agent: {result.output}\n")


if __name__ == '__main__':
    main()
```
