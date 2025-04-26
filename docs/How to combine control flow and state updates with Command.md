# Combining Control Flow and State Updates with Command

## Overview
This guide demonstrates how to use the `Command` object in LangGraph to simultaneously control execution flow and update state within graph nodes. This approach streamlines complex workflows by combining routing decisions and state modifications into a single operation.

## Key Concepts
- **Command Object**: A special return type that combines state updates with routing instructions
- **Dynamic Routing**: Conditional navigation between nodes based on runtime decisions
- **Cross-Graph Navigation**: Moving between parent graphs and subgraphs
- **State Reducers**: Functions that resolve state updates when combining parent and subgraph states

## Prerequisites
- LangGraph package installed
- Basic understanding of StateGraph and node structure
- Familiarity with TypedDict for state typing

```python
# Install required packages
pip install -U langgraph

# Optional: Set up LangSmith for debugging
import os
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "your_langsmith_api_key"
```

## Implementation

### Basic Command Usage
Use the Command object to update state and specify the next node in a single operation:

```python
import random
from typing_extensions import TypedDict, Literal
from langgraph.graph import StateGraph, START
from langgraph.types import Command

# Define the graph state
class State(TypedDict):
    foo: str

# Node that uses Command to determine routing and update state
def node_a(state: State) -> Command[Literal["node_b", "node_c"]]:
    # Randomly choose a path
    value = random.choice(["a", "b"])
    
    # Decide which node to go to next
    goto = "node_b" if value == "a" else "node_c"
    
    # Return a Command that both updates state and specifies next node
    return Command(update={"foo": value}, goto=goto)

# Simple nodes that just update state
def node_b(state: State):
    return {"foo": state["foo"] + "b"}

def node_c(state: State):
    return {"foo": state["foo"] + "c"}

# Build the graph without conditional edges
builder = StateGraph(State)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_node("node_c", node_c)
builder.add_edge(START, "node_a")
graph = builder.compile()
```

### Cross-Graph Navigation
Navigate between parent graphs and subgraphs using Command.PARENT:

```python
import operator
from typing_extensions import Annotated

# Define a state with a reducer for the 'foo' field
class SharedState(TypedDict):
    foo: Annotated[str, operator.add]

# Subgraph node that navigates to the parent graph
def subgraph_node_a(state: SharedState) -> Command[Literal["node_b", "node_c"]]:
    value = random.choice(["a", "b"])
    
    # Determine which node in the parent graph to go to
    goto = "node_b" if value == "a" else "node_c"
    
    # Command with graph=PARENT to exit the subgraph
    return Command(
        update={"foo": value}, 
        goto=goto, 
        graph=Command.PARENT  # This is the key part for cross-graph navigation
    )

# Create the subgraph
subgraph_builder = StateGraph(SharedState)
subgraph_builder.add_node("node_a", subgraph_node_a)
subgraph_builder.add_edge(START, "node_a")
subgraph = subgraph_builder.compile()

# Define parent graph nodes
def parent_node_b(state: SharedState):
    return {"foo": state["foo"] + "b"}

def parent_node_c(state: SharedState):
    return {"foo": state["foo"] + "c"}

# Create the parent graph and add the subgraph
parent_builder = StateGraph(SharedState)
parent_builder.add_node("subgraph", subgraph)
parent_builder.add_node("node_b", parent_node_b)
parent_builder.add_node("node_c", parent_node_c)
parent_builder.add_edge(START, "subgraph")
parent_graph = parent_builder.compile()
```

### Advanced Routing with Multiple Conditions
Create more complex routing logic with multiple conditions:

```python
from enum import Enum

class RouteOptions(str, Enum):
    OPTION_A = "option_a"
    OPTION_B = "option_b"
    OPTION_C = "option_c"
    END = "end"

class ComplexState(TypedDict):
    count: int
    value: str
    
def decision_node(state: ComplexState) -> Command[RouteOptions]:
    count = state["count"] + 1
    
    # Complex routing logic
    if count > 5:
        return Command(
            update={"count": count, "value": "finished"},
            goto=RouteOptions.END
        )
    elif count % 3 == 0:
        return Command(
            update={"count": count, "value": "divisible by 3"},
            goto=RouteOptions.OPTION_C
        )
    elif count % 2 == 0:
        return Command(
            update={"count": count, "value": "even"},
            goto=RouteOptions.OPTION_B
        )
    else:
        return Command(
            update={"count": count, "value": "odd"},
            goto=RouteOptions.OPTION_A
        )

# Define simple processing nodes
def option_a(state: ComplexState):
    return {"value": f"Processed option A: {state['value']}"}

def option_b(state: ComplexState):
    return {"value": f"Processed option B: {state['value']}"}

def option_c(state: ComplexState):
    return {"value": f"Processed option C: {state['value']}"}

# Build complex routing graph
complex_builder = StateGraph(ComplexState)
complex_builder.add_node("decision", decision_node)
complex_builder.add_node("option_a", option_a)
complex_builder.add_node("option_b", option_b)
complex_builder.add_node("option_c", option_c)
complex_builder.add_edge(START, "decision")
complex_builder.add_edge("option_a", "decision")
complex_builder.add_edge("option_b", "decision")
complex_builder.add_edge("option_c", "decision")
complex_graph = complex_builder.compile()
```

## Usage Example
Here's how to use the graphs defined above:

```python
# Basic Command example
result = graph.invoke({"foo": ""})
print(f"Basic Command result: {result}")
# Possible outputs:
# {"foo": "ab"} - if node_a chose "a" and went to node_b
# {"foo": "bc"} - if node_a chose "b" and went to node_c

# Cross-graph navigation example
parent_result = parent_graph.invoke({"foo": ""})
print(f"Cross-graph navigation result: {parent_result}")
# Possible outputs:
# {"foo": "ab"} - if subgraph_node_a chose "a" and went to parent's node_b
# {"foo": "bc"} - if subgraph_node_a chose "b" and went to parent's node_c

# Complex routing example
from pprint import pprint
complex_result = complex_graph.invoke({"count": 0, "value": "initial"})
pprint(complex_result)
# This will cycle through options a, b, c based on the counter value
# until count > 5, then it will stop at "end"
```

### Tracing Execution Flow
To understand complex routing, you can track the execution flow:

```python
def trace_execution(graph, initial_state, max_steps=10):
    state = initial_state
    step = 0
    
    print(f"Initial state: {state}")
    
    while step < max_steps:
        step += 1
        try:
            result = graph.invoke(state)
            print(f"Step {step}: {result}")
            state = result
        except Exception as e:
            print(f"Error at step {step}: {e}")
            break
    
    print(f"Final state: {state}")

# Trace the complex graph's execution
trace_execution(complex_graph, {"count": 0, "value": "initial"})
```

## Benefits
- Simplifies conditional logic by combining state updates with routing decisions
- Eliminates the need for separate conditional edge definitions
- Enables dynamic routing based on runtime state
- Facilitates seamless navigation between subgraphs and parent graphs
- Supports complex workflow orchestration with minimal boilerplate

## Considerations
- Use TypedDict annotations to ensure type safety in Command routing
- Define reducers for state fields that may be updated by both parent and subgraph
- Consider using enums for routing options to improve code clarity
- Test complex routing logic thoroughly to ensure all paths work as expected
- Use LangSmith tracing for debugging complex workflows