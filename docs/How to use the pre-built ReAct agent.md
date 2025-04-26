# Using Pre-Built ReAct Agents in LangGraph

## Overview
This guide demonstrates how to use pre-built ReAct agents in LangGraph to create applications that interact with tools and respond to user queries.

## Key Concepts
- **ReAct Agent**: A pre-built agent architecture that decides whether to use tools or respond directly to the user.
- **Tool Integration**: Tools are used to fetch external data or perform specific actions.
- **Agent Workflow**: The agent decides whether to call tools or respond directly based on user input.

## Prerequisites
- Familiarity with agent architectures, chat models, and tools.
- Installed Python packages:
  ```bash
  pip install langgraph langchain-openai
  ```
- Environment variables (e.g., `OPENAI_API_KEY`) configured.

## Implementation Steps

### 1. Initialize the Model
Set up the language model for the agent.

```python
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o", temperature=0)
```

### 2. Define Tools
Create tools for the agent to use. For example, a weather tool:

```python
from typing import Literal
from langchain_core.tools import tool

@tool
def get_weather(city: Literal["nyc", "sf"]):
    """Use this to get weather information."""
    if city == "nyc":
        return "It might be cloudy in nyc"
    elif city == "sf":
        return "It's always sunny in sf"
    else:
        raise AssertionError("Unknown city")

tools = [get_weather]
```

### 3. Create the Agent Graph
Use the `create_react_agent` function to define the agent graph.

```python
from langgraph.prebuilt import create_react_agent

graph = create_react_agent(model, tools=tools)
```

### 4. Visualize the Graph
Generate a visual representation of the graph.

```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

### 5. Run the Agent
Test the agent with inputs that require tool calls and direct responses.

```python
inputs = {"messages": [("user", "what is the weather in sf")]}
for s in graph.stream(inputs, stream_mode="values"):
    print(s)

inputs = {"messages": [("user", "who built you?")]}
for s in graph.stream(inputs, stream_mode="values"):
    print(s)
```

## Notes
- Pre-built agents are a quick way to start but can be customized for advanced workflows.
- Tools must be defined with clear input/output specifications.

## References
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/how-tos/create-react-agent/)


