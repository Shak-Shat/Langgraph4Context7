# Multi-agent Systems in LangGraph

## Overview
This guide explains how to implement multi-agent systems using LangGraph, allowing you to break complex applications into smaller, specialized agents that communicate and collaborate to solve tasks.

## Key Concepts
- Multi-agent architectures (Network, Supervisor, Hierarchical, Custom)
- Agent handoffs and control flow management
- Inter-agent communication patterns
- State management across multiple agents

## Prerequisites
- LangGraph and LangChain packages
- Understanding of basic agent concepts
- Familiarity with Python typing for route annotation
- Understanding of state management in graphs

## Implementation

### Core Handoffs Mechanism
```python
from langgraph.types import Command
from typing import Literal

def agent(state) -> Command[Literal["agent", "another_agent"]]:
    # Determine which agent to call next
    goto = get_next_agent(...)  # 'agent' / 'another_agent'
    
    # Return a Command object specifying routing and state updates
    return Command(
        # Specify which agent to call next
        goto=goto,
        # Update the graph state
        update={"my_state_key": "my_state_value"}
    )
```

For cross-graph navigation:
```python
def some_node_inside_alice(state):
    return Command(
        goto="bob",
        update={"my_state_key": "my_state_value"},
        # Navigate to parent graph
        graph=Command.PARENT,
    )
```

### Handoffs as Tools
```python
def transfer_to_bob(state):
    """Transfer to bob."""
    return Command(
        goto="bob",
        update={"my_state_key": "my_state_value"},
        graph=Command.PARENT,
    )
```

When implementing with custom tool execution:
```python
def call_tools(state):
    # Process tool calls
    commands = [tools_by_name[tool_call["name"]].invoke(tool_call) 
                for tool_call in tool_calls]
    return commands
```

## Multi-Agent Architectures

### 1. Network Architecture
In this architecture, agents can communicate with any other agent and decide routing.

```python
from typing import Literal
from langchain_openai import ChatOpenAI
from langgraph.types import Command
from langgraph.graph import StateGraph, MessagesState, START, END

model = ChatOpenAI()

def agent_1(state: MessagesState) -> Command[Literal["agent_2", "agent_3", END]]:
    # Determine next agent using LLM
    response = model.invoke(...)
    return Command(
        goto=response["next_agent"],
        update={"messages": [response["content"]]},
    )

def agent_2(state: MessagesState) -> Command[Literal["agent_1", "agent_3", END]]:
    response = model.invoke(...)
    return Command(
        goto=response["next_agent"],
        update={"messages": [response["content"]]},
    )

def agent_3(state: MessagesState) -> Command[Literal["agent_1", "agent_2", END]]:
    response = model.invoke(...)
    return Command(
        goto=response["next_agent"],
        update={"messages": [response["content"]]},
    )

builder = StateGraph(MessagesState)
builder.add_node(agent_1)
builder.add_node(agent_2)
builder.add_node(agent_3)

builder.add_edge(START, "agent_1")
network = builder.compile()
```

### 2. Supervisor Architecture
A supervisor agent decides which specialized agent to call next.

```python
from typing import Literal
from langchain_openai import ChatOpenAI
from langgraph.types import Command
from langgraph.graph import StateGraph, MessagesState, START, END

model = ChatOpenAI()

def supervisor(state: MessagesState) -> Command[Literal["agent_1", "agent_2", END]]:
    response = model.invoke(...)
    return Command(goto=response["next_agent"])

def agent_1(state: MessagesState) -> Command[Literal["supervisor"]]:
    response = model.invoke(...)
    return Command(
        goto="supervisor",
        update={"messages": [response]},
    )

def agent_2(state: MessagesState) -> Command[Literal["supervisor"]]:
    response = model.invoke(...)
    return Command(
        goto="supervisor",
        update={"messages": [response]},
    )

builder = StateGraph(MessagesState)
builder.add_node(supervisor)
builder.add_node(agent_1)
builder.add_node(agent_2)

builder.add_edge(START, "supervisor")
supervisor_graph = builder.compile()
```

### 3. Tool-Calling Supervisor Architecture
Agents are implemented as tools for a ReAct-style supervisor.

```python
from typing import Annotated
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import InjectedState, create_react_agent

model = ChatOpenAI()

# Agent functions that will be called as tools
def agent_1(state: Annotated[dict, InjectedState]):
    response = model.invoke(...)
    return response.content

def agent_2(state: Annotated[dict, InjectedState]):
    response = model.invoke(...)
    return response.content

tools = [agent_1, agent_2]
# Build a supervisor with tool-calling using prebuilt ReAct agent
supervisor = create_react_agent(model, tools)
```

### 4. Hierarchical Architecture
For complex systems with multiple teams of agents under team supervisors.

```python
from typing import Literal
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.types import Command

model = ChatOpenAI()

# Team 1 definitions
def team_1_supervisor(state: MessagesState) -> Command[Literal["team_1_agent_1", "team_1_agent_2", END]]:
    response = model.invoke(...)
    return Command(goto=response["next_agent"])

def team_1_agent_1(state: MessagesState) -> Command[Literal["team_1_supervisor"]]:
    response = model.invoke(...)
    return Command(goto="team_1_supervisor", update={"messages": [response]})

def team_1_agent_2(state: MessagesState) -> Command[Literal["team_1_supervisor"]]:
    response = model.invoke(...)
    return Command(goto="team_1_supervisor", update={"messages": [response]})

team_1_builder = StateGraph(MessagesState)
team_1_builder.add_node(team_1_supervisor)
team_1_builder.add_node(team_1_agent_1)
team_1_builder.add_node(team_1_agent_2)
team_1_builder.add_edge(START, "team_1_supervisor")
team_1_graph = team_1_builder.compile()

# Top-level supervisor
def top_level_supervisor(state: MessagesState) -> Command[Literal["team_1_graph", "team_2_graph", END]]:
    response = model.invoke(...)
    return Command(goto=response["next_team"])

# Main graph connecting everything
builder = StateGraph(MessagesState)
builder.add_node(top_level_supervisor)
builder.add_node("team_1_graph", team_1_graph)
builder.add_node("team_2_graph", team_2_graph)
builder.add_edge(START, "top_level_supervisor")
builder.add_edge("team_1_graph", "top_level_supervisor")
builder.add_edge("team_2_graph", "top_level_supervisor")
graph = builder.compile()
```

### 5. Custom Workflow Architecture
Define explicit or dynamic control flow between agents.

```python
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, MessagesState, START

model = ChatOpenAI()

def agent_1(state: MessagesState):
    response = model.invoke(...)
    return {"messages": [response]}

def agent_2(state: MessagesState):
    response = model.invoke(...)
    return {"messages": [response]}

builder = StateGraph(MessagesState)
builder.add_node(agent_1)
builder.add_node(agent_2)
# Define the flow explicitly
builder.add_edge(START, "agent_1")
builder.add_edge("agent_1", "agent_2")
```

## Communication Strategies

### State Management Options
- **Graph State**: Agents communicate through updates to a shared state
- **Tool Calls**: Agents are invoked as tools with specific inputs/outputs
- **Different State Schemas**: Agents can have specialized state schemas
- **Shared Message List**: Agents communicate through a common message channel

### Sharing Information
1. **Share Full History**: All agents have access to the complete conversation history and reasoning steps
2. **Share Final Results Only**: Agents maintain private reasoning and only share final outputs

## Benefits
- Modularity for easier development, testing, and maintenance
- Specialization of agents for different tasks or domains
- Explicit control over agent communication patterns
- Better handling of complex control flows
- Reduced context window pressure per agent

## Considerations
- Balance complexity between single agent vs. multi-agent architecture
- Design appropriate communication channels between agents
- Manage state effectively across multiple agents
- Consider computational overhead of multi-agent systems
