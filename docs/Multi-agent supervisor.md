# Multi-Agent System with Supervisor

## Overview
This document outlines the creation of a multi-agent system using LangGraph, where a supervisor agent orchestrates task delegation among specialized worker agents.

## Key Concepts
- **Supervisor Agent:** Manages task coordination by selecting worker agents or completing tasks.
- **Worker Agents:** Specialized agents for tasks such as web research or plotting.
- **LangGraph Prebuilt Features:** Utilizes LangGraph’s `create_react_agent` and structured workflows.

## Prerequisites
Ensure the following dependencies are installed:
```bash
pip install langgraph langchain-anthropic langchain-community
```
Set up API keys for LangSmith and Anthropic:
```python
# Set up API keys
import os
os.environ["ANTHROPIC_API_KEY"] = "your_key_here"
```

## Implementation

### 1. Define Tools
```python
from langchain_community.tools.tavily_search import TavilySearchResults
from langchain_core.tools import tool
from langchain_experimental.utilities import PythonREPL
from typing import Annotated

tavily_tool = TavilySearchResults(max_results=5)
repl = PythonREPL()

@tool
def python_repl_tool(code: Annotated[str, "Python code to execute to generate your chart."]):
    """Executes Python code locally. Handle with care to avoid unsafe commands."""
    try:
        result = repl.run(code)
    except BaseException as e:
        return f"Failed to execute. Error: {repr(e)}"
    return f"Successfully executed:\n{result}"
```

### 2. Create Supervisor Agent
```python
from typing import Literal
from typing_extensions import TypedDict
from langchain_anthropic import ChatAnthropic
from langgraph.graph import MessagesState, END
from langgraph.types import Command

# Define worker agents
members = ["researcher", "coder"]
options = members + ["FINISH"]

system_prompt = (
    "You are a supervisor managing a conversation between the following workers: "
    f"{members}. Respond with the worker to act next or FINISH when tasks are complete."
)

class Router(TypedDict):
    """Worker to route to next. If no workers needed, route to FINISH."""
    next: Literal[*options]

# LLM-based Supervisor
llm = ChatAnthropic(model="claude-3-5-sonnet-latest")

def supervisor_node(state: MessagesState) -> Command:
    messages = [{"role": "system", "content": system_prompt}] + state["messages"]
    response = llm.invoke(messages).structure_output(Router)
    goto = response["next"]
    if goto == "FINISH":
        goto = END
    return Command(goto=goto, update={"next": goto})
```

### 3. Define Worker Nodes
```python
from langgraph.graph import StateGraph, START, END
from langgraph.prebuilt import create_react_agent

# Create worker agents
research_agent = create_react_agent(llm, tools=[tavily_tool], prompt="You are a researcher.")
coder_agent = create_react_agent(llm, tools=[python_repl_tool], prompt="You are a coder.")

def research_node(state: MessagesState) -> Command:
    result = research_agent.invoke(state)
    return Command(update={"messages": result["messages"]}, goto="supervisor")

def code_node(state: MessagesState) -> Command:
    result = coder_agent.invoke(state)
    return Command(update={"messages": result["messages"]}, goto="supervisor")
```

### 4. Construct Graph Workflow
```python
builder = StateGraph(MessagesState)
builder.add_node("supervisor", supervisor_node)
builder.add_node("researcher", research_node)
builder.add_node("coder", code_node)
builder.add_edge(START, "supervisor")
graph = builder.compile()
```

### 5. Invoke Example
```python
for s in graph.stream(
    {"messages": [("user", "Find the latest GDP of New York and California, then calculate the average")]}
):
    print(s)
```

## Usage Example
This supervisor-based workflow distributes tasks dynamically among agents based on the user query.

## Benefits
- Orchestrates multi-agent workflows efficiently.
- Flexible for complex tasks, enabling dynamic delegation.
- Scalable worker management.

## Considerations
- Ensure safe execution for Python code.
- Test error handling thoroughly to avoid misuse across tools.

---

This completes the transformation of `Multi-agent supervisor.md`. Let me know if you’re ready for the next document!