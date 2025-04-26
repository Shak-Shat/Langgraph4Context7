# Streaming State Updates in LangGraph

## Overview
This document demonstrates how to stream incremental updates to your graph state as nodes execute, rather than streaming the entire state value. This approach allows for more efficient and focused monitoring of state changes during graph execution.

## Key Concepts
- `stream_mode="updates"` returns only the changes made by each node
- Updates are streamed after each node completes execution
- More efficient than `stream_mode="values"` for larger state objects
- Useful for UI updates and monitoring specific state changes

## Prerequisites
```python
from langgraph_sdk import get_client
```

## Implementation

### Setting Up the Client and Thread
First, establish a connection to your LangGraph deployment and create a thread:

```python
client = get_client(url=<DEPLOYMENT_URL>)

# Using the graph deployed with the name "agent"
assistant_id = "agent"

# Create thread
thread = await client.threads.create()
print(thread)
```

### Streaming Updates Mode
Stream only the changes made by each node after execution:

```python
input = {
    "messages": [
        {
            "role": "user",
            "content": "what's the weather in la"
        }
    ]
}

async for chunk in client.runs.stream(
    thread["thread_id"],
    assistant_id,
    input=input,
    stream_mode="updates",
):
    print(f"Receiving new event of type: {chunk.event}...")
    print(chunk.data)
    print("\n\n")
```

## Usage Example
When you run the above code with a weather query, you'll receive a series of updates:

1. First, a metadata event with the run ID
2. Then, updates from the agent node with tool calls
3. Next, updates from the action node with tool responses
4. Finally, updates with the agent's response to the user

Each update contains only the specific changes made by that node, making it easier to track the progression of state changes.

## Benefits
- Reduced data transfer compared to full state streaming
- Easier to track specific changes in complex states
- More efficient for UI updates in client applications
- Better visibility into the node-by-node execution flow

## Considerations
- You'll need to reconstruct the full state if required
- Updates are specific to the node that just executed
- Event types include: metadata, updates, and end
- The client needs to handle different event types appropriately
