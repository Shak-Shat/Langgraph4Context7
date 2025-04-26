# Streaming from Subgraphs in LangGraph

## Overview
This document explains how to stream outputs from both parent graphs and their embedded subgraphs in LangGraph. When working with complex graph structures, it's often important to see the intermediate results from all levels of the graph hierarchy during execution.

## Key Concepts
- Subgraph streaming enables visibility into nested graph execution
- The `subgraphs=True` parameter activates streaming from nested subgraphs
- Namespaces are used to identify which graph or subgraph produced each update
- Streaming follows the execution path through both parent and child graphs

## Prerequisites
```python
from langgraph.graph import START, StateGraph
from typing import TypedDict
```

## Implementation

### Creating a Subgraph
First, define and compile a subgraph:

```python
# Define subgraph state
class SubgraphState(TypedDict):
    foo: str  # note that this key is shared with the parent graph state
    bar: str

def subgraph_node_1(state: SubgraphState):
    return {"bar": "bar"}

def subgraph_node_2(state: SubgraphState):
    return {"foo": state["foo"] + state["bar"]}

# Build the subgraph
subgraph_builder = StateGraph(SubgraphState)
subgraph_builder.add_node("subgraph_node_1", subgraph_node_1)
subgraph_builder.add_node("subgraph_node_2", subgraph_node_2)
subgraph_builder.add_edge(START, "subgraph_node_1")
subgraph_builder.add_edge("subgraph_node_1", "subgraph_node_2")
subgraph = subgraph_builder.compile()
```

### Building the Parent Graph
Next, create a parent graph that incorporates the subgraph:

```python
# Define parent graph state
class ParentState(TypedDict):
    foo: str

def node_1(state: ParentState):
    return {"foo": "hi! " + state["foo"]}

# Build the parent graph
builder = StateGraph(ParentState)
builder.add_node("node_1", node_1)
builder.add_node("node_2", subgraph)  # Add the subgraph as a node
builder.add_edge(START, "node_1")
builder.add_edge("node_1", "node_2")
graph = builder.compile()
```

## Usage Examples

### Streaming from Parent Graph Only
By default, streaming only shows updates from the parent graph nodes:

```python
for chunk in graph.stream({"foo": "foo"}, stream_mode="updates"):
    print(chunk)

# Output:
# {'node_1': {'foo': 'hi! foo'}}
# {'node_2': {'foo': 'hi! foobar'}}
```

### Streaming from Both Parent and Subgraphs
To see updates from both the parent graph and all subgraphs, use `subgraphs=True`:

```python
for chunk in graph.stream(
    {"foo": "foo"},
    stream_mode="updates",
    subgraphs=True,
):
    print(chunk)

# Output:
# ((), {'node_1': {'foo': 'hi! foo'}})
# (('node_2:b692b345-cfb3-b709-628c-f0ba9608f72e',), {'subgraph_node_1': {'bar': 'bar'}})
# (('node_2:b692b345-cfb3-b709-628c-f0ba9608f72e',), {'subgraph_node_2': {'foo': 'hi! foobar'}})
# ((), {'node_2': {'foo': 'hi! foobar'}})
```

## Benefits
- Complete visibility into the execution of complex nested graphs
- Ability to debug and monitor subgraph execution
- More granular insight into how state changes propagate
- Better understanding of data flow in hierarchical graph structures

## Considerations
- Increased volume of streamed data when subgraphs are included
- Need to process namespace information to identify subgraph outputs
- May require additional client-side logic to present hierarchical updates
- Consider performance impact when streaming from large, complex graph structures
