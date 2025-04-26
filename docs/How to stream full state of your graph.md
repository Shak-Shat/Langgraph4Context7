# Streaming Full State in LangGraph

## Overview
This document explains how to stream the complete state of a LangGraph at each execution step. Using `stream_mode="values"` provides a snapshot of the entire graph state after each node completes execution, unlike the incremental updates approach.

## Key Concepts
- `stream_mode="values"` streams the complete state after each superstep
- Each stream event contains the entire graph state, not just changes
- Useful for maintaining a complete view of the graph state
- Allows tracking the evolution of the full state during execution

## Prerequisites
```python
from langgraph_sdk import get_client
```

## Implementation

### Setting Up the Client and Thread
First, establish a connection to your LangGraph deployment:

```python
client = get_client(url=<DEPLOYMENT_URL>)

# Using the graph deployed with the name "agent"
assistant_id = "agent"

# Create thread
thread = await client.threads.create()
```

### Streaming in Values Mode
Stream the complete state after each node execution:

```python
input = {"messages": [{"role": "user", "content": "what's the weather in la"}]}

# Stream values
async for chunk in client.runs.stream(
    thread["thread_id"],
    assistant_id, 
    input=input,
    stream_mode="values"
):
    print(f"Receiving new event of type: {chunk.event}...")
    print(chunk.data)
    print("\n\n")
```

### Retrieving Only the Final State
If you only need the final state after execution completes:

```python
final_answer = None
async for chunk in client.runs.stream(
    thread["thread_id"],
    assistant_id,
    input=input,
    stream_mode="values"
):
    if chunk.event == "values":
        final_answer = chunk.data
```

## Usage Example
When streaming in values mode, you'll receive a sequence of state snapshots:

1. Initial state with just the user message
2. State after the agent node runs (including tool calls)
3. State after tool execution (including tool responses)
4. Final state with the completed AI response

Each snapshot provides the complete graph state at that point, showing how the state evolves through the execution process.

## Benefits
- Complete visibility into the graph state at each step
- Simplifies client-side state management
- Eliminates need to reconstruct state from incremental updates
- Useful for debugging and monitoring complex graph execution

## Considerations
- Increased data transfer compared to updates-only streaming
- Redundant information when state is large but changes are small
- May consume more bandwidth for complex states
- Best for applications needing the full state context at each step
