# Data Analysis Agent

**Goal:** Build an agent that can load, explore, and analyse data files using pandas.

**Steps:**

1. Create the agent with analysis-focused instructions, that can parse CSV or SQL.
2. Add tools for loading and inspecting data format (CSV) or for inspecting schema and running queries (SQL).
3. Invoke the agent with a chat loop.

## Example Using Pandas To Parse CSV

```python
import os
import pandas as pd
from pydantic_ai import Agent, ModelRetry

# Shared state for loaded dataframes
datasets: dict[str, pd.DataFrame] = {}

INSTRUCTIONS = """
You are a data analyst assistant.
Your job is to help the user explore, understand, and analyse their data.

Workflow:
 1. Load a dataset with load_csv
 2. Inspect it with describe_data or get_columns
 3. Answer questions using query_data

query_data examples:
 - Filter rows: query("age > 30")
 - Get a value: query("name == 'John'")["age"].iloc[0]
 - Group and aggregate: groupby("region")["sales"].sum()
 - Sort: sort_values("age", ascending=False).head(10)
 - Count unique: nunique()
 - Column stats: ["age"].mean()

Always explain your findings in plain language.
When presenting numbers, round to 2 decimal places.
"""

agent = Agent(
    'gemini-2.5-flash',
    instructions=INSTRUCTIONS,
)


@agent.tool_plain
def load_csv(path: str, name: str = "default") -> str:
    """Load a CSV file into memory for analysis.

    Args:
        path: Path to the CSV file.
        name: A short name to refer to this dataset later.
    """
    if not os.path.isfile(path):
        raise ModelRetry(f"File '{path}' not found. Check the path and try again.")

    try:
        df = pd.read_csv(path)
    except Exception as e:
        raise ModelRetry(f"Could not read '{path}' as CSV: {e}")

    datasets[name] = df
    return f"Loaded '{name}' from {path} — {len(df)} rows, {len(df.columns)} columns: {', '.join(df.columns)}"


@agent.tool_plain
def get_columns(name: str = "default") -> str:
    """Get column names and data types for a loaded dataset.

    Args:
        name: The dataset name used when loading.
    """
    if name not in datasets:
        raise ModelRetry(f"No dataset named '{name}'. Load one first with load_csv.")

    df = datasets[name]
    info = [f"  {col}: {dtype}" for col, dtype in df.dtypes.items()]
    return f"Columns in '{name}' ({len(df)} rows):\n" + "\n".join(info)


@agent.tool_plain
def describe_data(name: str = "default", columns: list[str] | None = None) -> str:
    """Get summary statistics for a dataset.

    Args:
        name: The dataset name used when loading.
        columns: Specific columns to describe. If None, describes all numeric columns.
    """
    if name not in datasets:
        raise ModelRetry(f"No dataset named '{name}'. Load one first with load_csv.")

    df = datasets[name]
    if columns:
        missing = [c for c in columns if c not in df.columns]
        if missing:
            raise ModelRetry(f"Columns not found: {missing}. Available: {list(df.columns)}")
        df = df[columns]

    return df.describe(include='all').round(2).to_string()


@agent.tool_plain(retries=3)
def query_data(name: str, expression: str) -> str:
    """Run a pandas query expression on a dataset and return the result.

    Args:
        name: The dataset name used when loading.
        expression: A pandas expression chained on the dataframe
            (e.g., "query(\"name == 'John'\")[\"age\"].iloc[0]",
            "groupby('region')['sales'].sum()",
            "describe()").
    """
    if name not in datasets:
        raise ModelRetry(f"No dataset named '{name}'. Load one first with load_csv.")

    df = datasets[name]
    try:
        result = eval(f"df.{expression}", {"df": df, "pd": pd})
    except Exception as e:
        raise ModelRetry(
            f"Expression `df.{expression}` failed: {e}. "
            f"Columns available: {list(df.columns)}. "
            f"Examples: query(\"name == 'John'\")[\"age\"].iloc[0], "
            f"groupby('col')['val'].sum(), sort_values('col').head(10)"
        )

    if isinstance(result, pd.DataFrame):
        if len(result) > 50:
            return f"Result: {len(result)} rows (showing first 20):\n{result.head(20).to_string()}"
        return result.to_string()
    if isinstance(result, pd.Series):
        return result.to_string()
    return str(result)


def main():
    """Run the data analyst in a chat loop."""
    print("Data analyst started! Type 'quit' to stop.\n")

    result = None
    while True:
        prompt = input("You: ").strip()
        if prompt.lower() in ['quit', 'q', 'exit']:
            break
        if not prompt:
            continue

        history = result.new_messages() if result else []
        result = agent.run_sync(prompt, message_history=history)
        print(f"Analyst: {result.output}\n")


if __name__ == '__main__':
    main()
```

**Key Concepts:**
- `ModelRetry` - Tells the agent what went wrong so it can try a different approach
- `tool_plain` - Tools here don't need `RunContext` since datasets are in module-level state
- `message_history` - Carries conversation context so the agent remembers which datasets are loaded

**Tool Summary:**
| Tool | Purpose |
|------|---------|
| `load_csv` | Load a CSV file into memory |
| `get_columns` | Inspect column names and types |
| `describe_data` | Get summary statistics |
| `query_data` | Run pandas expressions on the data |


## SQL Analysis Agent

```python
"""
SQL analysis agent using Pydantic AI with SQLite.
Set your API key before running:
    export GEMINI_API_KEY='your-api-key-here'
"""
import sqlite3
from dataclasses import dataclass
from pydantic_ai import Agent, RunContext, ModelRetry


@dataclass
class DbDeps:
    db_path: str

    def connect(self) -> sqlite3.Connection:
        return sqlite3.connect(self.db_path)


INSTRUCTIONS = """
You are a SQL analyst assistant.
Your job is to help the user explore and query their SQLite database.

Workflow:
 1. Start with list_tables to see what's available
 2. Use describe_table to understand columns and types
 3. Use run_query to answer questions with SQL

Rules:
 - Always use read-only queries (SELECT only)
 - Limit results to 50 rows unless the user asks for more
 - Explain your SQL and findings in plain language
"""

agent = Agent(
    'gemini-2.5-flash',
    deps_type=DbDeps,
    instructions=INSTRUCTIONS,
)


@agent.tool
def list_tables(ctx: RunContext[DbDeps]) -> str:
    """List all tables in the database.

    Returns table names and their row counts.
    """
    conn = ctx.deps.connect()
    try:
        cursor = conn.execute(
            "SELECT name FROM sqlite_master WHERE type='table' ORDER BY name"
        )
        tables = [row[0] for row in cursor.fetchall()]

        if not tables:
            return "Database has no tables."

        lines = []
        for table in tables:
            count = conn.execute(f"SELECT COUNT(*) FROM [{table}]").fetchone()[0]
            lines.append(f"  {table} ({count} rows)")
        return "Tables:\n" + "\n".join(lines)
    finally:
        conn.close()


@agent.tool
def describe_table(ctx: RunContext[DbDeps], table: str) -> str:
    """Get column names, types, and a sample row for a table.

    Args:
        table: The table name to describe.
    """
    conn = ctx.deps.connect()
    try:
        cursor = conn.execute(f"PRAGMA table_info([{table}])")
        columns = cursor.fetchall()

        if not columns:
            raise ModelRetry(f"Table '{table}' not found. Use list_tables to see available tables.")

        lines = [f"  {col[1]}: {col[2]}{'  (PK)' if col[5] else ''}" for col in columns]
        info = f"Columns in '{table}':\n" + "\n".join(lines)

        sample = conn.execute(f"SELECT * FROM [{table}] LIMIT 3").fetchall()
        if sample:
            col_names = [col[1] for col in columns]
            sample_lines = [str(dict(zip(col_names, row))) for row in sample]
            info += "\n\nSample rows:\n" + "\n".join(sample_lines)

        return info
    finally:
        conn.close()


@agent.tool
def run_query(ctx: RunContext[DbDeps], sql: str) -> str:
    """Run a read-only SQL query and return the results.

    Args:
        sql: A SELECT query to execute. Must be read-only.
    """
    stripped = sql.strip().upper()
    if not stripped.startswith("SELECT") and not stripped.startswith("WITH"):
        raise ModelRetry("Only SELECT and WITH (CTE) queries are allowed. Rewrite as a read-only query.")

    conn = ctx.deps.connect()
    try:
        cursor = conn.execute(sql)
        columns = [desc[0] for desc in cursor.description] if cursor.description else []
        rows = cursor.fetchall()

        if not rows:
            return "Query returned no results."

        header = " | ".join(columns)
        separator = "-|-".join("-" * len(c) for c in columns)
        body = "\n".join(" | ".join(str(v) for v in row) for row in rows[:50])

        result = f"{header}\n{separator}\n{body}"
        if len(rows) > 50:
            result += f"\n... ({len(rows)} total rows, showing first 50)"
        return result
    except sqlite3.OperationalError as e:
        raise ModelRetry(f"SQL error: {e}. Check your query syntax and try again.")
    finally:
        conn.close()


def main():
    """Run the SQL analyst in a chat loop."""
    import sys

    db_path = sys.argv[1] if len(sys.argv) > 1 else "data.db"
    deps = DbDeps(db_path=db_path)

    print(f"SQL analyst connected to {db_path}. Type 'quit' to stop.\n")

    result = None
    while True:
        prompt = input("You: ").strip()
        if prompt.lower() in ['quit', 'q', 'exit']:
            break
        if not prompt:
            continue

        history = result.new_messages() if result else []
        result = agent.run_sync(prompt, deps=deps, message_history=history)
        print(f"Analyst: {result.output}\n")


if __name__ == '__main__':
    main()
```

**Key Concepts:**
- `deps_type=DbDeps` - The database path is injected as a dependency, not hardcoded
- `RunContext[DbDeps]` - Tools access the database connection through `ctx.deps`
- `ModelRetry` - Guides the agent to fix bad SQL or wrong table names
- Read-only enforcement - `run_query` rejects anything that isn't a SELECT or WITH

**Tool Summary:**
| Tool | Purpose |
|------|---------|
| `list_tables` | Show all tables and row counts |
| `describe_table` | Inspect columns, types, and sample data |
| `run_query` | Execute read-only SQL and return results |

**Why dependency injection here?**

The CSV example used module-level state because there was no external resource to manage. This example uses `deps_type` because the database path is a runtime concern. Different users might point at different databases. Deps make that clean:

```python
# Analyse a production database
deps = DbDeps(db_path="/var/data/production.db")
result = agent.run_sync("What are the top 10 customers by revenue?", deps=deps)

# Same agent, different database
test_deps = DbDeps(db_path="test_fixtures.db")
result = agent.run_sync("How many rows in the users table?", deps=test_deps)
```
