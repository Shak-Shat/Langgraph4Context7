# Building Hierarchical Agent Teams

## Overview
This guide demonstrates how to create hierarchical agent teams in LangGraph, where a top-level supervisor delegates tasks to mid-level supervisors and specialized worker agents. This approach enables handling complex workflows by breaking them into manageable components and distributing responsibilities effectively.

## Key Concepts
- **Hierarchical Structure**: A multi-level organization of agents with supervisors and workers
- **Agent Specialization**: Assigning specific roles and tools to different agents based on their responsibilities
- **Subgraphs**: Encapsulating team functionality as reusable components
- **Dynamic Routing**: Supervisors that decide which agent should handle each task
- **State Management**: Maintaining consistent state across the agent hierarchy

## Prerequisites
- LangGraph and LangChain packages installed
- API keys for language models (like OpenAI) and search engines (like Tavily)
- Understanding of basic LangGraph concepts

```python
# Install required packages
pip install -U langgraph langchain_community langchain_anthropic langchain_experimental

# Set up environment variables for API access
import os
os.environ["OPENAI_API_KEY"] = "your-openai-api-key"
os.environ["TAVILY_API_KEY"] = "your-tavily-api-key"
```

## Implementation

### Setting Up the Work Environment
First, create a temporary directory for file operations:

```python
from pathlib import Path
from tempfile import TemporaryDirectory
from typing import List

_TEMP_DIRECTORY = TemporaryDirectory()
WORKING_DIRECTORY = Path(_TEMP_DIRECTORY.name)
```

### Defining Specialized Tools
Create tools for different agent types:

```python
from langchain_community.document_loaders import WebBaseLoader
from langchain_community.tools.tavily_search import TavilySearchResults
from langchain_core.tools import tool
from langchain_experimental.utilities import PythonREPL

# Research tools
tavily_tool = TavilySearchResults(max_results=5)

@tool
def scrape_webpages(urls: List[str]) -> str:
    """Scrape the provided web pages for detailed information."""
    loader = WebBaseLoader(urls)
    docs = loader.load()
    return "\n\n".join(
        [f'<Document name="{doc.metadata.get("title", "")}">\n{doc.page_content}\n</Document>'
         for doc in docs]
    )

# Document writing tools
@tool
def create_outline(points: List[str], file_name: str) -> str:
    """Create and save an outline."""
    with (WORKING_DIRECTORY / file_name).open("w") as file:
        for i, point in enumerate(points):
            file.write(f"{i + 1}. {point}\n")
    return f"Outline saved to {file_name}"

@tool
def write_document(content: str, file_name: str) -> str:
    """Save a text document."""
    with (WORKING_DIRECTORY / file_name).open("w") as file:
        file.write(content)
    return f"Document saved to {file_name}"

@tool
def read_document(file_name: str) -> str:
    """Read the content of a document."""
    try:
        with (WORKING_DIRECTORY / file_name).open("r") as file:
            return file.read()
    except FileNotFoundError:
        return f"File {file_name} not found."

@tool
def edit_document(content: str, file_name: str) -> str:
    """Edit an existing document."""
    try:
        with (WORKING_DIRECTORY / file_name).open("r") as file:
            existing_content = file.read()
        
        with (WORKING_DIRECTORY / file_name).open("w") as file:
            file.write(content)
        
        return f"Document {file_name} updated."
    except FileNotFoundError:
        return f"File {file_name} not found."
```

### Creating the State Class and Supervisor Node
Define the state structure and a function to create supervisor nodes:

```python
from langgraph.graph import StateGraph, MessagesState
from langgraph.types import Command
from langchain_core.language_models import BaseChatModel

class State(MessagesState):
    next: str

def make_supervisor_node(llm: BaseChatModel, members: list[str]):
    """Create a supervisor node with routing capabilities."""
    system_prompt = (
        f"You are a supervisor managing workers: {', '.join(members)}. "
        f"Your job is to decide which worker should handle the current task or if the task is complete.\n"
        f"Available options: {', '.join(members + ['FINISH'])}\n"
        f"Respond with the name of the next worker or FINISH if the task is complete."
    )
    
    def supervisor_node(state: State) -> Command:
        messages = state["messages"]
        response = llm.invoke([
            {"role": "system", "content": system_prompt},
            *[{"role": m[0], "content": m[1]} for m in messages]
        ])
        
        # Extract the decision from the response
        content = response.content
        for member in members + ["FINISH"]:
            if member.lower() in content.lower():
                goto = member
                break
        else:
            goto = "FINISH"
        
        # Return command with destination and state update
        return Command(
            goto=goto if goto != "FINISH" else "__end__", 
            update={"messages": [("assistant", response.content)], "next": goto}
        )
    
    return supervisor_node
```

### Building the Research Team
Create a research team that can search and scrape web content:

```python
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent

# Initialize language model
llm = ChatOpenAI(model="gpt-4o")

# Create individual agents with their specialized tools
search_agent = create_react_agent(llm, tools=[tavily_tool])
web_scraper_agent = create_react_agent(llm, tools=[scrape_webpages])

# Build the research team graph
research_builder = StateGraph(State)
research_builder.add_node("supervisor", make_supervisor_node(llm, ["search", "web_scraper"]))
research_builder.add_node("search", lambda state: search_agent.invoke(state))
research_builder.add_node("web_scraper", lambda state: web_scraper_agent.invoke(state))

# Define the workflow
research_builder.add_edge("START", "supervisor")
research_builder.add_edge("supervisor", "search")
research_builder.add_edge("supervisor", "web_scraper")
research_builder.add_edge("search", "supervisor")
research_builder.add_edge("web_scraper", "supervisor")
research_builder.add_edge("supervisor", "END")

# Compile the graph
research_graph = research_builder.compile()
```

### Building the Document Writing Team
Create a document writing team that can generate content and charts:

```python
# Create individual agents with their specialized tools
doc_writer_agent = create_react_agent(llm, tools=[write_document, edit_document, create_outline, read_document])
chart_generating_agent = create_react_agent(llm, tools=[PythonREPL()])

# Build the document writing team graph
paper_writing_builder = StateGraph(State)
paper_writing_builder.add_node("supervisor", make_supervisor_node(llm, ["doc_writer", "chart_generator"]))
paper_writing_builder.add_node("doc_writer", lambda state: doc_writer_agent.invoke(state))
paper_writing_builder.add_node("chart_generator", lambda state: chart_generating_agent.invoke(state))

# Define the workflow
paper_writing_builder.add_edge("START", "supervisor")
paper_writing_builder.add_edge("supervisor", "doc_writer")
paper_writing_builder.add_edge("supervisor", "chart_generator")
paper_writing_builder.add_edge("doc_writer", "supervisor")
paper_writing_builder.add_edge("chart_generator", "supervisor")
paper_writing_builder.add_edge("supervisor", "END")

# Compile the graph
paper_writing_graph = paper_writing_builder.compile()
```

### Creating the Top-Level Supervisor
Integrate the teams under a top-level supervisor:

```python
# Create the top-level supervisor node
teams_supervisor_node = make_supervisor_node(llm, ["research_team", "writing_team"])

# Build the hierarchical system
super_builder = StateGraph(State)
super_builder.add_node("supervisor", teams_supervisor_node)
super_builder.add_node("research_team", lambda state: research_graph.invoke(state))
super_builder.add_node("writing_team", lambda state: paper_writing_graph.invoke(state))

# Define the workflow
super_builder.add_edge("START", "supervisor")
super_builder.add_edge("supervisor", "research_team")
super_builder.add_edge("supervisor", "writing_team")
super_builder.add_edge("research_team", "supervisor")
super_builder.add_edge("writing_team", "supervisor")
super_builder.add_edge("supervisor", "END")

# Compile the graph
super_graph = super_builder.compile()
```

## Usage Example
Here's how to use the hierarchical agent team to perform a complex task:

```python
# Run the hierarchical agent system
result = super_graph.invoke(
    {
        "messages": [("user", "Research AI agents and write a brief report about them. Include a simple visualization of the agent types.")],
    },
    {"recursion_limit": 150},  # Set a high recursion limit for complex tasks
)

# To see step-by-step progress, use streaming
for chunk in super_graph.stream(
    {
        "messages": [("user", "Research AI agents and write a brief report about them. Include a simple visualization of the agent types.")],
    },
    {"recursion_limit": 150},
):
    # Process each chunk as needed
    if "messages" in chunk and chunk["messages"]:
        last_message = chunk["messages"][-1]
        if last_message[0] == "assistant":
            print(f"Update: {last_message[1][:100]}...")

# When execution is complete, read the resulting document
if result.get("next") == "FINISH":
    # Find the document name from the interaction
    document_name = None
    for message in result["messages"]:
        if "saved to" in message[1]:
            potential_doc = message[1].split("saved to ")[-1].strip()
            if (WORKING_DIRECTORY / potential_doc).exists():
                document_name = potential_doc
                break
    
    if document_name:
        with (WORKING_DIRECTORY / document_name).open("r") as file:
            print(f"Final document content:\n{file.read()}")
```

## Benefits
- **Complex Task Management**: Break complex workflows into modular, manageable components
- **Specialized Expertise**: Each agent can be optimized for specific subtasks
- **Flexible Delegation**: Supervisors dynamically route tasks to the appropriate specialists
- **Scalability**: Easily add new agent types or entire teams to the hierarchy
- **Reusability**: Team subgraphs can be reused across different applications

## Considerations
- **Cost Management**: Hierarchical systems may increase API usage and costs
- **Error Handling**: Implement robust error handling to prevent cascading failures
- **Security**: Be cautious with tools that execute code or modify the filesystem
- **Performance**: Too many levels of hierarchy may impact response times
- **State Consistency**: Ensure state is consistently passed between teams and agents