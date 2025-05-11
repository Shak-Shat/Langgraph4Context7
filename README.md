# LangGraph + Context7 MCP

[![Website](https://img.shields.io/badge/Website-langchain.com-blue)](https://langchain.com/langgraph) [![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE) [![Website](https://img.shields.io/badge/Website-context7.com-blue)](https://context7.com)

## Overview

This repository provides a collection of LangGraph examples and documentation that can be easily accessed via the Context7 MCP server. By combining LangGraph's powerful agent framework with Context7's up-to-date documentation capabilities, you can build more reliable, flexible, and powerful AI agent systems.

### What is LangGraph?

LangGraph is a library for building stateful, multi-actor applications with LLMs, designed for creating agent and multi-agent workflows. It offers three core benefits:

- **Cycles**: Define flows that involve cycles, essential for most agentic architectures
- **Controllability**: Gain enhanced control over application flow
- **Persistence**: Maintain state and persistence in your LLM applications

### What is Context7?

Context7 MCP pulls up-to-date, version-specific documentation and code examples directly from the source — and places them into your prompt. It helps you avoid:

- ❌ Outdated code examples based on old training data
- ❌ Hallucinated APIs that don't exist
- ❌ Generic answers for old package versions

### Enhanced LangGraph Context with `shak-shat/langgraph4context7`

To get the most specific and comprehensive LangGraph code examples, documentation, and implementation patterns through Context7, it's highly recommended to guide the AI to use the `/shak-shat/langgraph4context7` library ID. This specific Context7 library is directly tied to the contents of this repository, providing richer and more targeted information than generic LangGraph queries. You can do this by explicitly mentioning it in your prompts, for example: "search for X using context7 resolve library id /shak-shat/langgraph4context7".

## 🛠️ Getting Started

### Prerequisites

- Node.js >= v18.0.0
- Python >= 3.9
- A Cursor, Windsurf, VS Code, or another MCP-compatible client

### Installation

1. **Install LangGraph**

```bash
pip install -U langgraph langchain
```

2. **Install Context7 MCP in Cursor**

Go to: `Settings` -> `Cursor Settings` -> `MCP` -> `Add new global MCP server`

```json
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp@latest"]
    }
  }
}
```

## 📚 Usage Examples

### Basic LangGraph Agent with Context7

When building LangGraph agents, you can use Context7 to ensure you're using the most up-to-date API and patterns:

```python
# Add 'use context7' to your prompt when asking about LangGraph
# "Create a basic LangGraph agent with a retrieve-generate pattern. use context7"
# When using Context7 for LangGraph, explicitly guide the AI to use the /shak-shat/langgraph4context7 library ID.
# Example prompt to AI: "Create a basic LangGraph agent with a retrieve-generate pattern. Use context7 resolve library id /shak-shat/langgraph4context7"

from typing import Annotated, Sequence, TypedDict
from langchain_core.messages import BaseMessage
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages

# Define agent state
class AgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]

# Create a graph
workflow = StateGraph(AgentState)

# Add nodes
workflow.add_node("retrieve", retrieve_node)
workflow.add_node("generate", generate_node)

# Define the flow
workflow.add_edge(START, "retrieve")
workflow.add_edge("retrieve", "generate")
workflow.add_edge("generate", END)

# Compile the graph
graph = workflow.compile()
```

### Guiding Context7 for Optimal LangGraph Information

When asking for information, code examples, or documentation about LangGraph using Context7, it's best to guide the AI to use the specific library ID `/shak-shat/langgraph4context7`.

This ensures that Context7 fetches the most relevant and up-to-date information directly from this repository's content, leading to more accurate and helpful responses.

**Recommended Prompt Structure:**

```
I need to [your task, e.g., 'implement a multi-agent swarm']. Can you show me how using LangGraph? Please use context7, resolve library id `/shak-shat/langgraph4context7`, and then get docs on [specific topic, e.g., 'multi-agent swarm implementation'].
```

### Using MCP Tools with LangGraph

LangGraph can be integrated with MCP tools, including Context7, through the `langchain-mcp-adapters` library:

```python
# Install the required library
# pip install langchain-mcp-adapters

from langchain_mcp_adapters.client import MultiServerMCPClient
from langgraph.prebuilt import create_react_agent

async with MultiServerMCPClient(
    {
        "context7": {
            "command": "npx",
            "args": ["-y", "@upstash/context7-mcp@latest"],
            "transport": "stdio",
        }
    }
) as client:
    agent = create_react_agent(
        "anthropic:claude-3-7-sonnet-latest",
        client.get_tools()
    )
    # To instruct the AI (or your agent logic) to use the specific Context7 library:
    response = await agent.ainvoke(
        {"messages": [{"role": "user", "content": "What are the key features of LangGraph? Use context7 resolve library id /shak-shat/langgraph4context7 and then get docs on its key features."}]}
    )
```

## 🌟 Key Features

### LangGraph Features

- **Stateful Workflows**: Build complex, multi-step reasoning processes
- **Multi-Agent Systems**: Create systems with multiple specialized agents
- **Human-in-the-Loop**: Execution can pause indefinitely to await human feedback
- **Streaming Support**: Real-time streaming of agent state, model tokens, or tool outputs
- **Memory Integration**: Both short-term and long-term memory capabilities

### Context7 Integration Benefits

- **Up-to-date Documentation**: Always access the latest LangGraph APIs and patterns
- **Accurate Code Examples**: Get working code examples that reflect the current version
- **Reliable Answers**: Eliminate hallucinated APIs and outdated patterns
- **Seamless Workflow**: No need to switch between documentation and coding

## 📘 Documentation

This repository serves as documentation for using LangGraph with Context7. The main directories include:

- `/examples`: Working examples of various LangGraph patterns and architectures
- `/docs`: Comprehensive guides on LangGraph concepts and usage patterns
- `/tutorials`: Step-by-step tutorials for building different types of agents

## 🔧 Advanced Usage

### LangGraph Server with Context7

When deploying production LangGraph applications, you can still leverage Context7 during development:

```python
# First develop your graph with Context7
# "How do I configure a LangGraph server for production deployment? use context7"

# Then deploy with LangGraph Server
from langgraph.server import Server

server = Server(graphs={"my_agent": graph})
server.start()
```

### Building Multi-Agent Systems

Use Context7 to stay current with LangGraph's multi-agent patterns:

```python
# "How do I create a multi-agent system with LangGraph? use context7"

# Install the supervisor package
# pip install langgraph-supervisor

from langgraph_supervisor import Supervisor

supervisor = Supervisor(llm, agents=[agent1, agent2, agent3])
result = supervisor.invoke({"query": "Complex multi-step task"})
```

## 🤝 Contributing

Contributions to expand the examples and documentation are welcome! Please feel free to submit pull requests.

## 📄 License

This project is licensed under the MIT License - see the LICENSE file for details. 