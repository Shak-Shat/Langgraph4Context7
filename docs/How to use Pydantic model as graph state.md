# Using Pydantic Models for Graph State Validation

## Overview
This guide explains how to use Pydantic models for runtime validation of state inputs in LangGraph workflows. It ensures data consistency and error handling during graph execution.

## Key Concepts
- **Pydantic for Validation**: Implements runtime validation on inputs to nodes using Pydantic `BaseModel`.
- **State Graph**: A LangGraph structure that accepts a schema for state inputs and enables automated validation during execution.
- **Error Handling**: Errors arise from mismatched input types and are surfaced during graph invocation.

## Prerequisites
- Familiarity with LangGraph components (`StateGraph`, `START`, `END`, and nodes).
- Installed Python packages:
  ```bash
  pip install langgraph
  ```
- Environment variables (e.g., `OPENAI_API_KEY`) configured.

## Implementation Steps

### 1. Define the State Schema with Pydantic
Create the state schema using `BaseModel` to specify input validation rules.

```python
from langgraph.graph import StateGraph, START, END
from pydantic import BaseModel

# Define the shared state schema for the graph
class OverallState(BaseModel):
    a: str

def node(state: OverallState):
    return {"a": "goodbye"}
```

### 2. Build and Compile the State Graph
Configure and compile a LangGraph state graph using the Pydantic schema.

```python
# Build the state graph
builder = StateGraph(OverallState)
builder.add_node(node)  # Add node to the graph
builder.add_edge(START, "node")  # Start the graph with the node
builder.add_edge("node", END)  # End the graph after the node
graph = builder.compile()

# Invoke the graph with valid input
result = graph.invoke({"a": "hello"})
print(result)  # Output: {'a': 'goodbye'}
```

### 3. Test Runtime Validation
Demonstrate how invalid inputs trigger validation errors.

```python
# Test with invalid input (should fail)
try:
    graph.invoke({"a": 123})  # 'a' must be a string
except Exception as e:
    print("Validation Error:", e)
```

### 4. Multi-Node Validation
Validation works across multiple nodes. Errors occur when invalid inputs are passed between nodes.

```python
def bad_node(state: OverallState):
    return {"a": 123}  # Invalid

def ok_node(state: OverallState):
    return {"a": "goodbye"}

builder = StateGraph(OverallState)
builder.add_node(bad_node)
builder.add_node(ok_node)
builder.add_edge(START, "bad_node")
builder.add_edge("bad_node", "ok_node")
builder.add_edge("ok_node", END)
graph = builder.compile()

try:
    graph.invoke({"a": "hello"})
except Exception as e:
    print("Validation Error:", e)
```

## Notes
- Pydantic validation occurs only on inputs, not outputs.
- Validation errors do not indicate which node caused the issue.

## References
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/how-tos/state-model/)