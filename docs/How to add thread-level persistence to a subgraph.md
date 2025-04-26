# Thread-Level Persistence for Subgraphs in LangGraph

## Overview
This guide demonstrates how to implement thread-level persistence in LangGraph applications that use subgraphs, allowing state to be maintained across interactions for both parent graphs and their subgraphs.

## Key Concepts
- Parent graphs can propagate persistence to subgraphs automatically
- Checkpointers should only be provided at the parent graph level
- Subgraph states are stored with specialized identifiers
- Shared and subgraph-specific state keys can coexist

## Prerequisites
```python
%pip install -U langgraph
```

## Implementation

### Creating the Subgraph
```python
from langgraph.graph import START, StateGraph
from langgraph.checkpoint.memory import MemorySaver
from typing import TypedDict


# subgraph

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
```

### Creating the Parent Graph
```python
# parent graph

class State(TypedDict):
    foo: str


def node_1(state: State):
    return {"foo": "hi! " + state["foo"]}


builder = StateGraph(State)
builder.add_node("node_1", node_1)
# note that we're adding the compiled subgraph as a node to the parent graph
builder.add_node("node_2", subgraph)
builder.add_edge(START, "node_1")
builder.add_edge("node_1", "node_2")
```

### Adding Persistence
```python
checkpointer = MemorySaver()
# You must only pass checkpointer when compiling the parent graph.
# LangGraph will automatically propagate the checkpointer to the child subgraphs.
graph = builder.compile(checkpointer=checkpointer)
```

## Usage Example

### Running the Graph with Persistence
```python
config = {"configurable": {"thread_id": "1"}}

for _, chunk in graph.stream({"foo": "foo"}, config, subgraphs=True):
    print(chunk)
```

Example output:
```
{'node_1': {'foo': 'hi! foo'}}
{'subgraph_node_1': {'bar': 'bar'}}
{'subgraph_node_2': {'foo': 'hi! foobar'}}
{'node_2': {'foo': 'hi! foobar'}}
```

### Accessing Parent Graph State
```python
graph.get_state(config).values
```

Output:
```
{'foo': 'hi! foobar'}
```

### Accessing Subgraph State
```python
# Find the state snapshot before we return results from node_2 (the node with subgraph)
state_with_subgraph = [
    s for s in graph.get_state_history(config) if s.next == ("node_2",)
][0]

# Extract the subgraph config from the state snapshot
subgraph_config = state_with_subgraph.tasks[0].state
subgraph_config
```

Output:
```
{'configurable': {'thread_id': '1',
  'checkpoint_ns': 'node_2:6ef111a6-f290-7376-0dfc-a4152307bc5b'}}
```

```python
# Access the subgraph state using the extracted config
graph.get_state(subgraph_config).values
```

Output:
```
{'foo': 'hi! foobar', 'bar': 'bar'}
```

## Benefits
- Automatic propagation of persistence from parent to child graphs
- Clean separation of concerns between graph levels
- Ability to maintain both shared and subgraph-specific state
- Simplified persistence management with single checkpointer configuration

## Considerations
- Only configure checkpointers at the parent graph level
- Accessing subgraph state requires additional steps to find the correct config
- Shared state keys between parent and subgraphs require careful management
- Thread IDs are automatically propagated with specialized namespaces for subgraphs
