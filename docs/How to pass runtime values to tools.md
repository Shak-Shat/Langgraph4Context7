# Passing Runtime Values to Tools

## Overview
This guide explains how to pass runtime values to tools in LangGraph applications. You'll learn how to use special annotations that allow certain tool parameters to be auto-populated from the graph state or application memory rather than being generated by the language model, enabling more secure and context-aware workflows.

## Key Concepts
- **Injected Parameters**: Tool parameters that are populated at runtime rather than by the language model
- **InjectedState**: Annotation to mark parameters that should receive the current graph state
- **InjectedStore**: Annotation to mark parameters that should receive access to shared memory
- **Tool Schema Modification**: How parameter annotations affect the schema seen by the language model

## Prerequisites
- LangGraph and LangChain packages installed
- Understanding of basic LangGraph concepts and tool usage
- An OpenAI API key (if using OpenAI models)

```python
# Install required packages
pip install -U langgraph langchain-openai

# Set up environment variables
import os
os.environ["OPENAI_API_KEY"] = "your-api-key"
```

## Implementation

### Creating Tools with Injected State
Define tools that can access the graph state but hide these parameters from the LLM:

```python
from typing import List
from langchain_core.tools import tool
from langgraph.prebuilt import InjectedState
from langgraph.prebuilt.chat_agent_executor import AgentState

# Define your state type
class State(AgentState):
    docs: List[str]

@tool
def get_context(question: str, state: InjectedState(dict)) -> str:
    """Retrieve relevant context based on the current query and graph state.
    
    Args:
        question: The user's question that needs context
        state: Will be automatically populated with the current graph state
        
    Returns:
        A string containing all relevant document content
    """
    # Access the 'docs' field from state
    return "\n\n".join(doc for doc in state["docs"])
```

### Examining the Tool Schema
Verify that injected parameters are hidden from the LLM:

```python
# Check the tool schema that will be presented to the LLM
schema = get_context.tool_call_schema.schema()
print(schema)
# Notice the 'state' parameter is not included in the schema
```

The output will show that the `state` parameter is not included in the schema, ensuring the LLM will only try to provide the `question` parameter.

### Building a Graph with Injected State Tools
Integrate the tools into a LangGraph workflow:

```python
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import ToolNode, create_react_agent
from langgraph.checkpoint.memory import MemorySaver

# Initialize language model
model = ChatOpenAI(model="gpt-4o", temperature=0)

# Define tools that will use injected state
tools = [get_context]

# Create a tool node to handle tool execution
tool_node = ToolNode(tools)

# Set up persistent storage
checkpointer = MemorySaver()

# Create the agent graph
graph = create_react_agent(model, tools, state_schema=State, checkpointer=checkpointer)
```

### Running the Graph with State Injection
Execute the graph with initial state that includes documents:

```python
# Set up initial state including document context
docs = [
    "FooBar company just raised 1 Billion dollars!",
    "FooBar company was founded in 2019",
]

# Initialize input state
inputs = {
    "messages": [{"role": "user", "content": "What's the latest news about FooBar?"}],
    "docs": docs,
}

# Configuration for graph execution
config = {"configurable": {"thread_id": "thread-123"}}

# Stream the execution results
for chunk in graph.stream(inputs, config, stream_mode="values"):
    if "messages" in chunk and chunk["messages"]:
        last_message = chunk["messages"][-1]
        if hasattr(last_message, "content"):
            print(f"Agent: {last_message.content}")
```

The agent will use the injected documents to answer the question, even though the documents weren't directly passed to the tool by the LLM.

### Using Shared Memory with InjectedStore
For more complex scenarios, you can use shared memory that persists between runs:

```python
from langgraph.store.memory import InMemoryStore
from langgraph.store.base import BaseStore
from langchain_core.runnables import RunnableConfig
from langgraph.prebuilt import InjectedStore

# Create a shared memory store
doc_store = InMemoryStore()

# Add user-specific documents to the store
doc_store.put(("documents", "user-1"), "doc_0", {"doc": "FooBar company raised 1 Billion dollars!"})
doc_store.put(("documents", "user-1"), "doc_1", {"doc": "FooBar company was founded in 2019"})
doc_store.put(("documents", "user-2"), "doc_0", {"doc": "WidgetCorp launched a new product line."})

@tool
def get_user_context(question: str, config: RunnableConfig, store: InjectedStore(BaseStore)) -> str:
    """Retrieve context based on the user's specific documents.
    
    Args:
        question: The user's question that needs context
        config: The runtime configuration (automatically injected)
        store: The shared memory store (automatically injected)
        
    Returns:
        A string containing all relevant document content for the user
    """
    # Extract user ID from config
    user_id = config.get("configurable", {}).get("user_id")
    
    # Retrieve documents for this specific user
    docs = [item.value["doc"] for item in store.search(("documents", user_id))]
    
    return "\n\n".join(doc for doc in docs)
```

### Building and Running a Graph with Shared Memory
Set up a graph that uses the shared memory store:

```python
# Create the agent graph with the store
tools = [get_user_context]
graph_with_store = create_react_agent(model, tools, checkpointer=checkpointer, store=doc_store)

# Run for user 1
for chunk in graph_with_store.stream(
    {"messages": [{"role": "user", "content": "What's the latest news about FooBar?"}]},
    {"configurable": {"thread_id": "thread-1", "user_id": "user-1"}},
    stream_mode="values",
):
    if "messages" in chunk and chunk["messages"]:
        last_message = chunk["messages"][-1]
        if hasattr(last_message, "content"):
            print(f"User 1 Response: {last_message.content}")

# Run for user 2
for chunk in graph_with_store.stream(
    {"messages": [{"role": "user", "content": "What's the latest news?"}]},
    {"configurable": {"thread_id": "thread-2", "user_id": "user-2"}},
    stream_mode="values",
):
    if "messages" in chunk and chunk["messages"]:
        last_message = chunk["messages"][-1]
        if hasattr(last_message, "content"):
            print(f"User 2 Response: {last_message.content}")
```

Each user will receive information only from their own documents, even though they're using the same tools and graph.

## Usage Example
Here's a complete example of a document retrieval agent that uses injected state:

```python
from typing import List, TypedDict
from langchain_core.tools import tool
from langgraph.prebuilt import InjectedState, create_react_agent
from langchain_openai import ChatOpenAI

# Define state schema
class DocumentState(TypedDict):
    messages: List
    documents: List[str]
    search_history: List[str]

# Define a tool with injected state
@tool
def search_documents(query: str, state: InjectedState(dict)) -> str:
    """Search the available documents for information about the query.
    
    Args:
        query: What to search for in the documents
        state: Will be automatically populated with graph state
        
    Returns:
        Relevant document content
    """
    documents = state.get("documents", [])
    
    # Simple keyword search (in practice, use a real search algorithm)
    results = []
    for doc in documents:
        if query.lower() in doc.lower():
            results.append(doc)
    
    # Update search history in state
    return "\n\n".join(results) if results else "No relevant documents found."

@tool
def get_search_history(state: InjectedState(dict)) -> str:
    """Get the history of recent searches.
    
    Args:
        state: Will be automatically populated with graph state
        
    Returns:
        List of recent search queries
    """
    history = state.get("search_history", [])
    return "Recent searches:\n" + "\n".join(history)

# Set up the model and tools
model = ChatOpenAI(model="gpt-3.5-turbo")
tools = [search_documents, get_search_history]

# Create the agent graph
graph = create_react_agent(model, tools)

# Run with initial state
documents = [
    "FooBar Inc. reported Q2 earnings of $2.5M, up 15% YoY.",
    "FooBar Inc. recently acquired WidgetCo for $50M.",
    "WidgetCo specializes in IoT devices for smart homes.",
    "Market analysts predict FooBar stock will rise 20% this year."
]

# Initialize with documents and empty search history
result = graph.invoke({
    "messages": [{"role": "user", "content": "What are FooBar's recent acquisitions?"}],
    "documents": documents,
    "search_history": []
})

print(result["messages"][-1].content)
```

## Benefits
- **Parameter Security**: Hide sensitive parameters from LLM access
- **Context Awareness**: Tools can access the current application state
- **User-Specific Data**: Easily implement personalized tool behavior
- **Reduced Prompt Engineering**: No need to pass state information through prompts
- **Memory Persistence**: Share information between different tools and graph runs

## Considerations
- **Schema Clarity**: Ensure documentation clearly indicates which parameters are injected
- **Typing**: Use proper type annotations for better IDE support and error checking
- **Memory Management**: Be mindful of memory usage when storing large datasets
- **User Isolation**: Implement proper checks when accessing user-specific data
- **Error Handling**: Add robust error handling for cases when expected state fields are missing
