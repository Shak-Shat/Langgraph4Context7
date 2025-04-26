# How to Manage Conversation History in LangGraph

## Overview
This guide explains how to effectively manage conversation history in LangGraph applications. As conversations grow longer, the history can consume increasing amounts of the context window, leading to more expensive LLM calls and potential errors. This document provides strategies for managing conversation history while maintaining the necessary context.

## Key Concepts
- Conversation history persistence across interactions
- Message filtering techniques to prevent context window overflow
- Strategic message retention for maintaining relevant context
- State management in LangGraph workflows

## Prerequisites
```python
# Install required packages
%pip install -U langgraph langchain_anthropic

# Set up API keys
import os
import getpass

def _set_env(var: str):
    if not os.environ.get(var):
        os.environ[var] = getpass.getpass(f"{var}: ")

_set_env("ANTHROPIC_API_KEY")

# Import necessary modules
from typing import Literal
from langchain_anthropic import ChatAnthropic
from langchain_core.tools import tool
from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import MessagesState, StateGraph, START, END
from langgraph.prebuilt import ToolNode
from langchain_core.messages import HumanMessage
```

## Implementation

### Setting Up a Basic Agent
First, let's build a simple ReAct-style agent with memory:

```python
# Create a memory saver for persistence
memory = MemorySaver()

# Define a search tool
@tool
def search(query: str):
    """Call to surf the web."""
    return "It's sunny in San Francisco, but you better look out if you're a Gemini 😈."

# Set up tools and model
tools = [search]
tool_node = ToolNode(tools)
model = ChatAnthropic(model_name="claude-3-haiku-20240307")
bound_model = model.bind_tools(tools)

# Define the routing logic
def should_continue(state: MessagesState):
    """Return the next node to execute."""
    last_message = state["messages"][-1]
    # If there is no function call, then we finish
    if not last_message.tool_calls:
        return END
    # Otherwise if there is, we continue
    return "action"

# Define the model calling function
def call_model(state: MessagesState):
    response = bound_model.invoke(state["messages"])
    return {"messages": response}

# Build the graph
workflow = StateGraph(MessagesState)
workflow.add_node("agent", call_model)
workflow.add_node("action", tool_node)
workflow.add_edge(START, "agent")
workflow.add_conditional_edges("agent", should_continue, ["action", END])
workflow.add_edge("action", "agent")

# Compile the graph with memory
app = workflow.compile(checkpointer=memory)
```

### Testing the Basic Agent
Let's see how the agent manages conversation history by default:

```python
# Set a thread ID for persistence
config = {"configurable": {"thread_id": "2"}}

# First interaction
input_message = HumanMessage(content="hi! I'm bob")
for event in app.stream({"messages": [input_message]}, config, stream_mode="values"):
    event["messages"][-1].pretty_print()

# Second interaction - the agent remembers the user's name
input_message = HumanMessage(content="what's my name?")
for event in app.stream({"messages": [input_message]}, config, stream_mode="values"):
    event["messages"][-1].pretty_print()
```

### Implementing Message Filtering
To prevent context window overflow, we can filter messages before passing them to the LLM:

```python
# Define a filtering function
def filter_messages(messages: list):
    # Simple example: only use the last message
    return messages[-1:]

# Modified model calling function with filtering
def call_model_with_filtering(state: MessagesState):
    messages = filter_messages(state["messages"])
    response = bound_model.invoke(messages)
    return {"messages": response}

# Update the graph with the new model function
workflow = StateGraph(MessagesState)
workflow.add_node("agent", call_model_with_filtering)  # Using the filtered version
workflow.add_node("action", tool_node)
workflow.add_edge(START, "agent")
workflow.add_conditional_edges("agent", should_continue, ["action", END])
workflow.add_edge("action", "agent")

# Compile the graph
app_with_filtering = workflow.compile(checkpointer=memory)
```

### Testing the Filtered Agent
Now let's see how the filtered agent behaves:

```python
# First interaction
input_message = HumanMessage(content="hi! I'm bob")
for event in app_with_filtering.stream({"messages": [input_message]}, config, stream_mode="values"):
    event["messages"][-1].pretty_print()

# Second interaction - the agent no longer remembers the name
input_message = HumanMessage(content="what's my name?")
for event in app_with_filtering.stream({"messages": [input_message]}, config, stream_mode="values"):
    event["messages"][-1].pretty_print()
```

## Benefits
- Prevents context window overflow in long conversations
- Reduces token usage and associated costs
- Maintains essential conversation context
- Improves response times by limiting context size
- Enables more efficient use of LLM capabilities

## Considerations
- Simple filtering (keeping only the last message) eliminates all conversation history
- More sophisticated filtering strategies can retain important context while limiting size
- Consider using LangChain's built-in message filtering and trimming utilities
- The filtering strategy should balance context retention with token efficiency
- Thread IDs can be used to maintain persistence across separate interactions
