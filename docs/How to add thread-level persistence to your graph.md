# Thread-Level Persistence in LangGraph

## Overview
This guide demonstrates how to implement thread-level persistence in LangGraph applications, allowing AI systems to maintain conversation context across multiple interactions within the same thread.

## Key Concepts
- Thread-level persistence maintains state between interactions
- Uses checkpointers to save and retrieve graph state
- Different thread IDs create separate conversation contexts
- Simple implementation with MemorySaver

## Prerequisites
```python
%pip install --quiet -U langgraph langchain_anthropic
import os
```

## Implementation

### Setting Up the Model
```python
from langchain_anthropic import ChatAnthropic

model = ChatAnthropic(model="claude-3-5-sonnet-20240620")
```

### Creating a Basic Graph
```python
from typing import Annotated
from typing_extensions import TypedDict

from langgraph.graph import StateGraph, MessagesState, START


def call_model(state: MessagesState):
    response = model.invoke(state["messages"])
    return {"messages": response}


builder = StateGraph(MessagesState)
builder.add_node("call_model", call_model)
builder.add_edge(START, "call_model")
graph = builder.compile()
```

### Graph Without Persistence
```python
input_message = {"role": "user", "content": "hi! I'm bob"}
for chunk in graph.stream({"messages": [input_message]}, stream_mode="values"):
    chunk["messages"][-1].pretty_print()

input_message = {"role": "user", "content": "what's my name?"}
for chunk in graph.stream({"messages": [input_message]}, stream_mode="values"):
    chunk["messages"][-1].pretty_print()
```

Example output (without persistence):
```
================================[1m Human Message [0m=================================

hi! I'm bob

==================================[1m Ai Message [0m==================================

Hello Bob! It's nice to meet you. How are you doing today? Is there anything I can help you with or would you like to chat about something in particular?

================================[1m Human Message [0m=================================

what's my name?

==================================[1m Ai Message [0m==================================

I apologize, but I don't have access to your personal information, including your name. I'm an AI language model designed to provide general information and answer questions to the best of my ability based on my training data. I don't have any information about individual users or their personal details. If you'd like to share your name, you're welcome to do so, but I won't be able to recall it in future conversations.
```

### Adding Persistence
```python
from langgraph.checkpoint.memory import MemorySaver

memory = MemorySaver()
graph = builder.compile(checkpointer=memory)
# If you're using LangGraph Cloud or LangGraph Studio, you don't need to pass the checkpointer when compiling the graph, since it's done automatically.
```

### Using Persistence with Thread IDs
```python
config = {"configurable": {"thread_id": "1"}}
input_message = {"role": "user", "content": "hi! I'm bob"}
for chunk in graph.stream({"messages": [input_message]}, config, stream_mode="values"):
    chunk["messages"][-1].pretty_print()
```

Example output (first message with persistence):
```
================================[1m Human Message [0m=================================

hi! I'm bob

==================================[1m Ai Message [0m==================================

Hello Bob! It's nice to meet you. How are you doing today? Is there anything in particular you'd like to chat about or any questions you have that I can help you with?
```

### Resuming Previous Threads
```python
input_message = {"role": "user", "content": "what's my name?"}
for chunk in graph.stream({"messages": [input_message]}, config, stream_mode="values"):
    chunk["messages"][-1].pretty_print()
```

Example output (with memory):
```
================================[1m Human Message [0m=================================

what's my name?

==================================[1m Ai Message [0m==================================

Your name is Bob, as you introduced yourself at the beginning of our conversation.
```

### Starting New Conversations
```python
input_message = {"role": "user", "content": "what's my name?"}
for chunk in graph.stream(
    {"messages": [input_message]},
    {"configurable": {"thread_id": "2"}},
    stream_mode="values",
):
    chunk["messages"][-1].pretty_print()
```

Example output (new thread):
```
================================[1m Human Message [0m=================================

what's is my name?

==================================[1m Ai Message [0m==================================

I apologize, but I don't have access to your personal information, including your name. As an AI language model, I don't have any information about individual users unless it's provided within the conversation. If you'd like to share your name, you're welcome to do so, but otherwise, I won't be able to know or guess it.
```

## Benefits
- Maintains conversation context across multiple interactions
- Allows for personalized responses based on user history
- Simplifies implementation of stateful conversational agents
- Enables users to resume conversations where they left off

## Considerations
- Thread IDs must be properly managed to maintain distinct conversations
- Memory usage increases with conversation length
- For cross-thread persistence, additional solutions are required
- Consider security and privacy implications of stored conversation history
