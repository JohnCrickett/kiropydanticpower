# Evaluating Agents

**Goal:** Systematically test that agents produce correct, reliable outputs and catch regressions when prompts, tools, or models change.

**Steps:**

1. Create a test database or dataset with known data.
2. Define eval cases with inputs, expected outputs, and evaluators.
3. Write a wrapper function that runs a single question through the agent.
4. Run the eval suite and inspect the report.

## Eval Case Categories

Structure cases around these categories:

| Category | What it tests | Example |
|----------|--------------|---------|
| Factual lookups | Retrieve a specific known value | "What is Alice's salary?" → 95000 |
| Aggregations | Calculate correctly | "Average salary?" → ~83285 |
| Cross-table | Join data from multiple sources | "Which departments are over budget?" |
| Edge cases | Handle missing data gracefully | "What is Zara's salary?" → not found |
| Refusals | Reject bad requests | Write queries are blocked |
| Ambiguity | Handle vague questions | "Tell me about the team" |

## Choosing Evaluators

- **`Contains`** — For factual answers where a specific value must appear. Fast, deterministic, no LLM cost.
- **`LLMJudge`** — When the answer could be phrased many ways but must be semantically correct. Costs a model call per case. Always provide a clear rubric.
- **`IsInstance`** — When the output type matters, not the content.
- **Custom evaluators** — For domain-specific checks (e.g., "output is valid SQL").

Prefer `Contains` where possible. Fall back to `LLMJudge` for fuzzier checks.

## Example: Evaluating the SQL Analysis Agent

```python
"""
Evals for the SQL analysis agent.
    export GEMINI_API_KEY='your-api-key-here'
    pip install pydantic-ai pydantic-evals
"""
import sqlite3
import os
from pydantic_evals import Case, Dataset
from pydantic_evals.evaluators import LLMJudge, Contains

from dataclasses import dataclass
from pydantic_ai import Agent, RunContext, ModelRetry


@dataclass
class DbDeps:
    db_path: str

    def connect(self) -> sqlite3.Connection:
        return sqlite3.connect(self.db_path)


agent = Agent(
    'gemini-2.5-flash',
    deps_type=DbDeps,
    retries=3,
    instructions="""
You are a SQL analyst assistant.
Use list_tables and describe_table to understand the schema.
Use run_query to answer questions with SELECT queries.
Limit results to 50 rows. Explain findings in plain language.
""",
)


@agent.tool
def list_tables(ctx: RunContext[DbDeps]) -> str:
    """List all tables in the database with row counts."""
    conn = ctx.deps.connect()
    try:
        tables = conn.execute(
            "SELECT name FROM sqlite_master WHERE type='table' ORDER BY name"
        ).fetchall()
        lines = []
        for (name,) in tables:
            count = conn.execute(f"SELECT COUNT(*) FROM [{name}]").fetchone()[0]
            lines.append(f"  {name} ({count} rows)")
        return "Tables:\n" + "\n".join(lines) if lines else "No tables found."
    finally:
        conn.close()


@agent.tool
def describe_table(ctx: RunContext[DbDeps], table: str) -> str:
    """Get column names and types for a table.

    Args:
        table: The table name to describe.
    """
    conn = ctx.deps.connect()
    try:
        columns = conn.execute(f"PRAGMA table_info([{table}])").fetchall()
        if not columns:
            raise ModelRetry(f"Table '{table}' not found. Use list_tables first.")
        lines = [f"  {col[1]}: {col[2]}" for col in columns]
        return f"Columns in '{table}':\n" + "\n".join(lines)
    finally:
        conn.close()


@agent.tool
def run_query(ctx: RunContext[DbDeps], sql: str) -> str:
    """Run a read-only SQL query.

    Args:
        sql: A SELECT query to execute.
    """
    if not sql.strip().upper().startswith(("SELECT", "WITH")):
        raise ModelRetry("Only SELECT queries are allowed.")
    conn = ctx.deps.connect()
    try:
        cursor = conn.execute(sql)
        columns = [d[0] for d in cursor.description] if cursor.description else []
        rows = cursor.fetchall()
        if not rows:
            return "No results."
        header = " | ".join(columns)
        body = "\n".join(" | ".join(str(v) for v in row) for row in rows[:50])
        return f"{header}\n{body}"
    except sqlite3.OperationalError as e:
        raise ModelRetry(f"SQL error: {e}. Check the query and try again.")
    finally:
        conn.close()


TEST_DB = "test_eval.db"


def create_test_db():
    """Create a test database with known data."""
    if os.path.exists(TEST_DB):
        os.remove(TEST_DB)

    conn = sqlite3.connect(TEST_DB)
    conn.executescript("""
        CREATE TABLE employees (
            id INTEGER PRIMARY KEY,
            name TEXT,
            department TEXT,
            salary INTEGER,
            start_date TEXT
        );
        INSERT INTO employees VALUES (1, 'Alice', 'Engineering', 95000, '2020-03-15');
        INSERT INTO employees VALUES (2, 'Bob', 'Engineering', 88000, '2021-07-01');
        INSERT INTO employees VALUES (3, 'Carol', 'Marketing', 72000, '2019-11-20');
        INSERT INTO employees VALUES (4, 'Dave', 'Marketing', 68000, '2022-01-10');
        INSERT INTO employees VALUES (5, 'Eve', 'Engineering', 102000, '2018-06-01');
        INSERT INTO employees VALUES (6, 'Frank', 'Sales', 77000, '2021-09-15');
        INSERT INTO employees VALUES (7, 'Grace', 'Sales', 81000, '2020-04-01');

        CREATE TABLE departments (
            name TEXT PRIMARY KEY,
            budget INTEGER,
            head TEXT
        );
        INSERT INTO departments VALUES ('Engineering', 500000, 'Eve');
        INSERT INTO departments VALUES ('Marketing', 200000, 'Carol');
        INSERT INTO departments VALUES ('Sales', 250000, 'Grace');
    """)
    conn.commit()
    conn.close()


def ask_agent(question: str) -> str:
    """Run a single question through the agent and return the output."""
    deps = DbDeps(db_path=TEST_DB)
    result = agent.run_sync(question, deps=deps)
    return result.output


dataset = Dataset(
    cases=[
        # Factual lookups
        Case(
            name="specific_salary",
            inputs="What is Alice's salary?",
            expected_output="95000",
            evaluators=[Contains(value="95000")],
        ),
        Case(
            name="count_employees",
            inputs="How many employees are there?",
            expected_output="7",
            evaluators=[Contains(value="7")],
        ),
        Case(
            name="list_departments",
            inputs="What departments exist?",
            expected_output="Engineering, Marketing, Sales",
            evaluators=[
                Contains(value="Engineering"),
                Contains(value="Marketing"),
                Contains(value="Sales"),
            ],
        ),

        # Aggregations
        Case(
            name="average_salary",
            inputs="What is the average salary across all employees?",
            expected_output="83285",
            evaluators=[
                LLMJudge(rubric="The answer should state an average salary close to 83,285 (within rounding)."),
            ],
        ),
        Case(
            name="highest_paid",
            inputs="Who earns the most?",
            expected_output="Eve with 102000",
            evaluators=[
                Contains(value="Eve"),
                Contains(value="102000"),
            ],
        ),
        Case(
            name="department_salary_totals",
            inputs="What is the total salary spend per department?",
            expected_output="Engineering: 285000, Marketing: 140000, Sales: 158000",
            evaluators=[
                Contains(value="285000"),
                Contains(value="140000"),
                Contains(value="158000"),
            ],
        ),

        # Cross-table queries
        Case(
            name="department_budget_vs_salary",
            inputs="Which departments are over budget on salaries?",
            expected_output="None — all departments are within budget",
            evaluators=[
                LLMJudge(rubric=(
                    "The answer should correctly identify that no department's total "
                    "salary exceeds its budget. Engineering: 285000 vs 500000, "
                    "Marketing: 140000 vs 200000, Sales: 158000 vs 250000."
                )),
            ],
        ),
        Case(
            name="department_heads",
            inputs="Who heads each department and what do they earn?",
            expected_output="Eve: Engineering, 102000. Carol: Marketing, 72000. Grace: Sales, 81000.",
            evaluators=[
                Contains(value="Eve"),
                Contains(value="Carol"),
                Contains(value="Grace"),
            ],
        ),

        # Edge cases
        Case(
            name="nonexistent_employee",
            inputs="What is Zara's salary?",
            expected_output="No employee named Zara found",
            evaluators=[
                LLMJudge(rubric="The answer should indicate that no employee named Zara exists in the database."),
            ],
        ),
        Case(
            name="empty_result",
            inputs="Which employees started before 2018?",
            expected_output="None",
            evaluators=[
                LLMJudge(rubric="The answer should indicate that no employees started before 2018."),
            ],
        ),
    ],
)


if __name__ == '__main__':
    create_test_db()

    print("Running SQL agent evals...\n")
    report = dataset.evaluate_sync(ask_agent)
    print(report)

    if os.path.exists(TEST_DB):
        os.remove(TEST_DB)
```

**Key Concepts:**
- `Dataset` — A collection of eval cases to run as a batch
- `Case` — A single test: input question, expected output, and evaluators to check the result
- `Contains` — Deterministic check that a substring appears in the output
- `LLMJudge` — Semantic check using an LLM with a rubric describing what correct looks like
- `evaluate_sync` — Runs all cases and produces a report

**Best Practices:**
- Write eval cases alongside the agent, not after
- Start with 5-10 cases covering the happy path, add edge cases as failures emerge
- Use real questions from actual users when possible
- Test one thing per case
- Pin test data: create the database or dataset fresh each run
- Run evals before every model or prompt change
- If a case is flaky (passes sometimes, fails sometimes), the agent needs better instructions or tools
