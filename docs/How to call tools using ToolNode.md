# Using ToolNode for Tool Calling in LangGraph

## Overview
This guide demonstrates how to implement and use LangGraph's `ToolNode` to efficiently execute tool calls within structured agent workflows. `ToolNode` simplifies the process of connecting language models with external tools and handling their responses within a graph structure.

## Key Concepts
- **ToolNode**: A specialized node that processes and executes tool calls from language models
- **Parallel Tool Calls**: Ability to execute multiple tools simultaneously
- **Tool Integration**: Seamless connection between language models and external functionality
- **State Management**: Maintaining context through message-based state updates

## Prerequisites
- LangGraph and LangChain packages installed
- Language model API keys configured
- Basic understanding of tools and ReAct agents

```python
# Install required packages
pip install -U langgraph langchain_anthropic

# Set up API key
import os, getpass

def set_env(var: str):
    if not os.environ.get(var):
        os.environ[var] = getpass.getpass(f"{var}: ")

set_env("ANTHROPIC_API_KEY")
```

## Implementation

### Defining Custom Tools
First, create the tools that will be used in your application:

```python
from langchain_core.tools import tool

@tool
def get_weather(location: str):
    """Call to get the current weather."""
    if location.lower() in ["sf", "san francisco"]:
        return "It's 60 degrees and foggy."
    else:
        return "It's 90 degrees and sunny."

@tool
def get_coolest_cities():
    """Get a list of coolest cities."""
    return "nyc, sf"
```

### Creating a ToolNode
Register your tools with a `ToolNode` to enable processing of tool calls:

```python
from langgraph.prebuilt import ToolNode

# Create a collection of tools
tools = [get_weather, get_coolest_cities]

# Create a ToolNode with these tools
tool_node = ToolNode(tools)
```

### Using ToolNode Directly
You can manually invoke the ToolNode with AIMessages containing tool calls:

```python
from langchain_core.messages import AIMessage

# Create an AI message with a tool call
message_with_tool_call = AIMessage(
    content="",
    tool_calls=[{
        "name": "get_weather",
        "args": {"location": "sf"},
        "id": "tool_call_id",
        "type": "tool_call",
    }]
)

# Invoke the ToolNode with the message
output = tool_node.invoke({"messages": [message_with_tool_call]})
print(output)
# Output:
# {'messages': [ToolMessage(content="It's 60 degrees and foggy.", name='get_weather', tool_call_id='tool_call_id')]}
```

### Handling Multiple Tool Calls
ToolNode can process multiple tool calls in a single invocation:

```python
# Create an AI message with multiple tool calls
message_with_multiple_tool_calls = AIMessage(
    content="",
    tool_calls=[
        {
            "name": "get_coolest_cities",
            "args": {},
            "id": "tool_call_id_1",
            "type": "tool_call",
        },
        {
            "name": "get_weather",
            "args": {"location": "sf"},
            "id": "tool_call_id_2",
            "type": "tool_call",
        },
    ]
)

# Process all tool calls in one invocation
output = tool_node.invoke({"messages": [message_with_multiple_tool_calls]})
print(output)
# Output:
# {'messages': [
#   ToolMessage(content='nyc, sf', name='get_coolest_cities', tool_call_id='tool_call_id_1'),
#   ToolMessage(content="It's 60 degrees and foggy.", name='get_weather', tool_call_id='tool_call_id_2')
# ]}
```

### Integrating with Chat Models
Combine ToolNode with a LangChain chat model to create a complete tool calling pipeline:

```python
from langchain_anthropic import ChatAnthropic

# Initialize the model with tools
model_with_tools = ChatAnthropic(
    model="claude-3-haiku-20240307", 
    temperature=0
).bind_tools(tools)

# Generate a response with tool calls
query = "What's the weather in sf?"
response = model_with_tools.invoke(query)
print(response.tool_calls)
# Output:
# [{'name': 'get_weather', 'args': {'location': 'San Francisco'}, 'id': 'tool_call_id', 'type': 'tool_call'}]

# Execute the tool calls using ToolNode
tool_output = tool_node.invoke({"messages": [response]})
print(tool_output)
# Output:
# {'messages': [ToolMessage(content="It's 60 degrees and foggy.", name='get_weather', tool_call_id='tool_call_id')]}
```

### Building a ReAct Agent with ToolNode
Create a complete ReAct agent that cycles between generating tool calls and executing them:

```python
from langgraph.graph import StateGraph, MessagesState, START, END

# Create a state graph
workflow = StateGraph(MessagesState)

# Define the model node function
def call_model(state: MessagesState):
    messages = state["messages"]
    response = model_with_tools.invoke(messages)
    return {"messages": [response]}

# Define the routing function
def should_continue(state: MessagesState):
    messages = state["messages"]
    last_message = messages[-1]
    if last_message.tool_calls:
        return "tools"  # More tool calls to process
    return END  # No more tool calls, we're done

# Add nodes and edges
workflow.add_node("agent", call_model)
workflow.add_node("tools", tool_node)
workflow.add_edge(START, "agent")
workflow.add_conditional_edges("agent", should_continue, ["tools", END])
workflow.add_edge("tools", "agent")

# Compile the graph
app = workflow.compile()
```

## Usage Example
Here's a complete example of running a ReAct agent with ToolNode:

```python
from langchain_core.messages import HumanMessage

# Create a query that requires multiple tool calls
query = "First tell me the coolest cities, then tell me the weather in each."
messages = [HumanMessage(content=query)]

print(f"User: {query}\n")

# Stream the responses to see the agent's reasoning
for chunk in app.stream(
    {"messages": messages}, 
    stream_mode="values"
):
    # Get the last message in the chunk
    last_message = chunk["messages"][-1]
    
    # Print the content based on message type
    if hasattr(last_message, "role") and last_message.role == "ai":
        print(f"AI: {last_message.content}")
        if hasattr(last_message, "tool_calls") and last_message.tool_calls:
            print(f"Tool Calls: {last_message.tool_calls}")
    
    elif hasattr(last_message, "name"):
        print(f"Tool ({last_message.name}): {last_message.content}")
    
    print("-" * 50)
```

### Advanced Configuration: Error Handling
Configure how ToolNode handles tool execution errors:

```python
# Create a ToolNode with custom error handling
error_handling_tool_node = ToolNode(
    tools=tools,
    handle_tool_errors=True,  # Default: True
    # You can also provide a custom error handler function
    error_handler=lambda tool_name, error, args: f"Error in {tool_name}: {str(error)}"
)

# Example of a tool that might raise an error
@tool
def divide(a: float, b: float) -> float:
    """Divide a by b."""
    return a / b

# Using the error handling tool node
error_prone_tools = [divide]
safe_tool_node = ToolNode(error_prone_tools)

# This will handle the division by zero error
message_with_error = AIMessage(
    content="",
    tool_calls=[{
        "name": "divide",
        "args": {"a": 1, "b": 0},
        "id": "error_call_id",
        "type": "tool_call",
    }]
)

result = safe_tool_node.invoke({"messages": [message_with_error]})
print(result)
```

## Benefits
- Simplifies the implementation of tool-using agents
- Enables parallel execution of multiple tool calls
- Integrates cleanly with LangGraph's state management
- Reduces boilerplate code for common tool calling patterns
- Provides built-in error handling for robust applications

## Considerations
- Tools must be correctly registered and their signatures properly defined
- Error handling should be configured based on your application's needs
- Plan for appropriate fallbacks when tools fail to execute
- Consider optimizing for latency when using multiple tools in sequence
- Test tool integrations thoroughly with various inputs
