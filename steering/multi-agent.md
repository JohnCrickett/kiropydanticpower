# Create a Multi-Agent System

**Goal:** Build a multi-agent system where specialized agents delegate work to each other. For example to review code, fix issues, and summarise changes.

**Steps:**

1. Define output models for structured results at each stage.
2. Create the different agents
3. Wire them together using agent delegation (tools that call other agents).
4. Invoke the pipeline with a chat loop.

## Example Code Review Multi-Agent System

```python
import os
from dataclasses import dataclass, field
from pydantic import BaseModel, Field
from pydantic_ai import Agent, RunContext, ModelRetry


class Issue(BaseModel):
    line: int = Field(description="Line number where the issue occurs")
    severity: str = Field(description="One of: error, warning, suggestion")
    description: str = Field(description="What the problem is")
    fix_hint: str = Field(description="Brief suggestion on how to fix it")


class ReviewResult(BaseModel):
    filepath: str = Field(description="Path to the reviewed file")
    issues: list[Issue] = Field(description="List of issues found")
    overall: str = Field(description="One-paragraph summary of code quality")


class FixResult(BaseModel):
    filepath: str = Field(description="Path to the fixed file")
    code: str = Field(description="The complete corrected file contents")
    changes: list[str] = Field(description="List of changes made, one per issue fixed")


class ReviewReport(BaseModel):
    filepath: str = Field(description="Path to the reviewed file")
    issues_found: int = Field(description="Number of issues the reviewer found")
    issues_fixed: int = Field(description="Number of issues the fixer addressed")
    summary: str = Field(description="Plain-English summary of the review and fixes")
    fixed_code: str = Field(description="The final corrected code")


@dataclass
class PipelineDeps:
    workspace: str
    review: ReviewResult | None = field(default=None)
    fix: FixResult | None = field(default=None)


reviewer_agent = Agent(
    'gemini-2.5-flash',
    deps_type=PipelineDeps,
    output_type=ReviewResult,
    instructions="""
You are a senior code reviewer.
Read the file and identify issues: bugs, security problems, style violations,
missing error handling, and performance concerns.

Be specific: include line numbers, severity, and a hint for how to fix each issue.
If the code is clean, return an empty issues list and say so in the overall summary.
""",
)

fixer_agent = Agent(
    'gemini-2.5-flash',
    deps_type=PipelineDeps,
    output_type=FixResult,
    instructions="""
You are a code fixer.
You receive a file and a list of review issues.
Fix every issue you can. Keep changes minimal and don't refactor beyond what's needed.
Return the complete corrected file and a list of what you changed.
""",
)

summariser_agent = Agent(
    'gemini-2.5-flash',
    deps_type=PipelineDeps,
    output_type=ReviewReport,
    instructions="""
You are a technical writer.
You receive a code review and the fixes that were applied.
Write a clear, concise summary a developer can read in 30 seconds.
Include: what was found, what was fixed, and the final code.
""",
)


@reviewer_agent.tool
def read_file(ctx: RunContext[PipelineDeps], filepath: str) -> str:
    """Read a file from the workspace for review.

    Args:
        filepath: Path relative to the workspace root.
    """
    full_path = os.path.join(ctx.deps.workspace, filepath)
    if not os.path.isfile(full_path):
        raise ModelRetry(f"File '{filepath}' not found in {ctx.deps.workspace}.")

    with open(full_path, encoding='utf-8') as f:
        lines = f.readlines()

    numbered = [f"{i+1}: {line}" for i, line in enumerate(lines)]
    return "".join(numbered)


orchestrator = Agent(
    'gemini-2.5-flash',
    deps_type=PipelineDeps,
    output_type=ReviewReport,
    retries=2,
    instructions="""
You are a code review orchestrator.
When asked to review a file:
 1. Call review_file to get issues
 2. Call fix_issues to get corrected code
 3. Call summarise_review to produce the final report
Always run all three steps. Return the final ReviewReport.
""",
)


@orchestrator.tool
async def review_file(ctx: RunContext[PipelineDeps], filepath: str) -> str:
    """Run the code reviewer on a file.

    Args:
        filepath: Path relative to the workspace root.
    """
    result = await reviewer_agent.run(
        f"Review the file: {filepath}",
        deps=ctx.deps,
        usage=ctx.usage,
    )
    ctx.deps.review = result.output
    issues = [f"  [{i.severity}] Line {i.line}: {i.description}" for i in result.output.issues]
    return f"Review complete. {len(result.output.issues)} issues found:\n" + "\n".join(issues)


@orchestrator.tool
async def fix_issues(ctx: RunContext[PipelineDeps], filepath: str) -> str:
    """Run the code fixer using the review results.

    Args:
        filepath: Path relative to the workspace root.
    """
    if not ctx.deps.review:
        raise ModelRetry("No review results yet. Call review_file first.")

    review_summary = "\n".join(
        f"Line {i.line} [{i.severity}]: {i.description} — Hint: {i.fix_hint}"
        for i in ctx.deps.review.issues
    )

    result = await fixer_agent.run(
        f"Fix the issues in {filepath}:\n{review_summary}",
        deps=ctx.deps,
        usage=ctx.usage,
    )
    ctx.deps.fix = result.output
    return f"Fixed {len(result.output.changes)} issues:\n" + "\n".join(f"  - {c}" for c in result.output.changes)


@orchestrator.tool
async def summarise_review(ctx: RunContext[PipelineDeps]) -> str:
    """Produce a final summary report from the review and fixes."""
    if not ctx.deps.review or not ctx.deps.fix:
        raise ModelRetry("Need both review and fix results first. Run review_file and fix_issues.")

    review_text = ctx.deps.review.model_dump_json()
    fix_text = ctx.deps.fix.model_dump_json()

    result = await summariser_agent.run(
        f"Summarise this code review.\nReview:\n{review_text}\nFixes:\n{fix_text}",
        deps=ctx.deps,
        usage=ctx.usage,
    )
    return result.output.model_dump_json()


def main():
    """Run the code review pipeline."""
    import sys

    workspace = sys.argv[1] if len(sys.argv) > 1 else "."
    deps = PipelineDeps(workspace=workspace)

    print(f"Code review pipeline ready. Workspace: {workspace}")
    print("Try: 'Review main.py' or 'What issues are in utils/helpers.py?'")
    print("Type 'quit' to stop.\n")

    result = None
    while True:
        prompt = input("You: ").strip()
        if prompt.lower() in ['quit', 'q', 'exit']:
            break
        if prompt.lower() == 'reset':
            deps.review = None
            deps.fix = None
            print("Pipeline state cleared.\n")
        if not prompt:
            continue


        history = result.new_messages() if result else []
        result = orchestrator.run_sync(prompt, deps=deps, message_history=history)

        report = result.output
        print(f"\n{'='*60}")
        print(f"Review Report: {report.filepath}")
        print(f"{'='*60}")
        print(f"Issues found: {report.issues_found}")
        print(f"Issues fixed: {report.issues_fixed}")
        print(f"\n{report.summary}")
        print(f"\n--- Fixed Code ---")
        print(report.fixed_code)
        print(f"{'='*60}\n")


if __name__ == '__main__':
    main()
```

**Multi-Agent Pattern: Delegation**

The orchestrator agent calls three specialist agents through its tools:

```
User
 └─→ Orchestrator
      ├─→ review_file  ──→ Reviewer Agent (reads code, finds issues)
      ├─→ fix_issues   ──→ Fixer Agent (applies corrections)
      └─→ summarise    ──→ Summariser Agent (writes the report)
```

Each specialist has its own `output_type`, `instructions`, and focus. The orchestrator
decides the order and passes context between them via `PipelineDeps`.

**Key Concepts:**
- **Agent delegation** — one agent calls another from within a tool using `await agent.run()`
- **Shared deps** — `PipelineDeps` carries state (review results, fix results) between stages
- **Usage rollup** — passing `usage=ctx.usage` ensures token counts from all agents are tracked together
- **Structured outputs** — each agent returns a typed `BaseModel`, not free text
- **Pipeline state reset** — deps are cleared each review so results don't bleed between runs

**Why three agents instead of one?**

A single agent could review, fix, and summarise. But splitting the work gives you:
- **Better focus** — each agent's instructions are short and specific
- **Independent testing** — you can eval the reviewer without running the full pipeline
- **Model flexibility** — use a cheaper model for summarisation, a stronger one for review
- **Reusability** — the reviewer agent can be used standalone in other workflows

**Alternative Patterns:**

| Pattern | When to use |
|---------|------------|
| Delegation (this example) | One agent needs another's output to continue |
| Programmatic hand-off | Deterministic logic or a human decides what runs next |
| Graph-based | 3+ agents with conditional branching or loops |
