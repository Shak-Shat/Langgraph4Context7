# Adding Cross-Thread Persistence in LangGraph

## Overview
This guide demonstrates how to implement persistence across multiple threads in LangGraph applications, allowing you to store and retrieve data that persists beyond a single conversation thread.

## Key Concepts
- **Cross-Thread Persistence**: Store and retrieve data across multiple threads or conversations
- **Store Interface**: Define namespaces and keys for organizing persistent data
- **Memory Integration**: Retain user-specific information between different sessions
- **Namespaces**: Logical groupings for stored data, typically by user ID

## Prerequisites
- LangGraph and LangChain packages installed
- OpenAI API key configured in environment variables
- Basic understanding of LangGraph state management

```python
# Install required packages
pip install langgraph langchain_openai

# Set OpenAI API key
import os
os.environ["OPENAI_API_KEY"] = "your-openai-api-key"
```

## Implementation

### Creating a Persistent Store
First, define an in-memory store with embedding capabilities for vector search:

```python
from langgraph.store.memory import InMemoryStore
from langchain_openai import OpenAIEmbeddings

# Initialize an in-memory store with embedding capabilities
in_memory_store = InMemoryStore(
    index={
        "embed": OpenAIEmbeddings(model="text-embedding-3-small"),
        "dims": 1536,
    }
)
```

### Defining a Graph with Persistence
Create a graph that uses the store to save and retrieve data across different threads:

```python
import uuid
from langgraph.graph import StateGraph, MessagesState, START
from typing import TypedDict, List, Dict, Any
from langchain_core.messages import BaseMessage
from langchain_openai import ChatOpenAI
from langgraph.graph.graph import RunnableConfig
from langgraph.store import BaseStore

# Initialize the model
model = ChatOpenAI()

# Define a node function that uses the store
def call_model(state: MessagesState, config: RunnableConfig, *, store: BaseStore):
    # Extract user ID from config
    user_id = config["configurable"]["user_id"]
    
    # Define a namespace based on user ID
    namespace = ("memories", user_id)
    
    # Search for relevant memories
    memories = store.search(namespace, query=str(state["messages"][-1].content))
    info = "\n".join([d.value["data"] for d in memories])
    
    # Use retrieved memories in the system message
    system_msg = f"You are a helpful assistant. User info: {info}"
    
    # Store new memory if the user asks to remember something
    if "remember" in state["messages"][-1].content.lower():
        store.put(namespace, str(uuid.uuid4()), {"data": "User name is Bob"})
    
    # Generate and return response
    response = model.invoke([{"role": "system", "content": system_msg}] + state["messages"])
    return {"messages": response}

# Create the graph
builder = StateGraph(MessagesState)
builder.add_node("call_model", call_model)
builder.add_edge(START, "call_model")
builder.add_edge("call_model", END)

# Compile the graph with the store
graph = builder.compile(store=in_memory_store)
```

### Using the Graph with Different Users
Run the graph with different user IDs to demonstrate isolated persistence:

```python
# First user conversation
config_user1 = {"configurable": {"thread_id": "1", "user_id": "1"}}
input_message1 = {"role": "user", "content": "Hi! Remember: my name is Bob"}

print("User 1 first message:")
for chunk in graph.stream({"messages": [input_message1]}, config_user1, stream_mode="values"):
    print(chunk["messages"][-1].content)

# Second user conversation
config_user2 = {"configurable": {"thread_id": "2", "user_id": "2"}}
input_message2 = {"role": "user", "content": "What's my name?"}

print("\nUser 2 first message:")
for chunk in graph.stream({"messages": [input_message2]}, config_user2, stream_mode="values"):
    print(chunk["messages"][-1].content)

# First user returns
input_message3 = {"role": "user", "content": "What's my name?"}

print("\nUser 1 returns:")
for chunk in graph.stream({"messages": [input_message3]}, config_user1, stream_mode="values"):
    print(chunk["messages"][-1].content)
```

## Usage Example
Here's a complete example showing how to use cross-thread persistence for a customer support bot that remembers customer details:

```python
# Initialize store and model
from langgraph.store.memory import InMemoryStore
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langgraph.graph import StateGraph, MessagesState, START, END
import uuid

# Setup store
store = InMemoryStore(
    index={
        "embed": OpenAIEmbeddings(model="text-embedding-3-small"),
        "dims": 1536,
    }
)

# Create customer support agent
def support_agent(state, config, *, store):
    customer_id = config["configurable"]["user_id"]
    namespace = ("customer_data", customer_id)
    
    # Extract order ID if mentioned
    message = state["messages"][-1].content.lower()
    if "order" in message and "#" in message:
        import re
        order_ids = re.findall(r"order #(\d+)", message)
        if order_ids:
            # Save order ID to customer data
            store.put(namespace, "order_info", {"order_id": order_ids[0]})
    
    # Retrieve customer data
    customer_data = {}
    try:
        for item in store.list(namespace):
            customer_data.update(item.value)
    except:
        pass
        
    # Format customer data for the model
    customer_info = ", ".join([f"{k}: {v}" for k, v in customer_data.items()])
    system_prompt = f"You are a customer support agent. Customer info: {customer_info}"
    
    # Generate response
    model = ChatOpenAI(temperature=0)
    response = model.invoke([{"role": "system", "content": system_prompt}] + state["messages"])
    return {"messages": response}

# Build and compile graph
builder = StateGraph(MessagesState)
builder.add_node("support_agent", support_agent)
builder.add_edge(START, "support_agent")
builder.add_edge("support_agent", END)
support_bot = builder.compile(store=store)

# Run example
config = {"configurable": {"user_id": "customer_123"}}
messages = [{"role": "user", "content": "I have a problem with my order #12345"}]
result = support_bot.invoke({"messages": messages}, config)
print(result["messages"][-1].content)

# Later conversation will remember the order number
later_message = [{"role": "user", "content": "Any updates on my order?"}]
result = support_bot.invoke({"messages": later_message}, config)
print(result["messages"][-1].content)
```

## Benefits
- Maintains user context across multiple conversation threads
- Creates isolated memory spaces for different users
- Enables personalized responses based on user history
- Allows for flexible data storage using namespaces and keys

## Considerations
- Use appropriate store implementation for production (e.g., Redis, MongoDB)
- Implement proper data retention policies and cleanup
- Consider privacy implications when storing user data
- For high-volume applications, use more scalable storage backends

## References
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/how-tos/cross-thread-persistence/)
