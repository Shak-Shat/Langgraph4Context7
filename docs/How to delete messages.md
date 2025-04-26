# Message Deletion in LangGraph

## Overview
This guide demonstrates how to delete messages from a message state in a LangGraph application, both manually and programmatically using the RemoveMessage modifier.

## Key Concepts
- Messages can be deleted from state using RemoveMessage
- Each state key has a reducer that specifies how to combine updates
- MessagesState includes a reducer for messages that handles RemoveMessage
- Message deletion can be implemented within graph nodes
- Message ordering rules need to be preserved when deleting messages

## Prerequisites
```python
# Install required packages
%pip install -U langgraph langchain_anthropic

# Set API keys
import os
import getpass

def _set_env(var: str):
    if not os.environ.get(var):
        os.environ[var] = getpass.getpass(f"{var}: ")

_set_env("ANTHROPIC_API_KEY")
```

## Implementation

### Build a Simple Agent
```python
from typing import Literal

from langchain_anthropic import ChatAnthropic
from langchain_core.tools import tool

from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import MessagesState, StateGraph, START, END
from langgraph.prebuilt import ToolNode

memory = MemorySaver()

@tool
def search(query: str):
    """Call to surf the web."""
    # This is a placeholder for the actual implementation
    return "It's sunny in San Francisco, but you better look out if you're a Gemini 😈."

tools = [search]
tool_node = ToolNode(tools)
model = ChatAnthropic(model_name="claude-3-haiku-20240307")
bound_model = model.bind_tools(tools)

def should_continue(state: MessagesState):
    """Return the next node to execute."""
    last_message = state["messages"][-1]
    # If there is no function call, then we finish
    if not last_message.tool_calls:
        return END
    # Otherwise if there is, we continue
    return "action"

# Define the function that calls the model
def call_model(state: MessagesState):
    response = model.invoke(state["messages"])
    # We return a list, because this will get added to the existing list
    return {"messages": response}

# Define a new graph
workflow = StateGraph(MessagesState)

# Define the two nodes we will cycle between
workflow.add_node("agent", call_model)
workflow.add_node("action", tool_node)

# Set the entrypoint as `agent`
workflow.add_edge(START, "agent")

# Add a conditional edge
workflow.add_conditional_edges(
    "agent",
    should_continue,
    ["action", END],
)

# Add a normal edge from `tools` to `agent`
workflow.add_edge("action", "agent")

# Compile the graph
app = workflow.compile(checkpointer=memory)
```

### Using the Agent
```python
from langchain_core.messages import HumanMessage

config = {"configurable": {"thread_id": "2"}}
input_message = HumanMessage(content="hi! I'm bob")
for event in app.stream({"messages": [input_message]}, config, stream_mode="values"):
    event["messages"][-1].pretty_print()

input_message = HumanMessage(content="what's my name?")
for event in app.stream({"messages": [input_message]}, config, stream_mode="values"):
    event["messages"][-1].pretty_print()
```

### Manual Message Deletion
```python
# Get current messages
messages = app.get_state(config).values["messages"]

# Import RemoveMessage
from langchain_core.messages import RemoveMessage

# Delete a message by ID
app.update_state(config, {"messages": RemoveMessage(id=messages[0].id)})

# Verify deletion
messages = app.get_state(config).values["messages"]
```

### Programmatic Message Deletion
```python
from langchain_core.messages import RemoveMessage
from langgraph.graph import END

def delete_messages(state):
    messages = state["messages"]
    if len(messages) > 3:
        return {"messages": [RemoveMessage(id=m.id) for m in messages[:-3]]}

# Modified decision function to call delete_messages
def should_continue(state: MessagesState) -> Literal["action", "delete_messages"]:
    """Return the next node to execute."""
    last_message = state["messages"][-1]
    # If there is no function call, then we call our delete_messages function
    if not last_message.tool_calls:
        return "delete_messages"
    # Otherwise if there is, we continue
    return "action"

# Define a new graph with message deletion
workflow = StateGraph(MessagesState)
workflow.add_node("agent", call_model)
workflow.add_node("action", tool_node)
workflow.add_node(delete_messages)

workflow.add_edge(START, "agent")
workflow.add_conditional_edges(
    "agent",
    should_continue,
)
workflow.add_edge("action", "agent")
workflow.add_edge("delete_messages", END)
app = workflow.compile(checkpointer=memory)
```

### Testing the Auto-Deletion
```python
from langchain_core.messages import HumanMessage

config = {"configurable": {"thread_id": "3"}}
input_message = HumanMessage(content="hi! I'm bob")
for event in app.stream({"messages": [input_message]}, config, stream_mode="values"):
    print([(message.type, message.content) for message in event["messages"]])

input_message = HumanMessage(content="what's my name?")
for event in app.stream({"messages": [input_message]}, config, stream_mode="values"):
    print([(message.type, message.content) for message in event["messages"]])

# Check the message state
messages = app.get_state(config).values["messages"]
```

## Benefits
- Manage conversation memory efficiently by removing outdated messages
- Control token usage by keeping message history within manageable limits
- Implement strategies like sliding window of context for long-running conversations
- Selectively remove specific messages based on content or purpose

## Considerations
- Many models expect specific message ordering (e.g., starting with a user message)
- Ensure message deletion logic preserves required message flow patterns
- Tool call messages typically need to be followed by their respective tool responses
- Deleting messages can affect context continuity and model understanding
- Careful testing needed to ensure message deletion doesn't break model interactions
