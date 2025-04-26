# Streaming in LangGraph

## Overview
Streaming is a critical feature for building responsive applications that provide real-time updates to users. LangGraph offers robust streaming capabilities for different types of data including workflow progress, LLM tokens, and custom updates, enabling developers to create engaging user experiences.

## Key Concepts
- Multiple streaming modes for different use cases
- Real-time feedback on graph execution
- LLM token streaming for natural interaction
- Custom update streaming for application-specific needs
- Debug streaming for development and troubleshooting

## Prerequisites
- Basic understanding of LangGraph
- Familiarity with asynchronous programming (for async streaming)
- Knowledge of graph execution flow

## Implementation

### Streaming Graph Outputs

LangGraph provides both synchronous and asynchronous methods for streaming graph outputs:

```python
# Synchronous streaming
for chunk in graph.stream(input_data, stream_mode="values"):
    # Process chunk

# Asynchronous streaming
async for chunk in graph.astream(input_data, stream_mode="updates"):
    # Process chunk asynchronously
```

### Streaming Modes

LangGraph supports several streaming modes that can be used individually or in combination:

1. **Values Mode**: Streams the complete state after each step
   ```python
   graph.stream(input_data, stream_mode="values")
   ```

2. **Updates Mode**: Streams only the changes to the state after each step
   ```python
   graph.stream(input_data, stream_mode="updates")
   ```

3. **Custom Mode**: Streams developer-defined data from within graph nodes
   ```python
   graph.stream(input_data, stream_mode="custom")
   ```

4. **Messages Mode**: Streams LLM tokens and metadata for nodes with LLM invocations
   ```python
   graph.stream(input_data, stream_mode="messages")
   ```

5. **Debug Mode**: Streams comprehensive debug information during execution
   ```python
   graph.stream(input_data, stream_mode="debug")
   ```

### Multiple Streaming Modes

You can combine streaming modes by passing them as a list:

```python
for chunk in graph.stream(input_data, stream_mode=["updates", "messages"]):
    # Each chunk will be a tuple: (stream_mode, data)
    mode, data = chunk
    if mode == "updates":
        # Process state updates
        pass
    elif mode == "messages":
        # Process LLM token streams
        pass
```

### Custom Streaming from Nodes

To stream custom data from within a node:

```python
def my_node(state, stream_writer=None):
    # Perform operations
    if stream_writer:
        stream_writer({"progress": "50%"})
    # Continue operations
    if stream_writer:
        stream_writer({"progress": "100%"})
    return {"result": "done"}

# When using the node in a graph:
graph.stream(input_data, stream_mode="custom")
```

## LangGraph Platform Integration

LangGraph Platform extends streaming capabilities with additional features:

- **values**: Streams full state after each super-step
- **messages-tuple**: Streams LLM tokens for messages generated inside nodes
- **updates**: Streams state updates after each node execution
- **debug**: Streams debug events throughout execution
- **events**: Streams all events during graph execution (useful for LCEL migration)

All platform events contain:
- **event**: Name of the event
- **data**: Associated data payload

## Usage Example

### Basic Streaming Example

```python
from langgraph.graph import StateGraph
from langgraph.prebuilt import ToolNode
from langchain_openai import ChatOpenAI

# Define state and nodes
# ...

# Create graph
builder = StateGraph(State)
builder.add_node("agent", agent_node)
builder.add_node("tools", ToolNode(tools))
# Add edges
# ...
graph = builder.compile()

# Stream with token updates for a chat interface
for chunk in graph.stream({"messages": [HumanMessage(content="Help me plan a trip")]}, 
                         stream_mode="messages"):
    # Display token by token to user
    print(chunk.content, end="", flush=True)
```

## Benefits
- Improved user experience with real-time feedback
- Reduced perceived latency during long-running operations
- Enhanced debugging capabilities during development
- Flexibility to choose streaming granularity based on needs
- Support for both synchronous and asynchronous applications

## Considerations
- Different streaming modes have different performance implications
- Multiple streaming modes may increase bandwidth usage
- Custom streaming requires careful implementation in nodes
- Debug streaming should be used primarily in development environments
- Platform-specific modes may have different semantics than library modes
