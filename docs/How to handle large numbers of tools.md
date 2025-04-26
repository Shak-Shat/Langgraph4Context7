# Handling Large Numbers of Tools

## Overview
This guide demonstrates how to effectively manage a large number of tools in LangGraph applications. You'll learn techniques for creating tool registries, using embeddings for semantic tool selection, and integrating dynamic tool retrieval into your agent workflows to improve performance and reduce token consumption.

## Key Concepts
- **Tool Registry**: A structured collection of tools with metadata for organization and retrieval
- **Semantic Tool Selection**: Using embeddings to find the most relevant tools for a given query
- **Vector Stores**: Database systems optimized for similarity search of embedded representations
- **Dynamic Tool Binding**: Attaching only relevant tools to an agent at runtime
- **Tool Filtering**: Limiting the number of tools provided to an agent based on contextual relevance

## Prerequisites
- LangGraph and LangChain packages installed
- OpenAI API key for embeddings and chat model
- Basic understanding of vector databases and embeddings

```python
# Install required packages
pip install langgraph langchain_openai numpy

# Set up environment variables
import os
from getpass import getpass

os.environ["OPENAI_API_KEY"] = getpass("Enter your OpenAI API key: ")
```

## Implementation

### Creating a Tool Registry
First, let's build a registry of tools with structured descriptions:

```python
import re
import uuid
from langchain_core.tools import StructuredTool

def create_tool(company: str) -> StructuredTool:
    """Create a structured tool for a specific company.
    
    Args:
        company: Name of the company
        
    Returns:
        A StructuredTool instance for querying company info
    """
    def company_tool(year: int) -> str:
        """Query financial data for a specific company year.
        
        Args:
            year: The fiscal year to query
            
        Returns:
            A string with financial information
        """
        return f"{company} had revenues of $100M in {year}."
    
    # Create safe function name from company name
    formatted_company = re.sub(r"[^\w\s]", "", company).replace(" ", "_").lower()
    
    return StructuredTool.from_function(
        func=company_tool,
        name=formatted_company,
        description=f"Get financial information about {company} for a specific year",
    )

# Example list of companies (this could be hundreds or thousands)
companies = [
    "Apple", "Microsoft", "Google", "Amazon", 
    "Meta", "Tesla", "NVIDIA", "AMD",
    "Intel", "Salesforce", "Adobe", "Oracle"
]

# Build tool registry with unique IDs
tool_registry = {
    str(uuid.uuid4()): create_tool(company) for company in companies
}

print(f"Created registry with {len(tool_registry)} tools")
```

### Building a Semantic Search System for Tools
Next, create a vector store to enable semantic search for relevant tools:

```python
from langchain_core.documents import Document
from langchain_core.vectorstores import InMemoryVectorStore
from langchain_openai import OpenAIEmbeddings

# Create document objects from tool descriptions
tool_documents = [
    Document(
        page_content=tool.description,
        metadata={"tool_name": tool.name},
        id=id,  # Store the UUID as document ID for retrieval
    )
    for id, tool in tool_registry.items()
]

# Initialize vector store with embeddings
embedding_model = OpenAIEmbeddings()
vector_store = InMemoryVectorStore(embedding=embedding_model)

# Add tools to vector store
document_ids = vector_store.add_documents(tool_documents)

print(f"Added {len(document_ids)} tool descriptions to vector store")
```

### Creating a Dynamic Tool Selection Workflow
Now, let's build a LangGraph workflow that selects relevant tools for each query:

```python
from langchain_openai import ChatOpenAI
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langchain_core.messages import HumanMessage, AIMessage

# Define state structure
class State(TypedDict):
    messages: list  # Conversation history
    selected_tools: list[str]  # IDs of selected tools

# Create selection function
def select_tools(state: State):
    """Select relevant tools based on the latest user message."""
    # Get the last user message
    for message in reversed(state["messages"]):
        if isinstance(message, HumanMessage):
            query = message.content
            break
    else:
        return {"selected_tools": []}  # No user message found
    
    # Perform semantic search to find relevant tools
    tool_documents = vector_store.similarity_search(
        query=query,
        k=3  # Retrieve top 3 most relevant tools
    )
    
    # Extract tool IDs from search results
    selected_tool_ids = [document.id for document in tool_documents]
    
    print(f"Selected {len(selected_tool_ids)} tools for query: {query}")
    return {"selected_tools": selected_tool_ids}

# Create agent function
def agent(state: State):
    """Run the agent with only the selected tools."""
    # Initialize language model
    llm = ChatOpenAI(temperature=0)
    
    # Get selected tools from registry
    if not state["selected_tools"]:
        # If no tools selected, use the model without tools
        response = llm.invoke(state["messages"])
    else:
        # Bind only the selected tools to the model
        selected_tools = [tool_registry[id] for id in state["selected_tools"]]
        tool_names = [tool.name for tool in selected_tools]
        print(f"Using tools: {', '.join(tool_names)}")
        
        # Bind selected tools to the model
        llm_with_tools = llm.bind_tools(selected_tools)
        response = llm_with_tools.invoke(state["messages"])
    
    # Return the AI response
    return {"messages": state["messages"] + [response]}

# Build the graph
builder = StateGraph(State)
builder.add_node("select_tools", select_tools)
builder.add_node("agent", agent)

# Configure workflow
builder.add_edge(START, "select_tools")
builder.add_edge("select_tools", "agent")
builder.add_edge("agent", END)

# Compile the graph
graph = builder.compile()
```

### Adding Tool Execution Capabilities
Let's enhance our workflow to handle tool execution:

```python
from langgraph.prebuilt import ToolNode
from langchain_core.messages import ToolMessage
from langgraph.graph import END

# Create a tool execution node
tool_node = ToolNode(tools=list(tool_registry.values()))

# Update state definition to include tool calls
class EnhancedState(TypedDict):
    messages: list
    selected_tools: list[str]

# Function to check if we need to execute tools
def should_use_tool(state: EnhancedState):
    """Determine if the AI's last message contains tool calls."""
    messages = state["messages"]
    last_message = messages[-1]
    
    # Check if the last message has tool calls
    if (isinstance(last_message, AIMessage) and 
        hasattr(last_message, "tool_calls") and 
        last_message.tool_calls):
        return "tool_node"
    return "END"

# Build enhanced graph
enhanced_builder = StateGraph(EnhancedState)
enhanced_builder.add_node("select_tools", select_tools)
enhanced_builder.add_node("agent", agent)
enhanced_builder.add_node("tool_node", tool_node)

# Configure workflow with tool execution
enhanced_builder.add_edge(START, "select_tools")
enhanced_builder.add_edge("select_tools", "agent")
enhanced_builder.add_conditional_edges("agent", should_use_tool, {
    "tool_node": "tool_node",
    "END": END
})
enhanced_builder.add_edge("tool_node", "agent")

# Compile the enhanced graph
enhanced_graph = enhanced_builder.compile()
```

## Usage Example
Let's see our dynamic tool selection in action:

```python
from langchain_core.messages import HumanMessage

# Test queries
queries = [
    "What was Apple's revenue in 2022?",
    "Can you tell me about NVIDIA's performance last year?",
    "How did the semiconductor industry do in 2021?"
]

# Run the graph for each query
for query in queries:
    print(f"\n--- Processing query: {query} ---")
    
    # Initialize state with user message
    initial_state = {
        "messages": [HumanMessage(content=query)],
        "selected_tools": []
    }
    
    # Run the graph
    result = enhanced_graph.invoke(initial_state)
    
    # Display results
    print("\nFinal conversation:")
    for message in result["messages"]:
        if hasattr(message, "tool_calls") and message.tool_calls:
            print(f"AI (with tool calls): {message.content}")
            for tool_call in message.tool_calls:
                print(f"  Tool call: {tool_call['name']}({tool_call['args']})")
        else:
            print(f"{message.type.upper()}: {message.content}")
```

## Benefits
- **Reduced Token Consumption**: Only relevant tool descriptions are sent to the model
- **Improved Response Time**: Processing fewer tools leads to faster completions
- **Enhanced Reasoning**: Models perform better when presented with fewer, more relevant tools
- **Scalability**: Enables working with hundreds or thousands of tools
- **Dynamic Adaptation**: Tool selection changes based on conversation context

## Considerations
- **Embedding Quality**: The effectiveness depends on the quality of tool descriptions and embeddings
- **Tool Description Clarity**: Write clear, descriptive tool documentation for better matching
- **Retrieval Parameters**: Adjust the number of tools retrieved based on your use case
- **Fallback Mechanisms**: Implement retry logic if initial tool selection is insufficient
- **Caching**: Consider caching embeddings for better performance with large tool sets