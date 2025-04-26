# Conversation History Summarization in LangGraph

## Overview
This guide demonstrates how to implement a conversation history summarization mechanism in LangGraph to manage context window utilization while maintaining conversational continuity.

## Key Concepts
- Dynamic conversation history summarization
- Message pruning to control context length
- State management for conversation memory
- Conditional workflow based on message count

## Prerequisites
- LangGraph and LangChain packages installed
- Anthropic API key configured
- Understanding of message state management
- Familiarity with conditional graph workflows

## Implementation

### Setup Dependencies
```python
from typing import Literal
from langchain_anthropic import ChatAnthropic
from langchain_core.messages import SystemMessage, RemoveMessage, HumanMessage
from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import MessagesState, StateGraph, START, END
```

### Define State Structure
```python
# Create persistent memory storage
memory = MemorySaver()

# Add a 'summary' attribute to the MessagesState
class State(MessagesState):
    summary: str

# Initialize LLM for conversation and summarization
model = ChatAnthropic(model_name="claude-3-haiku-20240307")
```

### Implement Core Graph Nodes

#### Conversation Node
```python
def call_model(state: State):
    # Include summary in system message if available
    summary = state.get("summary", "")
    if summary:
        system_message = f"Summary of conversation earlier: {summary}"
        messages = [SystemMessage(content=system_message)] + state["messages"]
    else:
        messages = state["messages"]
    response = model.invoke(messages)
    return {"messages": [response]}
```

#### Decision Node
```python
def should_continue(state: State) -> Literal["summarize_conversation", END]:
    """Return the next node to execute."""
    messages = state["messages"]
    # Summarize if more than six messages
    if len(messages) > 6:
        return "summarize_conversation"
    # Otherwise just end
    return END
```

#### Summarization Node
```python
def summarize_conversation(state: State):
    # Create or extend the conversation summary
    summary = state.get("summary", "")
    if summary:
        # Use existing summary as context
        summary_message = (
            f"This is summary of the conversation to date: {summary}\n\n"
            "Extend the summary by taking into account the new messages above:"
        )
    else:
        summary_message = "Create a summary of the conversation above:"

    messages = state["messages"] + [HumanMessage(content=summary_message)]
    response = model.invoke(messages)
    
    # Delete all but the last two messages to reduce context
    delete_messages = [RemoveMessage(id=m.id) for m in state["messages"][:-2]]
    return {"summary": response.content, "messages": delete_messages}
```

### Graph Construction
```python
# Define a new graph
workflow = StateGraph(State)

# Define the conversation node and the summarize node
workflow.add_node("conversation", call_model)
workflow.add_node(summarize_conversation)

# Set the entrypoint as conversation
workflow.add_edge(START, "conversation")

# Add conditional edge based on message count
workflow.add_conditional_edges(
    "conversation",
    should_continue,
)

# Add edge from summarize_conversation to END
workflow.add_edge("summarize_conversation", END)

# Compile the graph with memory persistence
app = workflow.compile(checkpointer=memory)
```

## Usage Example

### Helper Function for Output Display
```python
def print_update(update):
    for k, v in update.items():
        for m in v["messages"]:
            m.pretty_print()
        if "summary" in v:
            print(v["summary"])
```

### Conversation Flow
```python
# Configure with a thread ID for persistence
config = {"configurable": {"thread_id": "4"}}

# First message
input_message = HumanMessage(content="hi! I'm bob")
for event in app.stream({"messages": [input_message]}, config, stream_mode="updates"):
    print_update(event)

# Second message
input_message = HumanMessage(content="what's my name?")
for event in app.stream({"messages": [input_message]}, config, stream_mode="updates"):
    print_update(event)

# Continue conversation until summarization threshold
# After 7 messages (including responses), the conversation will be summarized
# and only the last 2 messages will remain in the context window
```

## Benefits
- Reduces token usage and context window consumption
- Enables longer conversations without losing critical information
- Maintains conversational continuity through summarization
- Prevents context window overflow errors
- Improves model response time by reducing input size

## Considerations
- Summary quality depends on the LLM's summarization abilities
- Fine-tuning the message threshold based on application needs
- Balance needed between retention and summarization frequency
- Consider using different models for conversation vs. summarization if performance is a concern
