# Persistence in LangGraph

## Overview
Persistence in LangGraph enables maintaining state across graph executions through checkpointers and memory stores. This capability allows for powerful features like conversation history, time travel (replaying past executions), and human-in-the-loop interactions.

## Key Concepts
- **Checkpointers**: Save snapshots of graph state at each step
- **Threads**: Unique identifiers for sequences of checkpoints
- **StateSnapshot**: Objects containing state values, next nodes, and metadata
- **Memory Store**: Repository for cross-thread data storage and retrieval
- **Replay**: Re-executing a graph from a specific checkpoint

## Prerequisites
```python
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver
from langgraph.store.memory import InMemoryStore
from typing import Annotated
from typing_extensions import TypedDict
from operator import add
import uuid
from langchain.embeddings import init_embeddings
```

## Implementation

### Basic Checkpointing
Checkpointers save snapshots of graph state at each step. Here's a simple example:

```python
class State(TypedDict):
    foo: int
    bar: Annotated[list[str], add]

def node_a(state: State):
    return {"foo": "a", "bar": ["a"]}

def node_b(state: State):
    return {"foo": "b", "bar": ["b"]}

workflow = StateGraph(State)
workflow.add_node(node_a)
workflow.add_node(node_b)
workflow.add_edge(START, "node_a")
workflow.add_edge("node_a", "node_b")
workflow.add_edge("node_b", END)

checkpointer = MemorySaver()
graph = workflow.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "1"}}
graph.invoke({"foo": ""}, config)
```

### Working with StateSnapshots
After graph execution, you can access the saved state:

```python
# Get the latest state snapshot
config = {"configurable": {"thread_id": "1"}}
graph.get_state(config)

# Get a specific checkpoint
config = {"configurable": {"thread_id": "1", "checkpoint_id": "1ef663ba-28fe-6528-8002-5a559208592c"}}
graph.get_state(config)

# Get full history of checkpoints
config = {"configurable": {"thread_id": "1"}}
list(graph.get_state_history(config))
```

### Replaying and Forking Execution
You can replay previous executions or fork from a specific checkpoint:

```python
# Replay from a specific checkpoint
config = {"configurable": {"thread_id": "1", "checkpoint_id": "0c62ca34-ac19-445d-bbb0-5b4984975b2a"}}
graph.invoke(None, config=config)
```

### Updating State
You can manually update the state using `update_state()`:

```python
# Update current state or fork from a specific checkpoint
graph.update_state(
    config,                  # Config with thread_id and optional checkpoint_id
    {"foo": 2, "bar": ["b"]} # Values to update
)
```

Note that updates respect channel reducers. For example, with:
```python
class State(TypedDict):
    foo: int
    bar: Annotated[list[str], add]
```

If the current state is `{"foo": 1, "bar": ["a"]}` and you update with `{"foo": 2, "bar": ["b"]}`, the new state will be `{"foo": 2, "bar": ["a", "b"]}` because `bar` has the `add` reducer.

### Memory Store for Cross-Thread Information
Memory Stores allow sharing information across threads:

```python
# Create an in-memory store
in_memory_store = InMemoryStore()

# Define namespace for a user's memories
user_id = "1"
namespace_for_memory = (user_id, "memories")

# Store a memory
memory_id = str(uuid.uuid4())
memory = {"food_preference": "I like pizza"}
in_memory_store.put(namespace_for_memory, memory_id, memory)

# Retrieve memories
memories = in_memory_store.search(namespace_for_memory)
```

### Semantic Search in Memory Store
You can enable semantic search by configuring an embedding model:

```python
store = InMemoryStore(
    index={
        "embed": init_embeddings("openai:text-embedding-3-small"),
        "dims": 1536,
        "fields": ["food_preference", "$"]
    }
)

# Search memories semantically
memories = store.search(
    namespace_for_memory,
    query="What does the user like to eat?",
    limit=3
)
```

### Using Memory Store in Graph Nodes
You can access and update the store from within graph nodes:

```python
def update_memory(state: MessagesState, config: RunnableConfig, *, store: BaseStore):
    # Get the user id from the config
    user_id = config["configurable"]["user_id"]
    
    # Namespace the memory
    namespace = (user_id, "memories")
    
    # Create a new memory ID
    memory_id = str(uuid.uuid4())
    
    # Store a new memory
    store.put(namespace, memory_id, {"memory": memory})

def call_model(state: MessagesState, config: RunnableConfig, *, store: BaseStore):
    # Get the user id from the config
    user_id = config["configurable"]["user_id"]
    
    # Search based on the most recent message
    memories = store.search(
        namespace,
        query=state["messages"][-1].content,
        limit=3
    )
    info = "\n".join([d.value["memory"] for d in memories])
    
    # Use memories in the model call
```

## Usage Example
To implement persistence in your LangGraph application:

1. Create a checkpointer for thread-level persistence:
   ```python
   checkpointer = MemorySaver()  # Or use SqliteSaver or PostgresSaver for production
   ```

2. Add a memory store for cross-thread information:
   ```python
   store = InMemoryStore(index={"embed": embedding_function, "dims": dimensions})
   ```

3. Compile your graph with both:
   ```python
   graph = workflow.compile(checkpointer=checkpointer, store=store)
   ```

4. Invoke with thread and user identifiers:
   ```python
   config = {"configurable": {"thread_id": "thread-123", "user_id": "user-456"}}
   graph.invoke(input_values, config)
   ```

## Benefits
- Enables conversation history across multiple interactions
- Supports human-in-the-loop workflows through state inspection
- Provides time travel debugging capabilities
- Offers fault tolerance with automatic recovery
- Allows cross-conversation memory sharing

## Considerations
- Different checkpointer implementations have different persistence characteristics
- Memory usage increases with number of checkpoints stored
- For production use, consider database-backed implementations
- Semantic search requires additional configuration
- Checkpointing has performance implications
