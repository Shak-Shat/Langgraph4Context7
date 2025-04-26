# Subgraphs in LangGraph

## Overview
Subgraphs allow you to build complex systems with multiple components that are themselves graphs, such as multi-agent systems. This document explains how to implement subgraphs in LangGraph and demonstrates different approaches for state communication between parent graphs and subgraphs.

## Key Concepts
- Subgraphs are graphs that can be embedded within other graphs
- Communication between parent and subgraph happens via state
- Two main approaches: shared state keys or different schemas with transformation
- Subgraphs enable modular system design and component reuse

## Prerequisites
```python
from langgraph.graph import START, StateGraph
from typing import TypedDict
```

## Implementation

### Adding a Subgraph with Shared State Keys
When parent and subgraph share state keys, you can directly add the compiled subgraph as a node.

```python
# Define subgraph
class SubgraphState(TypedDict):
    foo: str  # note that this key is shared with the parent graph state
    bar: str

def subgraph_node_1(state: SubgraphState):
    return {"bar": "bar"}

def subgraph_node_2(state: SubgraphState):
    # note that this node is using a state key ('bar') that is only available in the subgraph
    # and is sending update on the shared state key ('foo')
    return {"foo": state["foo"] + state["bar"]}

subgraph_builder = StateGraph(SubgraphState)
subgraph_builder.add_node(subgraph_node_1)
subgraph_builder.add_node(subgraph_node_2)
subgraph_builder.add_edge(START, "subgraph_node_1")
subgraph_builder.add_edge("subgraph_node_1", "subgraph_node_2")
subgraph = subgraph_builder.compile()

# Define parent graph
class ParentState(TypedDict):
    foo: str

def node_1(state: ParentState):
    return {"foo": "hi! " + state["foo"]}

builder = StateGraph(ParentState)
builder.add_node("node_1", node_1)
# note that we're adding the compiled subgraph as a node to the parent graph
builder.add_node("node_2", subgraph)
builder.add_edge(START, "node_1")
builder.add_edge("node_1", "node_2")
graph = builder.compile()
```

### Adding a Subgraph with Different Schemas
When parent and subgraph have different state schemas, you need to create a node function that transforms the state appropriately.

```python
# Define subgraph
class SubgraphState(TypedDict):
    # note that none of these keys are shared with the parent graph state
    bar: str
    baz: str

def subgraph_node_1(state: SubgraphState):
    return {"baz": "baz"}

def subgraph_node_2(state: SubgraphState):
    return {"bar": state["bar"] + state["baz"]}

subgraph_builder = StateGraph(SubgraphState)
subgraph_builder.add_node(subgraph_node_1)
subgraph_builder.add_node(subgraph_node_2)
subgraph_builder.add_edge(START, "subgraph_node_1")
subgraph_builder.add_edge("subgraph_node_1", "subgraph_node_2")
subgraph = subgraph_builder.compile()

# Define parent graph
class ParentState(TypedDict):
    foo: str

def node_1(state: ParentState):
    return {"foo": "hi! " + state["foo"]}

def node_2(state: ParentState):
    # transform the state to the subgraph state
    response = subgraph.invoke({"bar": state["foo"]})
    # transform response back to the parent state
    return {"foo": response["bar"]}

builder = StateGraph(ParentState)
builder.add_node("node_1", node_1)
# note that instead of using the compiled subgraph we are using `node_2` function that is calling the subgraph
builder.add_node("node_2", node_2)
builder.add_edge(START, "node_1")
builder.add_edge("node_1", "node_2")
graph = builder.compile()
```

## Usage Example
You can stream the execution of a graph with subgraphs to see both parent and subgraph outputs:

```python
# Without subgraph details
for chunk in graph.stream({"foo": "foo"}):
    print(chunk)

# With subgraph details
for chunk in graph.stream({"foo": "foo"}, subgraphs=True):
    print(chunk)
```

## Benefits
- Enables modular system design with encapsulated components
- Facilitates building multi-agent systems
- Allows for component reuse across different applications
- Makes complex workflows more manageable

## Considerations
- Choose your approach based on the relationship between parent and subgraph state
- You cannot invoke more than one subgraph inside the same node
- Consider performance implications of nested subgraphs
- State transformations add complexity but enable greater flexibility
