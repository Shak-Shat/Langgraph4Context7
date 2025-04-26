# Running LangGraph Asynchronously

## Overview
This document explains how to implement and use asynchronous execution in LangGraph to achieve significant performance improvements, particularly for IO-bound operations like concurrent API requests to LLM providers. By leveraging Python's async/await syntax, you can run graphs more efficiently when dealing with external dependencies.

## Key Concepts
- Asynchronous execution using Python's `async`/`await` syntax
- Converting synchronous graph implementations to asynchronous ones
- Leveraging the Runnable Protocol's async methods
- Streaming intermediate results and LLM tokens

## Prerequisites
```python
from typing import Annotated, Literal
from typing_extensions import TypedDict
from langgraph.graph.message import add_messages
from langgraph.graph import END, StateGraph, START
from langgraph.prebuilt import ToolNode
from langchain_anthropic import ChatAnthropic
from langchain_core.messages import HumanMessage
from langchain_core.tools import tool
```

## Implementation

### Defining State with Annotations
First, define your state object with appropriate annotations for how values should be updated:

```python
class State(TypedDict):
    messages: Annotated[list, add_messages]
```

### Creating Asynchronous Nodes
Define your nodes as asynchronous functions:

```python
# Define the function that determines whether to continue or not
def should_continue(state: State) -> Literal["end", "continue"]:
    messages = state["messages"]
    last_message = messages[-1]
    # If there is no tool call, then we finish
    if not last_message.tool_calls:
        return "end"
    # Otherwise if there is, we continue
    else:
        return "continue"

# Define the asynchronous function that calls the model
async def call_model(state: State):
    messages = state["messages"]
    response = await model.ainvoke(messages)
    # We return a list, because this will get added to the existing list
    return {"messages": [response]}
```

### Setting Up Tools
Create tools that your graph will use:

```python
@tool
def search(query: str):
    """Call to surf the web."""
    # Implementation here
    return ["The answer to your question lies within."]

tools = [search]
tool_node = ToolNode(tools)
```

### Configuring the LLM
Set up your language model with tool binding:

```python
model = ChatAnthropic(model="claude-3-haiku-20240307")
model = model.bind_tools(tools)
```

### Building the Graph
Construct the graph with nodes and edges:

```python
# Define a new graph
workflow = StateGraph(State)

# Add nodes
workflow.add_node("agent", call_model)
workflow.add_node("action", tool_node)

# Set the entrypoint
workflow.add_edge(START, "agent")

# Add conditional edges
workflow.add_conditional_edges(
    "agent",
    should_continue,
    {
        "continue": "action",
        "end": END,
    },
)

# Add normal edge from action back to agent
workflow.add_edge("action", "agent")

# Compile the graph
app = workflow.compile()
```

## Usage Examples

### Basic Asynchronous Invocation
Run the graph asynchronously with a simple input:

```python
inputs = {"messages": [HumanMessage(content="what is the weather in sf")]}
result = await app.ainvoke(inputs)
```

### Streaming Node Outputs
Stream intermediate results as they are produced:

```python
inputs = {"messages": [HumanMessage(content="what is the weather in sf")]}
async for output in app.astream(inputs, stream_mode="updates"):
    for key, value in output.items():
        print(f"Output from node '{key}':")
        print(value["messages"][-1].pretty_print())
```

### Streaming LLM Tokens
Stream individual LLM tokens as they are generated:

```python
inputs = {"messages": [HumanMessage(content="what is the weather in sf")]}
async for output in app.astream_log(inputs, include_types=["llm"]):
    for op in output.ops:
        if op["path"].startswith("/logs/") and op["path"].endswith("/streamed_output/-"):
            try:
                content = op["value"].content[0]
                if "text" in content:
                    print(content["text"], end="")
            except:
                pass
```

## Benefits
- Improved performance for IO-bound operations
- Reduced latency for concurrent API calls
- More responsive user experiences through streaming
- Ability to see intermediate results during execution
- Better resource utilization

## Considerations
- Requires familiarity with async programming patterns
- All nodes must be properly converted to async pattern
- Error handling is more complex in asynchronous code
- Ensure all libraries used support async operations
