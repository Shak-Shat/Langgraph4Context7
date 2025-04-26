# How to Transform Inputs and Outputs of a Subgraph

## Overview
When creating complex agent systems with LangGraph, you might need to use nested subgraphs where the states are completely independent - the parent graph and subgraphs have no overlapping keys. This document explains how to transform the state between parent graphs and subgraphs to enable proper communication between them.

## Key Concepts
- State transformation between parent graphs and subgraphs
- Independent state schemas for different graph levels
- Wrapper functions for handling state transformation
- Nested subgraph execution within parent graphs

## Prerequisites
```python
# Install required packages
%pip install -U langgraph

# Import necessary modules
from typing_extensions import TypedDict
from langgraph.graph.state import StateGraph, START, END
```

## Implementation

### Define a Grandchild Subgraph
Let's start by defining the deepest subgraph (grandchild) with its own independent state:

```python
# Define grandchild state schema
class GrandChildState(TypedDict):
    my_grandchild_key: str

# Define grandchild node function
def grandchild_1(state: GrandChildState) -> GrandChildState:
    # Only has access to grandchild state keys
    return {"my_grandchild_key": state["my_grandchild_key"] + ", how are you"}

# Create and compile grandchild graph
grandchild = StateGraph(GrandChildState)
grandchild.add_node("grandchild_1", grandchild_1)
grandchild.add_edge(START, "grandchild_1")
grandchild.add_edge("grandchild_1", END)
grandchild_graph = grandchild.compile()

# Test the grandchild graph independently
result = grandchild_graph.invoke({"my_grandchild_key": "hi Bob"})
# Output: {'my_grandchild_key': 'hi Bob, how are you'}
```

### Define a Child Subgraph with Transformation
Next, we'll define a middle-level subgraph (child) that calls the grandchild:

```python
# Define child state schema
class ChildState(TypedDict):
    my_child_key: str

# Define a wrapper function to transform state between child and grandchild
def call_grandchild_graph(state: ChildState) -> ChildState:
    # Transform child state to grandchild state
    grandchild_graph_input = {"my_grandchild_key": state["my_child_key"]}
    
    # Call grandchild graph with transformed state
    grandchild_graph_output = grandchild_graph.invoke(grandchild_graph_input)
    
    # Transform grandchild output back to child state
    return {"my_child_key": grandchild_graph_output["my_grandchild_key"] + " today?"}

# Create and compile child graph
child = StateGraph(ChildState)
child.add_node("child_1", call_grandchild_graph)  # Use wrapper function, not the graph directly
child.add_edge(START, "child_1")
child.add_edge("child_1", END)
child_graph = child.compile()

# Test the child graph independently
result = child_graph.invoke({"my_child_key": "hi Bob"})
# Output: {'my_child_key': 'hi Bob, how are you today?'}
```

### Define a Parent Graph with Transformations
Finally, we'll create the top-level parent graph that calls the child:

```python
# Define parent state schema
class ParentState(TypedDict):
    my_key: str

# Define parent node functions
def parent_1(state: ParentState) -> ParentState:
    # Only has access to parent state keys
    return {"my_key": "hi " + state["my_key"]}

def parent_2(state: ParentState) -> ParentState:
    return {"my_key": state["my_key"] + " bye!"}

# Define a wrapper function to transform state between parent and child
def call_child_graph(state: ParentState) -> ParentState:
    # Transform parent state to child state
    child_graph_input = {"my_child_key": state["my_key"]}
    
    # Call child graph with transformed state
    child_graph_output = child_graph.invoke(child_graph_input)
    
    # Transform child output back to parent state
    return {"my_key": child_graph_output["my_child_key"]}

# Create and compile parent graph
parent = StateGraph(ParentState)
parent.add_node("parent_1", parent_1)
parent.add_node("child", call_child_graph)  # Use wrapper function, not the graph directly
parent.add_node("parent_2", parent_2)

parent.add_edge(START, "parent_1")
parent.add_edge("parent_1", "child")
parent.add_edge("child", "parent_2")
parent.add_edge("parent_2", END)

parent_graph = parent.compile()

# Test the complete parent graph
result = parent_graph.invoke({"my_key": "Bob"})
# Output: {'my_key': 'hi Bob, how are you today? bye!'}
```

## Benefits
- Enables composition of independent graph components with different state schemas
- Maintains clean separation of concerns between graph levels
- Allows each subgraph to have its own focused state definition
- Supports testing each subgraph independently
- Facilitates reuse of subgraphs across different parent graphs

## Considerations
- You must explicitly transform state between parent and subgraph levels
- Direct passing of subgraphs to `.add_node()` will fail if states don't share keys
- Each subgraph maintains its own independent state
- Transformation functions need to handle all necessary state mapping
- Consider using shared state schemas if subgraphs have significant state overlap
