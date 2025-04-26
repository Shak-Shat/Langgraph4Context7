# Multi-Agent Network

## Overview
This document demonstrates the creation of a multi-agent network using LangGraph for collaborative problem-solving. Agents are specialized in tasks and communicate via a divide-and-conquer approach.

## Key Concepts
- **Multi-Agent Network:** A system where specialized agents coordinate tasks to solve complex problems.
- **Graph Workflow:** Agents are connected nodes in a state graph, passing tasks dynamically until completion.
- **LangGraph Integration:** Utilizes LangGraph libraries for defining workflows and agent interactions.

## Prerequisites
Ensure the following dependencies are installed:
```bash
pip install langgraph langchain-anthropic langchain-community
```
Set up API keys for LangSmith and other tools:
```python
import os
os.environ["ANTHROPIC_API_KEY"] = "your_key_here"
os.environ["LANGSMITH_API_KEY"] = "your_key_here"
```

## Implementation

### 1. Define Tools
```python
from langchain_community.tools.tavily_search import TavilySearchResults
from langchain_experimental.utilities import PythonREPL
from langchain_core.tools import tool
from typing import Annotated

# Define tools for search and Python code execution
tavily_tool = TavilySearchResults(max_results=5)
repl = PythonREPL()

@tool
def python_repl_tool(code: Annotated[str, "Python code to execute"]):
    try:
        result = repl.run(code)
    except BaseException as e:
        return f"Failed to execute: {repr(e)}"
    return f"Successfully executed:\n{result}"
```

### 2. Define Agent Nodes
```python
from langgraph.prebuilt import create_react_agent

# Initialization of LLM model
llm = ChatAnthropic(model="claude-3-5-sonnet-latest")

# Create agents
research_agent = create_react_agent(llm, tools=[tavily_tool])
chart_agent = create_react_agent(llm, tools=[python_repl_tool])
```

### 3. Create State Nodes and Define Workflow
```python
from langgraph.graph import MessagesState, StateGraph, START, END
from langgraph.types import Command

# Define state mapping
def get_next_node(last_message, goto):
    if "FINAL ANSWER" in last_message.content:
        return END
    return goto

def research_node(state: MessagesState):
    result = research_agent.invoke(state)
    goto = get_next_node(result["messages"][-1], "chart_generator")
    return Command(update={"messages": result["messages"]}, goto=goto)

def chart_node(state: MessagesState):
    result = chart_agent.invoke(state)
    goto = get_next_node(result["messages"][-1], "researcher")
    return Command(update={"messages": result["messages"]}, goto=goto)

# Create the graph
workflow = StateGraph(MessagesState)
workflow.add_node("researcher", research_node)
workflow.add_node("chart_generator", chart_node)
workflow.add_edge(START, "researcher")
graph = workflow.compile()
```

### 4. Run Invocation
```python
events = graph.stream({"messages": [("user", "Get UK's GDP over 5 years")]}, {"recursion_limit": 150})
for s in events:
    print(s)
```

## Usage Example
Multi-agent network allows agents to collaborate dynamically. Example:
```python
workflow.invoke({"messages": [("user", "Generate a chart from UK GDP data")]})
```

## Benefits
- Enables distributed problem-solving by combining agent expertise.
- Scalable architecture for complex tasks via LangGraph workflows and state checkpoints.

## Considerations
- Ensure sandboxing for tools like PythonREPL to prevent unsafe code execution.
- Define tool-specific behaviors for reliable results.

---

This completes the transformation of `Multi-agent network.md`. Let me know if you want to proceed with the next document!