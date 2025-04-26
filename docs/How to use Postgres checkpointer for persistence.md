# Using Postgres Checkpointer for Persistence

## Overview
This document demonstrates how to use Postgres as the backend for persisting agent states in LangGraph workflows. Persistent state allows agents to remember previous interactions and enables uninterrupted multi-turn operations.

## Key Concepts
- **Postgres Checkpointer:** Manages agent state persistence using LangGraph's checkpoint library.
- **Persistence in LangGraph:** Facilitates state-aware workflows for multi-turn agent interactions.
- **Connection Types:**
  - Synchronous
  - Connection Pooling (efficient for short-lived operations)
  - Asynchronous (non-blocking for high concurrency scenarios)

## Prerequisites
Set up a Postgres instance with the required access credentials. Example configuration:
```python
DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"
```
Ensure the following dependencies are installed:
```bash
pip install psycopg psycopg-pool langgraph langgraph-checkpoint-postgres
```
Set up API keys for LangSmith integration:
```python
import getpass
import os

def _set_env(var: str):
    if not os.environ.get(var):
        os.environ[var] = getpass.getpass(f"{var}: ")

_set_env("OPENAI_API_KEY")
_set_env("LANGSMITH_API_KEY")
```

## Implementation

### 1. Define Model and Tools
```python
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent
from langgraph.checkpoint.postgres import PostgresSaver

@tool
def get_weather(city: str):
    """Use this to get weather information."""
    if city == "nyc":
        return "It might be cloudy in nyc"
    elif city == "sf":
        return "It's always sunny in sf"
    else:
        return "Unknown city"

tools = [get_weather]
model = ChatOpenAI(model_name="gpt-4o-mini", temperature=0)
```

### 2. Synchronous Connection Setup
```python
DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"
connection_kwargs = {
    "autocommit": True,
    "prepare_threshold": 0,
}

from psycopg import Connection

with Connection.connect(DB_URI, **connection_kwargs) as conn:
    checkpointer = PostgresSaver(conn)
    checkpointer.setup()  # Initialize the database schema

    graph = create_react_agent(model, tools=tools, checkpointer=checkpointer)
    config = {"configurable": {"thread_id": "1"}}

    res = graph.invoke({"messages": [("human", "what's the weather in sf")]}, config)
```

### 3. Connection Pool Setup
```python
from psycopg_pool import ConnectionPool

with ConnectionPool(
    conninfo=DB_URI,
    max_size=20,
    kwargs=connection_kwargs,
) as pool:
    checkpointer = PostgresSaver(pool)
    checkpointer.setup()  # Initialize the database schema

    graph = create_react_agent(model, tools=tools, checkpointer=checkpointer)
    config = {"configurable": {"thread_id": "1"}}

    res = graph.invoke({"messages": [("human", "what's the weather in sf")]}, config)
```

### 4. Asynchronous Connection Setup
```python
from psycopg.async_connection import AsyncConnection

async with await AsyncConnection.connect(DB_URI, **connection_kwargs) as conn:
    from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver

    checkpointer = AsyncPostgresSaver(conn)
    await checkpointer.setup()  # Initialize the database schema

    graph = create_react_agent(model, tools=tools, checkpointer=checkpointer)
    config = {"configurable": {"thread_id": "2"}}

    res = await graph.ainvoke({"messages": [("human", "what's the weather in nyc")]}, config)
```

### 5. Using Connection Strings
```python
from langgraph.checkpoint.postgres import PostgresSaver

with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    graph = create_react_agent(model, tools=tools, checkpointer=checkpointer)
    config = {"configurable": {"thread_id": "3"}}

    res = graph.invoke({"messages": [("human", "what's the weather in sf")]}, config)
```

## Usage Example
Persistence allows agents to maintain state between interactions. Example usage:
```python
config = {"configurable": {"thread_id": "session_123"}}
messages = [("human", "what's the weather in nyc")]

res = graph.invoke({"messages": messages}, config)
```

## Benefits
- **State Persistence:** Agents remember previous interactions, improving user experience.
- **Flexibility:** Supports synchronous, connection pool, and asynchronous setups.
- **Scalable Workflows:** Enables high-concurrency operations for persistent multi-turn workflows.

## Considerations
- Ensure secure handling of API keys and credentials.
- Test database connections for performance bottlenecks.
- Use sandboxing for tools to prevent unsafe command execution.

---

This completes the transformation of `How to use Postgres checkpointer for persistence.md`. Let me know if you’re ready for the next document!