# How to Pass Private State Between Nodes

## Overview
In LangGraph, you might need nodes to exchange information that is crucial for intermediate logic but shouldn't be part of the main state schema. This private data is only relevant for specific nodes in the workflow and doesn't need to be exposed to all nodes or included in the final output of the graph.

## Key Concepts
- Private state passing between specific nodes
- Multiple typed schemas for different node interactions
- State isolation for security and cleaner code design
- Type hinting for better code organization

## Prerequisites
```python
# Install required packages
%pip install -U langgraph

# Import necessary modules
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict
```

## Implementation

### Define State Schemas
First, define the overall state schema and the private state schemas:

```python
# The overall state of the graph (public state shared across nodes)
class OverallState(TypedDict):
    a: str

# Output from node_1 contains private data not part of the overall state
class Node1Output(TypedDict):
    private_data: str

# Schema for node_2 input that only requires the private data
class Node2Input(TypedDict):
    private_data: str
```

### Define Node Functions
Create functions for each node with appropriate input and output types:

```python
# Node 1 accepts overall state but returns private data
def node_1(state: OverallState) -> Node1Output:
    output = {"private_data": "set by node_1"}
    print(f"Entered node `node_1`:\n\tInput: {state}.\n\tReturned: {output}")
    return output

# Node 2 accepts private data and returns updated overall state
def node_2(state: Node2Input) -> OverallState:
    output = {"a": "set by node_2"}
    print(f"Entered node `node_2`:\n\tInput: {state}.\n\tReturned: {output}")
    return output

# Node 3 only has access to the overall state (no access to private data)
def node_3(state: OverallState) -> OverallState:
    output = {"a": "set by node_3"}
    print(f"Entered node `node_3`:\n\tInput: {state}.\n\tReturned: {output}")
    return output
```

### Build and Invoke the Graph
Construct the graph with the defined nodes and edges:

```python
# Build the state graph with overall state as the main schema
builder = StateGraph(OverallState)

# Add nodes with their functions
builder.add_node(node_1)  # node_1 is the first node
builder.add_node(node_2)  # node_2 accepts private data from node_1
builder.add_node(node_3)  # node_3 does not see the private data

# Define the flow with edges
builder.add_edge(START, "node_1")  # Start with node_1
builder.add_edge("node_1", "node_2")  # Pass from node_1 to node_2
builder.add_edge("node_2", "node_3")  # Pass from node_2 to node_3 (only overall state)
builder.add_edge("node_3", END)  # End after node_3

# Compile and invoke the graph
graph = builder.compile()

# Invoke with initial state
response = graph.invoke({"a": "set at start"})

print(f"Output of graph invocation: {response}")
```

### Expected Output

When running the code, you'll see the private data flow properly between nodes:

```
Entered node `node_1`:
    Input: {'a': 'set at start'}.
    Returned: {'private_data': 'set by node_1'}

Entered node `node_2`:
    Input: {'private_data': 'set by node_1'}.
    Returned: {'a': 'set by node_2'}

Entered node `node_3`:
    Input: {'a': 'set by node_2'}.
    Returned: {'a': 'set by node_3'}

Output of graph invocation: {'a': 'set by node_3'}
```

## Benefits
- Cleaner state management by isolating temporary intermediate data
- Improved type safety with specific schemas for different node interactions
- Reduced complexity in the main state schema
- Better encapsulation of implementation details
- Prevention of data leakage to unintended nodes

## Considerations
- Type hints must be properly defined for correct state passing
- Private state is only passed between directly connected nodes
- The overall state schema defines what's visible to all nodes
- Each node can choose to return different state types
- Graph compilation validates that state types align between connected nodes
