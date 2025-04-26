# Managing State in Nested Subgraphs

## Overview
This guide demonstrates how to view, modify, and update state within LangGraph subgraphs, enabling powerful human-in-the-loop interactions with nested graph structures.

## Key Concepts
- Subgraph state can be accessed and modified at any level of nesting
- Breakpoints allow for inspection and intervention during graph execution
- State can be modified at specific points to influence graph behavior
- Developers can act as specific nodes or entire subgraphs to control execution
- State history enables rewinding and replaying from specific points

## Prerequisites
```python
# Install required packages
%pip install -U langgraph

# Set API keys
import os
import getpass

def _set_env(var: str):
    if not os.environ.get(var):
        os.environ[var] = getpass.getpass(f"{var}: ")

_set_env("OPENAI_API_KEY")
```

## Implementation

### Define a Simple Subgraph
```python
from langgraph.graph import StateGraph, END, START, MessagesState
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI

@tool
def get_weather(city: str):
    """Get the weather for a specific city"""
    return f"It's sunny in {city}!"

raw_model = ChatOpenAI(model="gpt-4o")
model = raw_model.with_structured_output(get_weather)

class SubGraphState(MessagesState):
    city: str

def model_node(state: SubGraphState):
    result = model.invoke(state["messages"])
    return {"city": result["city"]}

def weather_node(state: SubGraphState):
    result = get_weather.invoke({"city": state["city"]})
    return {"messages": [{"role": "assistant", "content": result}]}

subgraph = StateGraph(SubGraphState)
subgraph.add_node(model_node)
subgraph.add_node(weather_node)
subgraph.add_edge(START, "model_node")
subgraph.add_edge("model_node", "weather_node")
subgraph.add_edge("weather_node", END)
subgraph = subgraph.compile(interrupt_before=["weather_node"])
```

### Define Parent Graph
```python
from typing import Literal
from typing_extensions import TypedDict
from langgraph.checkpoint.memory import MemorySaver

memory = MemorySaver()

class RouterState(MessagesState):
    route: Literal["weather", "other"]

class Router(TypedDict):
    route: Literal["weather", "other"]

router_model = raw_model.with_structured_output(Router)

def router_node(state: RouterState):
    system_message = "Classify the incoming query as either about weather or not."
    messages = [{"role": "system", "content": system_message}] + state["messages"]
    route = router_model.invoke(messages)
    return {"route": route["route"]}

def normal_llm_node(state: RouterState):
    response = raw_model.invoke(state["messages"])
    return {"messages": [response]}

def route_after_prediction(
    state: RouterState,
) -> Literal["weather_graph", "normal_llm_node"]:
    if state["route"] == "weather":
        return "weather_graph"
    else:
        return "normal_llm_node"

graph = StateGraph(RouterState)
graph.add_node(router_node)
graph.add_node(normal_llm_node)
graph.add_node("weather_graph", subgraph)
graph.add_edge(START, "router_node")
graph.add_conditional_edges("router_node", route_after_prediction)
graph.add_edge("normal_llm_node", END)
graph.add_edge("weather_graph", END)
graph = graph.compile(checkpointer=memory)
```

## Usage Examples

### Basic Graph Execution
```python
# Test with a non-weather query
config = {"configurable": {"thread_id": "1"}}
inputs = {"messages": [{"role": "user", "content": "hi!"}]}
for update in graph.stream(inputs, config=config, stream_mode="updates"):
    print(update)
```

### Working with Breakpoints
```python
# Stream with a weather query (will trigger subgraph)
config = {"configurable": {"thread_id": "2"}}
inputs = {"messages": [{"role": "user", "content": "what's the weather in sf"}]}
for update in graph.stream(inputs, config=config, stream_mode="updates"):
    print(update)

# Stream with subgraph events included
config = {"configurable": {"thread_id": "3"}}
inputs = {"messages": [{"role": "user", "content": "what's the weather in sf"}]}
for update in graph.stream(inputs, config=config, stream_mode="values", subgraphs=True):
    print(update)

# Get the current state
state = graph.get_state(config)
print(state.next)  # Shows where execution is paused

# Get detailed state including subgraphs
state = graph.get_state(config, subgraphs=True)
print(state.tasks[0])  # Shows subgraph state

# Resume execution from breakpoint
for update in graph.stream(None, config=config, stream_mode="values", subgraphs=True):
    print(update)
```

### Resuming from Specific Subgraph Nodes
```python
# Find the state before the model_node
parent_graph_state_before_subgraph = next(
    h for h in graph.get_state_history(config) if h.next == ("weather_graph",)
)
subgraph_state_before_model_node = next(
    h
    for h in graph.get_state_history(parent_graph_state_before_subgraph.tasks[0].state)
    if h.next == ("model_node",)
)

# Confirm it's the right state
print(subgraph_state_before_model_node.next)  # Should be ('model_node',)

# Resume from the model_node
for value in graph.stream(
    None,
    config=subgraph_state_before_model_node.config,
    stream_mode="values",
    subgraphs=True,
):
    print(value)
```

### Modifying Subgraph State
```python
# Start streaming until breakpoint
config = {"configurable": {"thread_id": "4"}}
inputs = {"messages": [{"role": "user", "content": "what's the weather in sf"}]}
for update in graph.stream(inputs, config=config, stream_mode="updates"):
    print(update)

# Get state with subgraphs
state = graph.get_state(config, subgraphs=True)
print(state.values["messages"])

# Update city in subgraph state
graph.update_state(state.tasks[0].state.config, {"city": "la"})

# Resume execution
for update in graph.stream(None, config=config, stream_mode="updates", subgraphs=True):
    print(update)  # Should show "It's sunny in LA!"
```

### Acting as a Subgraph Node
```python
# Start streaming until breakpoint
config = {"configurable": {"thread_id": "14"}}
inputs = {"messages": [{"role": "user", "content": "what's the weather in sf"}]}
for update in graph.stream(inputs, config=config, stream_mode="updates", subgraphs=True):
    print(update)
print("interrupted!")

# Get state
state = graph.get_state(config, subgraphs=True)

# Act as the weather_node
graph.update_state(
    state.tasks[0].state.config,
    {"messages": [{"role": "assistant", "content": "rainy"}]},
    as_node="weather_node",
)

# Resume execution
for update in graph.stream(None, config=config, stream_mode="updates", subgraphs=True):
    print(update)

# Check final messages
print(graph.get_state(config).values["messages"])
```

### Acting as an Entire Subgraph
```python
# Start streaming until breakpoint
config = {"configurable": {"thread_id": "8"}}
inputs = {"messages": [{"role": "user", "content": "what's the weather in sf"}]}
for update in graph.stream(inputs, config=config, stream_mode="updates", subgraphs=True):
    print(update)
print("interrupted!")

# Act as entire weather_graph
graph.update_state(
    config,
    {"messages": [{"role": "assistant", "content": "rainy"}]},
    as_node="weather_graph",
)

# Resume execution
for update in graph.stream(None, config=config, stream_mode="updates"):
    print(update)

# Check final messages
print(graph.get_state(config).values["messages"])
```

### Working with Multi-Level Nested Subgraphs
```python
# Define a grandparent graph with the parent graph as a subgraph
class GrandfatherState(MessagesState):
    to_continue: bool

def router_node(state: GrandfatherState):
    # Dummy logic that will always continue
    return {"to_continue": True}

def route_after_prediction(state: GrandfatherState):
    if state["to_continue"]:
        return "graph"
    else:
        return END

grandparent_graph = StateGraph(GrandfatherState)
grandparent_graph.add_node(router_node)
grandparent_graph.add_node("graph", graph)
grandparent_graph.add_edge(START, "router_node")
grandparent_graph.add_conditional_edges(
    "router_node", route_after_prediction, ["graph", END]
)
grandparent_graph.add_edge("graph", END)
grandparent_graph = grandparent_graph.compile(checkpointer=MemorySaver())

# Run with nested subgraphs
config = {"configurable": {"thread_id": "2"}}
inputs = {"messages": [{"role": "user", "content": "what's the weather in sf"}]}
for update in grandparent_graph.stream(
    inputs, config=config, stream_mode="updates", subgraphs=True
):
    print(update)

# Get state at all levels
state = grandparent_graph.get_state(config, subgraphs=True)
print("Grandparent State:", state.values)
print("Parent Graph State:", state.tasks[0].state.values)
print("Subgraph State:", state.tasks[0].state.tasks[0].state.values)

# Update state in the deepest subgraph
grandparent_graph_state = state
parent_graph_state = grandparent_graph_state.tasks[0].state
subgraph_state = parent_graph_state.tasks[0].state
grandparent_graph.update_state(
    subgraph_state.config,
    {"messages": [{"role": "assistant", "content": "rainy"}]},
    as_node="weather_node",
)

# Resume execution
for update in grandparent_graph.stream(
    None, config=config, stream_mode="updates", subgraphs=True
):
    print(update)
```

## Benefits
- Enables human-in-the-loop intervention at precise points in complex graph execution
- Allows for dynamic state modification to handle edge cases and special requirements
- Supports debugging and testing by replaying from specific points in execution history
- Enables simulation of different paths through the graph for development or testing
- Provides visibility into complex nested state structures for monitoring and control

## Considerations
- State inspection and modification should preserve required state structure
- Proper configuration must be used when updating subgraph state
- Care needed to ensure coherence when acting as graph nodes
- Actions taken on behalf of nodes should maintain the expected format and content
- Using deeply nested subgraphs increases complexity of state management