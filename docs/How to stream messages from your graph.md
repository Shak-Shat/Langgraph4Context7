# Streaming Messages from a LangGraph

## Overview
This document explains how to stream individual LLM tokens as they are generated within a LangGraph execution. Using `stream_mode="messages-tuple"` allows real-time access to tokens from any chat model invocations inside your graph nodes.

## Key Concepts
- `stream_mode="messages-tuple"` streams individual LLM tokens as they're generated
- Messages are returned as tuples containing the message and metadata
- Metadata includes graph node information, allowing you to filter by source
- Useful for creating responsive UIs with real-time generation feedback

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

### Streaming in Messages-Tuple Mode
Stream individual message tokens from LLM calls inside graph nodes:

```python
input = {"messages": [{"role": "user", "content": "what's the weather in sf"}]}
config = {"configurable": {"model_name": "openai"}}

async for chunk in client.runs.stream(
    thread["thread_id"],
    assistant_id=assistant_id,
    input=input,
    config=config,
    stream_mode="messages-tuple",
):
    print(f"Receiving new event of type: {chunk.event}...")
    print(chunk.data)
    print("\n\n")
```

## Usage Example
When streaming in messages-tuple mode, you'll receive various types of events:

1. Initially, a metadata event with run information
2. Then, a series of message events containing:
   - LLM tokens as they're generated (e.g., partial words in tool calls)
   - Metadata with graph_id and langgraph_node identifying the source node
3. Tool responses as they're received
4. Final AI response tokens as they're generated

Messages are streamed as tuples containing both the content and metadata, allowing you to:
- Identify which node generated each token
- Show real-time typing effects in your UI
- Filter messages by source or type

## Benefits
- Creates responsive, real-time user experiences
- Provides visibility into token-by-token generation
- Enables selective rendering based on node source
- Allows progressive rendering of tool calls and responses

## Considerations
- Processing partial tokens requires additional client-side logic
- Message chunks must be assembled to reconstruct complete messages
- Different event types require appropriate handling
- Metadata should be used to route messages to appropriate UI components
