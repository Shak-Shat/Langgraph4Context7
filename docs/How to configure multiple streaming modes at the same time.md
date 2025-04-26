# Configuring Multiple Streaming Modes in LangGraph

## Overview
This guide demonstrates how to configure multiple streaming modes simultaneously in LangGraph applications. LangGraph offers various streaming options that allow you to receive different types of updates during graph execution, from incremental messages to complete state snapshots.

## Key Concepts
- **Stream Modes**: Different ways to receive updates from a running LangGraph workflow
- **Values Streaming**: Stream the output values from each node
- **Updates Streaming**: Stream incremental state updates as they occur
- **Full State Streaming**: Stream the complete graph state after each update
- **Messages Streaming**: Stream specifically the message content of nodes

## Prerequisites
- LangGraph package installed
- Basic understanding of LangGraph's StateGraph and graph execution
- A working LangGraph application

```python
# Install required packages
pip install -U langgraph langchain-openai

# Set up environment variables (if needed)
import os
os.environ["OPENAI_API_KEY"] = "your-api-key"
```

## Implementation

### Creating a Basic Graph
First, let's create a simple graph to demonstrate streaming:

```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langchain_openai import ChatOpenAI

# Define our state schema
class State(TypedDict):
    messages: list
    count: int

# Create a simple node function
def llm_node(state: State):
    model = ChatOpenAI()
    messages = state["messages"]
    response = model.invoke(messages)
    return {"messages": [response], "count": state["count"] + 1}

def counter_node(state: State):
    return {"count": state["count"] + 1}

# Build the graph
builder = StateGraph(State)
builder.add_node("llm", llm_node)
builder.add_node("counter", counter_node)
builder.add_edge(START, "llm")
builder.add_edge("llm", "counter")
builder.add_edge("counter", END)

# Compile the graph
graph = builder.compile()
```

### Streaming Multiple Modes Simultaneously
Use the `ConfigurableFieldSpec` to set up multiple streaming modes:

```python
from langgraph.prebuilt import ConfigurableFieldSpec
from langgraph.checkpoint import MemorySaver

# Initialize a memory saver for checkpointing
memory_saver = MemorySaver()

# Configure the graph with multiple streaming options
configurable_graph = graph.configurable_fields(
    streaming_options=ConfigurableFieldSpec(
        id="streaming_options",
        annotation=dict,
        default={
            "values": True,       # Stream output values
            "updates": True,      # Stream incremental updates
            "full_state": True,   # Stream the full state
            "messages": True      # Stream message content only
        },
        description="Streaming options configuration"
    ),
    checkpointer=ConfigurableFieldSpec(
        id="checkpointer",
        annotation=object,
        default=memory_saver,
        description="Checkpointer to use for persistence"
    )
)
```

### Using a Custom Stream Handler
Create a handler to process multiple stream types:

```python
class MultiStreamHandler:
    def __init__(self):
        self.values = []
        self.updates = []
        self.states = []
        self.messages = []
    
    def handle_stream(self, chunk, stream_type):
        if stream_type == "values":
            self.values.append(chunk)
            print(f"VALUES: {chunk}")
        elif stream_type == "updates":
            self.updates.append(chunk)
            print(f"UPDATES: {chunk}")
        elif stream_type == "full_state":
            self.states.append(chunk)
            print(f"FULL STATE: {chunk}")
        elif stream_type == "messages":
            if "messages" in chunk:
                self.messages.append(chunk["messages"][-1])
                print(f"MESSAGE: {chunk['messages'][-1].content}")

# Create an instance of the handler
handler = MultiStreamHandler()
```

### Running With Multiple Stream Modes
Execute the graph with multiple streaming configurations:

```python
# Input to the graph
input_state = {
    "messages": [{"role": "user", "content": "Tell me a short joke"}],
    "count": 0
}

# Run with values streaming
for chunk in configurable_graph.stream(
    input_state,
    stream_mode="values",
    configurable={
        "streaming_options": {"values": True, "updates": False, "full_state": False, "messages": False}
    }
):
    handler.handle_stream(chunk, "values")

# Run with updates streaming
thread_id = "updates_stream_example"
for chunk in configurable_graph.stream(
    input_state,
    stream_mode="updates",
    config={"thread_id": thread_id},
    configurable={
        "streaming_options": {"values": False, "updates": True, "full_state": False, "messages": False}
    }
):
    handler.handle_stream(chunk, "updates")

# Run with full state streaming
thread_id = "full_state_stream_example"
for chunk in configurable_graph.stream(
    input_state,
    stream_mode="full_state",
    config={"thread_id": thread_id},
    configurable={
        "streaming_options": {"values": False, "updates": False, "full_state": True, "messages": False}
    }
):
    handler.handle_stream(chunk, "full_state")
```

### Combining Multiple Stream Modes in One Call
Stream multiple modes simultaneously:

```python
# Run with multiple streaming modes at once
thread_id = "combined_stream_example"
for chunk in configurable_graph.stream(
    input_state,
    stream_mode="updates",  # Primary mode
    config={"thread_id": thread_id},
    configurable={
        "streaming_options": {
            "values": True,
            "updates": True,
            "full_state": True,
            "messages": True
        }
    }
):
    # Check what kind of chunk we received
    if "node_name" in chunk and "node_type" in chunk:
        handler.handle_stream(chunk, "updates")
    elif "messages" in chunk:
        handler.handle_stream(chunk, "messages")
    elif isinstance(chunk, dict) and all(k in chunk for k in ["messages", "count"]):
        handler.handle_stream(chunk, "values")
    else:
        handler.handle_stream(chunk, "full_state")
```

## Usage Example
Here's a complete example that demonstrates combining all streaming modes in a conversational agent:

```python
from langchain_core.messages import HumanMessage, AIMessage
from langgraph.prebuilt import ToolNode
from langchain_core.tools import tool

# Define a simple tool
@tool
def get_weather(location: str) -> str:
    """Get the weather for a location."""
    return f"The weather in {location} is sunny and 75 degrees."

# Create a tool node
tools = [get_weather]
tool_node = ToolNode(tools)

# Build an agent graph
def agent_node(state):
    messages = state["messages"]
    model = ChatOpenAI(model="gpt-4").bind_tools(tools)
    response = model.invoke(messages)
    return {"messages": [response]}

# Create a directed graph with branching
builder = StateGraph(State)
builder.add_node("agent", agent_node)
builder.add_node("tools", tool_node)
builder.add_edge(START, "agent")

# Add conditional routing
def should_continue(state):
    last_message = state["messages"][-1]
    if hasattr(last_message, "tool_calls") and last_message.tool_calls:
        return "tools"
    return END

builder.add_conditional_edges("agent", should_continue, {
    "tools": "tools",
    END: END
})
builder.add_edge("tools", "agent")

# Compile the graph
agent_graph = builder.compile()

# Configure with multiple streaming options
configurable_agent = agent_graph.configurable_fields(
    streaming_options=ConfigurableFieldSpec(
        id="streaming_options",
        annotation=dict,
        default={
            "values": True,
            "updates": True,
            "full_state": True,
            "messages": True
        },
        description="Streaming options configuration"
    )
)

# Execute with all streaming modes active
input_messages = {"messages": [HumanMessage(content="What's the weather in San Francisco?")], "count": 0}

print("Running the agent with all streaming modes:")
for chunk in configurable_agent.stream(input_messages, stream_mode="updates"):
    # Process the chunks based on their content
    if isinstance(chunk, dict) and "node_name" in chunk:
        print(f"UPDATE from {chunk['node_name']}")
    elif "messages" in chunk and chunk["messages"]:
        last_message = chunk["messages"][-1]
        if hasattr(last_message, "content"):
            print(f"MESSAGE: {last_message.content[:50]}..." if len(last_message.content) > 50 else last_message.content)
```

## Benefits
- Receive different types of information from your graph as it executes
- Build responsive UIs that show incremental progress
- Debug complex workflows by observing the complete state
- Optimize network usage by selecting only the streaming modes you need
- Combine streaming modes for rich, interactive applications

## Considerations
- Select the primary stream mode that best fits your application's needs
- Be aware of potential performance impacts when enabling multiple stream modes
- Implement proper handlers for each streaming type
- Use thread IDs to maintain consistency across streaming sessions
- Consider persistence needs when streaming in production applications

---

This transformed document retains all necessary technical details, organizes information clearly, and follows a structured hierarchy for knowledge-base-friendly presentation. Let me know if further refinement is required!
