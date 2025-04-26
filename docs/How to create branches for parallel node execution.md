# Creating Branches for Parallel Node Execution

## Overview
This guide demonstrates how to implement parallel node execution in LangGraph applications. You'll learn techniques for creating efficient workflows where multiple nodes can execute concurrently, process data in parallel paths, and rejoin results using reducers.

## Key Concepts
- **Parallel Execution**: Running multiple nodes concurrently within the same execution step
- **Fan-Out Pattern**: Distributing work from a single node to multiple downstream nodes
- **Fan-In Pattern**: Merging results from multiple nodes back into a unified state
- **Reducers**: Functions that specify how to combine state values from parallel branches
- **Conditional Branching**: Dynamic selection of execution paths based on state criteria

## Prerequisites
- LangGraph package installed
- Understanding of basic graph concepts (nodes, edges, state)
- Familiarity with TypedDict for state type definitions

```python
# Install required packages
pip install -U langgraph
```

## Implementation

### Basic Parallel Execution with Fan-Out and Fan-In
First, let's implement a simple graph with parallel branches:

```python
import operator
from typing_extensions import TypedDict
from typing import Annotated
from langgraph.graph import StateGraph, START, END

# Define state type with a reducer
class State(TypedDict):
    # The operator.add reducer combines lists from parallel branches
    aggregate: Annotated[list, operator.add]

# Define node functions
def a(state: State):
    print(f'Adding "A" to {state["aggregate"]}')
    return {"aggregate": ["A"]}

def b(state: State):
    print(f'Adding "B" to {state["aggregate"]}')
    return {"aggregate": ["B"]}

def c(state: State):
    print(f'Adding "C" to {state["aggregate"]}')
    return {"aggregate": ["C"]}

def d(state: State):
    print(f'Adding "D" to {state["aggregate"]}')
    return {"aggregate": ["D"]}

# Build graph with parallel branches
builder = StateGraph(State)
builder.add_node("a", a)
builder.add_node("b", b)
builder.add_node("c", c)
builder.add_node("d", d)

# Configure edges for parallel execution
builder.add_edge(START, "a")
builder.add_edge("a", "b")  # Fan-out from A to B
builder.add_edge("a", "c")  # Fan-out from A to C
builder.add_edge("b", "d")  # B feeds into D
builder.add_edge("c", "d")  # C feeds into D (fan-in)
builder.add_edge("d", END)

# Compile and run the graph
graph = builder.compile()
result = graph.invoke({"aggregate": []})
print(result)
```

When you run this graph, it produces:
```
Adding "A" to []
Adding "B" to ['A']
Adding "C" to ['A']
Adding "D" to ['A', 'B', 'C']
{'aggregate': ['A', 'B', 'C', 'D']}
```

The execution flow shows:
1. Node A executes first
2. Nodes B and C execute in parallel (fan-out)
3. Node D executes after both B and C complete (fan-in)
4. The reducer combines results from all branches in the `aggregate` list

### Creating Complex Branch Patterns
Let's extend our example with an additional node in one branch:

```python
# Add an intermediate node in the B branch
def b_2(state: State):
    print(f'Adding "B_2" to {state["aggregate"]}')
    return {"aggregate": ["B_2"]}

# Build a more complex graph
builder = StateGraph(State)
builder.add_node("a", a)
builder.add_node("b", b)
builder.add_node("b_2", b_2)  # New intermediate node
builder.add_node("c", c)
builder.add_node("d", d)

# Configure edges with branch of different length
builder.add_edge(START, "a")
builder.add_edge("a", "b")
builder.add_edge("a", "c")
builder.add_edge("b", "b_2")  # B leads to B_2
builder.add_edge(["b_2", "c"], "d")  # Both B_2 and C must complete before D
builder.add_edge("d", END)

# Compile and run
graph = builder.compile()
result = graph.invoke({"aggregate": []})
print(result)
```

When you run this more complex graph, it produces:
```
Adding "A" to []
Adding "B" to ['A']
Adding "C" to ['A']
Adding "B_2" to ['A', 'B', 'C']
Adding "D" to ['A', 'B', 'C', 'B_2']
{'aggregate': ['A', 'B', 'C', 'B_2', 'D']}
```

This demonstrates how branches can have different depths while still coordinating through fan-in.

### Implementing Conditional Branching
Combine parallel execution with conditional logic to dynamically select execution paths:

```python
from typing import Sequence

# Define an additional node
def e(state: State):
    print(f'Adding "E" to {state["aggregate"]}')
    return {"aggregate": ["E"]}

# Define routing function
def route_bc_or_cd(state: State) -> Sequence[str]:
    if state["which"] == "cd":
        return ["c", "d"]
    return ["b", "c"]

# Build graph with conditional branching
builder = StateGraph(State)
builder.add_node("a", a)
builder.add_node("b", b)
builder.add_node("c", c)
builder.add_node("d", d)
builder.add_node("e", e)

# Configure conditional routing
builder.add_edge(START, "a")
intermediates = ["b", "c", "d"]
builder.add_conditional_edges("a", route_bc_or_cd, intermediates)

# All branches eventually reach E
for node in intermediates:
    builder.add_edge(node, "e")
builder.add_edge("e", END)

# Compile the graph
graph = builder.compile()

# Test different paths
result_bc = graph.invoke({"aggregate": [], "which": "bc"})
print("BC Path Result:", result_bc)

result_cd = graph.invoke({"aggregate": [], "which": "cd"})
print("CD Path Result:", result_cd)
```

This graph can dynamically choose different paths:
```
# BC Path:
Adding "A" to []
Adding "B" to ['A']
Adding "C" to ['A']
Adding "E" to ['A', 'B', 'C']
BC Path Result: {'aggregate': ['A', 'B', 'C', 'E'], 'which': 'bc'}

# CD Path:
Adding "A" to []
Adding "C" to ['A']
Adding "D" to ['A']
Adding "E" to ['A', 'C', 'D']
CD Path Result: {'aggregate': ['A', 'C', 'D', 'E'], 'which': 'cd'}
```

## Usage Example
Here's a practical data processing workflow combining parallel execution and conditional branching:

```python
from typing_extensions import TypedDict, Annotated
from typing import Dict, List, Sequence
import operator
from langgraph.graph import StateGraph, START, END

# Define state with reducers
class ProcessingState(TypedDict):
    data: Dict
    results: Annotated[List, operator.add]  # Uses reducer to combine results
    path: str

# Define processing nodes
def load_data(state: ProcessingState):
    print("Loading data...")
    return {
        "data": {"numbers": [1, 2, 3, 4, 5], "text": "sample text"},
        "results": ["Data loaded"]
    }

def process_numbers(state: ProcessingState):
    print("Processing numbers...")
    numbers = state["data"]["numbers"]
    return {"results": [f"Numbers sum: {sum(numbers)}"]}

def process_text(state: ProcessingState):
    print("Processing text...")
    text = state["data"]["text"]
    return {"results": [f"Text length: {len(text)}"]}

def determine_path(state: ProcessingState) -> Sequence[str]:
    """Dynamically select processing branches based on state."""
    if state["path"] == "all":
        return ["process_numbers", "process_text"]  # Both branches
    elif state["path"] == "numbers":
        return ["process_numbers"]  # Numbers only
    else:
        return ["process_text"]  # Text only

def summarize(state: ProcessingState):
    print("Summarizing results...")
    return {"results": ["Summary: " + ", ".join(state["results"])]}

# Build the data processing graph
builder = StateGraph(ProcessingState)
builder.add_node("load_data", load_data)
builder.add_node("process_numbers", process_numbers)
builder.add_node("process_text", process_text)
builder.add_node("summarize", summarize)

# Configure the workflow
builder.add_edge(START, "load_data")
builder.add_conditional_edges(
    "load_data", 
    determine_path, 
    ["process_numbers", "process_text"]
)
builder.add_edge("process_numbers", "summarize")  # Both processing nodes
builder.add_edge("process_text", "summarize")     # feed into summarize
builder.add_edge("summarize", END)

# Compile and execute the graph
parallel_graph = builder.compile()

# Run the graph with all processing paths
result = parallel_graph.invoke({"data": {}, "results": [], "path": "all"})
print(result)

# Run the graph with numbers path only
numbers_result = parallel_graph.invoke({"data": {}, "results": [], "path": "numbers"})
print(numbers_result)

# Run the graph with text path only
text_result = parallel_graph.invoke({"data": {}, "results": [], "path": "text"})
print(text_result)
```

This will output:
```
# All paths:
Loading data...
Processing numbers...
Processing text...
Summarizing results...
{'data': {'numbers': [1, 2, 3, 4, 5], 'text': 'sample text'}, 'results': ['Data loaded', 'Numbers sum: 15', 'Text length: 11', 'Summary: Data loaded, Numbers sum: 15, Text length: 11'], 'path': 'all'}

# Numbers path only:
Loading data...
Processing numbers...
Summarizing results...
{'data': {'numbers': [1, 2, 3, 4, 5], 'text': 'sample text'}, 'results': ['Data loaded', 'Numbers sum: 15', 'Summary: Data loaded, Numbers sum: 15'], 'path': 'numbers'}

# Text path only:
Loading data...
Processing text...
Summarizing results...
{'data': {'numbers': [1, 2, 3, 4, 5], 'text': 'sample text'}, 'results': ['Data loaded', 'Text length: 11', 'Summary: Data loaded, Text length: 11'], 'path': 'text'}
```

## Benefits
- **Increased Performance**: Parallel execution reduces overall processing time
- **Better Resource Utilization**: Multiple tasks can run simultaneously
- **Dynamic Workflows**: Conditional branching enables adaptive execution paths
- **Modular Design**: Each branch can be developed and tested independently
- **Scalability**: Easily add new branches without redesigning the entire flow

## Considerations
- **State Management**: Ensure reducers correctly combine results from parallel branches
- **Execution Order**: Within parallel branches, execution order is not guaranteed
- **Debugging Complexity**: Parallel flows can be harder to debug than sequential ones
- **Resource Constraints**: Consider memory and processing limitations when designing parallel branches
- **Testing**: Thoroughly test all possible branch combinations

---

This transformation organizes content into a concise knowledge-base-friendly structure while preserving all technical details and code examples. Let me know if further refinements are needed!
