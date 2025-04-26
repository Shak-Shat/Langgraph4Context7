# How to Update Graph State from Nodes

## Overview
This document explains how to define and update state in LangGraph, including using state schemas to define a graph's structure and using reducers to control how state updates are processed. Understanding state management is essential for building effective LLM applications with LangGraph.

## Key Concepts
- State schema definition using TypedDict, Pydantic models, or dataclasses
- State reducers for controlling how updates are applied
- Message state management with built-in reducers
- Node functions that return state updates rather than mutating state directly

## Prerequisites
```python
# Install LangGraph
%pip install -U langgraph

# Import necessary modules
from langchain_core.messages import AnyMessage, AIMessage, HumanMessage
from typing_extensions import TypedDict, Annotated
from langgraph.graph import StateGraph, START
```

## Implementation

### Defining State Schema
```python
# Basic state definition with a message list and additional field
class State(TypedDict):
    messages: list[AnyMessage]
    extra_field: int
```

### Creating Node Functions
```python
# Define a node function that updates state
def node(state: State):
    messages = state["messages"]
    new_message = AIMessage("Hello!")
    
    # Return state updates instead of mutating state
    return {"messages": messages + [new_message], "extra_field": 10}
```

### Building the Graph
```python
# Create a graph with our state schema
graph_builder = StateGraph(State)
graph_builder.add_node(node)
graph_builder.set_entry_point("node")
graph = graph_builder.compile()

# Invoke the graph
result = graph.invoke({"messages": [HumanMessage("Hi")]})

# Inspect the results
for message in result["messages"]:
    message.pretty_print()
```

### Using State Reducers
Reducers control how updates from nodes are applied to each key in the state. By default, updates override the existing value, but custom reducers can change this behavior.

```python
# Define a custom reducer function
def add(left, right):
    """Combines two values by addition."""
    return left + right

# Use the reducer in state definition
class StateWithReducer(TypedDict):
    messages: Annotated[list[AnyMessage], add]
    extra_field: int

# Simplified node that leverages the reducer
def node_with_reducer(state: StateWithReducer):
    new_message = AIMessage("Hello!")
    # Only return the new message - the reducer will handle combining it
    return {"messages": [new_message], "extra_field": 10}

# Create graph with reducer-enabled state
graph = StateGraph(StateWithReducer).add_node(node_with_reducer).add_edge(START, "node_with_reducer").compile()
```

### Using Built-in Message Reducers
LangGraph provides a specialized reducer for message lists that handles common operations like appending new messages and supports various message formats.

```python
from langgraph.graph.message import add_messages

# State with built-in message reducer
class MessageState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
    extra_field: int

# Create a graph with this state
graph = StateGraph(MessageState).add_node(node).set_entry_point("node").compile()

# Can handle different message formats, including OpenAI format
input_message = {"role": "user", "content": "Hi"}
result = graph.invoke({"messages": [input_message]})
```

### Using Pre-built Message State
For convenience, LangGraph includes a pre-built MessagesState class for applications involving chat models.

```python
from langgraph.graph import MessagesState

# Extend the built-in MessagesState
class CustomMessageState(MessagesState):
    extra_field: int
```

## Benefits
- Clean separation of state definition and update logic
- Flexible state schemas that match application needs
- Automatic handling of state updates with reducers
- Support for different message formats and update patterns
- Improved code organization and maintainability

## Considerations
- Nodes should always return state updates rather than mutating state directly
- Each key in state can have its own independent reducer function
- Default behavior is to override existing values if no reducer is specified
- The message reducer can handle both message objects and dictionary formats
- Graph input and output schema is determined by the state schema by default 