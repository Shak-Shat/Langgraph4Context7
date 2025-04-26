# LangGraph: Graph Components Reference

## Overview
This guide provides a comprehensive reference for the graph-based components in LangGraph, a library designed for building robust, stateful applications with language models. LangGraph allows you to create directed graphs where nodes perform operations and communicate by passing state information, enabling complex LLM-based workflows.

## Key Concepts
- **Graphs**: Directed structures composed of nodes and edges that define the flow of execution
- **Nodes**: Components that perform operations and update the shared state
- **Edges**: Connections between nodes that determine the flow of execution
- **State**: Shared information that flows through the graph and is modified by nodes
- **Reducers**: Functions that combine multiple values for the same state key
- **Streaming**: Multiple modes for observing intermediate results during graph execution
- **Checkpointing**: Mechanisms for persisting graph state for resumption

## Graph Types

### Graph
The base class for representing a directed graph in LangGraph.

```python
from langgraph.graph import StateGraph, START, END
```

### CompiledGraph
The executable form of a graph, created when a graph is compiled.

```python
# Define a graph
builder = StateGraph(State)
builder.add_node("a", lambda _state: {"another_list": ["hi"]})
builder.add_node("b", lambda _state: {"alist": ["there"]})
builder.add_edge("a", "b")
builder.add_edge(START, "a")

# Compile into an executable graph
graph = builder.compile()
```

### StateGraph
A specialized graph where nodes communicate by reading and writing to a shared state.

```python
from typing_extensions import Annotated, TypedDict
import operator

# Define a state schema with reducers
class State(TypedDict):
    alist: Annotated[list, operator.add]
    another_list: Annotated[list, operator.add]

# Create a state graph with the schema
builder = StateGraph(State)
```

## Essential Methods

### Adding Nodes

```python
# Add a node with a function
def my_node(state, config):
   return {"x": state["x"] + 1}

builder = StateGraph(dict)
builder.add_node(my_node)  # node name will be "my_node"

# Add a node with a custom name
builder.add_node("custom_name", my_node)
```

### Adding Edges

```python
# Simple edge from one node to another
builder.add_edge("node_a", "node_b")

# Edge from START node
builder.add_edge(START, "first_node")

# Edge to END node
builder.add_edge("last_node", END)

# Edge from multiple source nodes to a single destination
builder.add_edge(["node_a", "node_b"], "node_c")  # node_c runs after both node_a and node_b complete
```

### Adding Conditional Edges

```python
# Define a conditional router
def route(state):
    if state["condition"]:
        return "node_a"
    else:
        return "node_b"

# Add a conditional edge
builder.add_conditional_edges("decision_node", route)

# With a path map
builder.add_conditional_edges(
    "decision_node",
    lambda s: "true" if s["condition"] else "false",
    {"true": "node_a", "false": "node_b"}
)

# With an END condition
def route_with_end(state):
    if state["done"]:
        return END
    else:
        return "continue_node"

builder.add_conditional_edges("check_done", route_with_end)
```

### Setting Entry and Exit Points

```python
# Set the first node (equivalent to add_edge(START, "first_node"))
builder.set_entry_point("first_node")

# Set a finish point (equivalent to add_edge("last_node", END))
builder.set_finish_point("last_node")

# Set a conditional entry point
def choose_entry(input_data):
    if input_data["type"] == "query":
        return "query_handler"
    else:
        return "command_handler"

builder.set_conditional_entry_point(choose_entry)
```

### Adding Sequences

```python
# Add a sequence of nodes
builder.add_sequence([
    "step1", 
    "step2", 
    "step3"
])

# With function definitions
builder.add_sequence([
    ("parse", parse_function),
    ("process", process_function),
    ("respond", respond_function)
])
```

### Compiling Graphs

```python
from langgraph.checkpoint.memory import MemorySaver

# Create a memory checkpointer
memory = MemorySaver()

# Compile with checkpointing
graph = builder.compile(checkpointer=memory)

# Compile with interruption points
graph = builder.compile(
    interrupt_before=["critical_node"],
    interrupt_after=["review_node"]
)
```

## Streaming Graph Execution

### Stream Modes

LangGraph provides several streaming modes to monitor execution:

- **values**: Emits complete state after each step
- **updates**: Emits only changes made by each node
- **custom**: Emits custom data using StreamWriter
- **messages**: Emits LLM messages token-by-token with metadata
- **debug**: Emits detailed events for each step

### Example: Values Mode

```python
# Stream complete state after each step
for event in graph.stream(
    {"alist": ["initial"]}, 
    stream_mode="values"
):
    print(event)

# Output:
# {"alist": ["initial"], "another_list": []}
# {"alist": ["initial"], "another_list": ["hi"]}
# {"alist": ["initial", "there"], "another_list": ["hi"]}
```

### Example: Updates Mode

```python
# Stream only updates from each node
for event in graph.stream(
    {"alist": ["initial"]}, 
    stream_mode="updates"
):
    print(event)

# Output:
# {"a": {"another_list": ["hi"]}}
# {"b": {"alist": ["there"]}}
```

### Example: Custom Streaming

```python
from langgraph.types import StreamWriter

def node_a(state: State, writer: StreamWriter):
    writer({"custom_data": "foo"})  # Emit custom data
    return {"alist": ["hi"]}

builder = StateGraph(State)
builder.add_node("a", node_a)
builder.add_edge(START, "a")
graph = builder.compile()

for event in graph.stream(
    {"alist": ["initial"]}, 
    stream_mode="custom"
):
    print(event)

# Output:
# {"custom_data": "foo"}
```

### Example: Message Streaming

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")

class State(TypedDict):
    question: str
    answer: str

def node_a(state: State):
    response = llm.invoke(state["question"])
    return {"answer": response.content}

builder = StateGraph(State)
builder.add_node("a", node_a)
builder.add_edge(START, "a")
graph = builder.compile()

for event in graph.stream(
    {"question": "What is the capital of France?"}, 
    stream_mode="messages"
):
    print(event[0].content, end="")  # Print just the content

# Output: The capital of France is Paris
```

## Configuration and State Management

### Configuration Schema

```python
class ConfigSchema(TypedDict):
    model: Optional[str]
    temperature: Optional[float]

graph = StateGraph(State, config_schema=ConfigSchema)

# Using configuration
def node_with_config(state: State, config: RunnableConfig):
    model_name = config["configurable"].get("model", "default-model")
    temp = config["configurable"].get("temperature", 0.7)
    # Use model and temperature...
    return {"result": "processed"}

# Invoke with configuration
result = graph.invoke(
    {"input": "query"}, 
    {"configurable": {"model": "gpt-4", "temperature": 0.2}}
)
```

### Checkpoint Management

```python
# Get the current state
state = graph.get_state(thread_config)

# Check pending executions
pending = state.next
print("Waiting on:", pending)  # e.g., ("human_review_node",)

# Update state
graph.update_state(
    thread_config, 
    {"new_value": 123},
    as_node="processing_node"
)
```

## Best Practices
- Define state schemas with appropriate reducers for combining values
- Use conditional edges to create dynamic execution paths
- Leverage streaming for real-time monitoring of execution
- Implement checkpointing for long-running or interactive applications
- Use configuration schemas to make graphs flexible and reusable
- Consider human-in-the-loop designs with strategic interruption points

## Considerations
- Graph complexity increases with the number of conditional edges
- State design impacts performance and maintainability
- Proper error handling should be implemented in node functions
- Carefully manage references to external resources in graph nodes
