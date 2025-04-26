# Thread-level Memory for ReAct Agents

## Overview
This guide demonstrates how to add persistent memory to a ReAct agent in LangGraph, allowing the agent to maintain conversation context across multiple interactions.

## Key Concepts
- Thread-level memory persistence in LangGraph
- Stateful conversation management across interactions
- Checkpointer interface for memory storage

## Prerequisites
- LangGraph and LangChain OpenAI installed
- OpenAI API key configured
- Familiarity with LangGraph persistence concepts
- Understanding of agent architectures and chat models

## Implementation

### Setup Dependencies
```python
# Install required packages
# pip install -U langgraph langchain-openai

import os
from typing import Literal
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langgraph.checkpoint.memory import MemorySaver
from langgraph.prebuilt import create_react_agent
```

### Initialize LLM Model
```python
# Initialize the OpenAI model
model = ChatOpenAI(model="gpt-4o", temperature=0)
```

### Define Custom Tools
```python
@tool
def get_weather(city: Literal["nyc", "sf"]):
    """Use this to get weather information."""
    if city == "nyc":
        return "It might be cloudy in nyc"
    elif city == "sf":
        return "It's always sunny in sf"
    else:
        raise AssertionError("Unknown city")

tools = [get_weather]
```

### Set Up Memory Persistence
```python
# Create memory persistence using LangGraph's checkpointer
memory = MemorySaver()
```

### Create ReAct Agent with Memory
```python
# Define the graph with memory persistence
graph = create_react_agent(model, tools=tools, checkpointer=memory)
```

## Usage Example

### Helper Function for Output Display
```python
def print_stream(stream):
    for s in stream:
        message = s["messages"][-1]
        if isinstance(message, tuple):
            print(message)
        else:
            message.pretty_print()
```

### First Interaction with the Agent
```python
# Define a unique thread ID for conversation persistence
config = {"configurable": {"thread_id": "1"}}

# Initial user query
inputs = {"messages": [("user", "What's the weather in NYC?")]}

# Run the graph and stream the results
print_stream(graph.stream(inputs, config=config, stream_mode="values"))
```

### Follow-up Interaction with Memory
```python
# Follow-up question using the same thread ID
inputs = {"messages": [("user", "What's it known for?")]}

# The agent remembers the previous context about NYC
print_stream(graph.stream(inputs, config=config, stream_mode="values"))
```

## Benefits
- Maintains conversation context between user interactions
- Improves user experience with contextual responses
- Enables complex multi-turn conversations
- Allows for stateful agents without rebuilding context

## Considerations
- Memory persistence requires unique thread IDs to separate conversations
- Consider memory management for long-running or numerous conversations
- Memory checkpointing may have performance implications for large conversation histories
