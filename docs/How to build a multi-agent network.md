# Building a Multi-Agent Network

## Overview
This guide demonstrates how to implement a multi-agent network architecture where multiple specialized agents can communicate directly with each other through a fully connected graph structure. This approach enables dynamic task handoffs and collaboration between agents based on their expertise.

## Key Concepts
- **Agent Handoffs**: Mechanism for transferring control between specialized agents
- **Fully-Connected Network**: Architecture where any agent can communicate with any other agent
- **Command Objects**: Structured control flow directives that manage state updates and routing
- **Tool-Based Communication**: Using tools as the interface for agent-to-agent communication
- **Graph-Based Workflow**: Representing agent interaction patterns as a directed graph

## Prerequisites
- LangGraph and LangChain packages installed
- Anthropic API key configured
- Basic understanding of LangGraph concepts

```python
# Install required packages
pip install -U langgraph langchain-anthropic

# Set up environment variables
import os, getpass
if not os.environ.get("ANTHROPIC_API_KEY"):
    os.environ["ANTHROPIC_API_KEY"] = getpass.getpass("ANTHROPIC_API_KEY: ")
```

## Implementation

### Creating Handoff Tools
First, define tools that enable agents to transfer control to other agents:

```python
from langchain_core.tools import tool
from langchain_core.tools.base import InjectedToolCallId
from langgraph.prebuilt import InjectedState
from typing import Annotated
from langgraph.types import Command

# Simple handoff tools
@tool
def transfer_to_travel_advisor():
    """Ask travel advisor for help with travel destinations."""
    pass

@tool
def transfer_to_hotel_advisor():
    """Ask hotel advisor for help with hotel recommendations."""
    pass

# More flexible handoff tool generator
def make_handoff_tool(agent_name: str):
    """Create a reusable handoff tool for any agent."""
    tool_name = f"transfer_to_{agent_name}"
    
    @tool(tool_name)
    def handoff_to_agent(
        state: Annotated[dict, InjectedState], 
        tool_call_id: Annotated[str, InjectedToolCallId]
    ):
        """Transfer the conversation to another agent."""
        tool_msg = {
            "role": "tool",
            "content": f"Successfully transferred to {agent_name}",
            "name": tool_name,
            "tool_call_id": tool_call_id,
        }
        return Command(
            goto=agent_name, 
            graph=Command.PARENT, 
            update={"messages": state["messages"] + [tool_msg]}
        )
    
    return handoff_to_agent
```

### Defining Agent Nodes
Create specialized agents as nodes in the network:

```python
from langchain_anthropic import ChatAnthropic
from langchain_core.messages import MessagesState

# Initialize the language model
model = ChatAnthropic(model="claude-3-5-sonnet-latest")

# Travel advisor agent node
def travel_advisor(state: MessagesState) -> dict | Command:
    """Agent that specializes in travel destinations."""
    system_prompt = (
        "You are a general travel expert that can recommend travel destinations "
        "(e.g. countries, cities, etc). If you need hotel recommendations, "
        "use the transfer_to_hotel_advisor tool."
    )
    
    # Combine system prompt with conversation history
    messages = [{"role": "system", "content": system_prompt}] + state["messages"]
    
    # Generate response with tool binding
    ai_msg = model.bind_tools([transfer_to_hotel_advisor]).invoke(messages)
    
    # Check if the agent wants to handoff to hotel advisor
    if len(ai_msg.tool_calls) > 0:
        # Create a tool response message
        tool_msg = {
            "role": "tool",
            "content": "Successfully transferred",
            "tool_call_id": ai_msg.tool_calls[-1]["id"],
        }
        # Return command to transfer control
        return Command(
            goto="hotel_advisor", 
            update={"messages": [ai_msg, tool_msg]}
        )
    
    # Return response directly
    return {"messages": [ai_msg]}

# Hotel advisor agent node
def hotel_advisor(state: MessagesState) -> dict | Command:
    """Agent that specializes in hotel recommendations."""
    system_prompt = (
        "You are a hotel expert that can provide hotel recommendations for "
        "a given destination. If you need help selecting travel destinations, "
        "use the transfer_to_travel_advisor tool."
    )
    
    # Combine system prompt with conversation history
    messages = [{"role": "system", "content": system_prompt}] + state["messages"]
    
    # Generate response with tool binding
    ai_msg = model.bind_tools([transfer_to_travel_advisor]).invoke(messages)
    
    # Check if the agent wants to handoff to travel advisor
    if len(ai_msg.tool_calls) > 0:
        # Create a tool response message
        tool_msg = {
            "role": "tool",
            "content": "Successfully transferred",
            "tool_call_id": ai_msg.tool_calls[-1]["id"],
        }
        # Return command to transfer control
        return Command(
            goto="travel_advisor", 
            update={"messages": [ai_msg, tool_msg]}
        )
    
    # Return response directly
    return {"messages": [ai_msg]}
```

### Building the Network
Connect the agents into a graph structure:

```python
from langgraph.graph import StateGraph, START

# Create a state graph with message-based state
builder = StateGraph(MessagesState)

# Add agent nodes
builder.add_node("travel_advisor", travel_advisor)
builder.add_node("hotel_advisor", hotel_advisor)

# Define the starting point (conversation begins with travel advisor)
builder.add_edge(START, "travel_advisor")

# Compile the graph
graph = builder.compile()
```

### Using Prebuilt ReAct Agents
Alternatively, you can use LangGraph's prebuilt ReAct agents for a more structured approach:

```python
import random
from typing_extensions import Literal
from langgraph.prebuilt import create_react_agent

# Define domain-specific tools
@tool
def get_travel_recommendations():
    """Get recommendations for warm travel destinations."""
    return random.choice(["Aruba", "Turks and Caicos"])

@tool
def get_hotel_recommendations(location: Literal["Aruba", "Turks and Caicos"]):
    """Get hotel recommendations for a given destination."""
    return {
        "Aruba": ["The Ritz-Carlton", "Bucuti & Tara Beach Resort"],
        "Turks and Caicos": ["Grace Bay Club", "COMO Parrot Cay"],
    }[location]

# Create React agents with tools and handoff capabilities
travel_advisor_tools = [
    get_travel_recommendations,
    make_handoff_tool(agent_name="hotel_advisor"),
]
travel_advisor_agent = create_react_agent(
    model=ChatAnthropic(model="claude-3-5-sonnet-latest"),
    tools=travel_advisor_tools,
    prompt=(
        "You are a general travel expert that recommends destinations. "
        "Transfer hotel queries to the hotel advisor."
    )
)

hotel_advisor_tools = [
    get_hotel_recommendations,
    make_handoff_tool(agent_name="travel_advisor"),
]
hotel_advisor_agent = create_react_agent(
    model=ChatAnthropic(model="claude-3-5-sonnet-latest"),
    tools=hotel_advisor_tools,
    prompt=(
        "You are a hotel expert that recommends accommodations. "
        "Transfer destination queries to the travel advisor."
    )
)

# Build graph with prebuilt agents
react_builder = StateGraph(MessagesState)
react_builder.add_node("travel_advisor", lambda state: travel_advisor_agent.invoke(state))
react_builder.add_node("hotel_advisor", lambda state: hotel_advisor_agent.invoke(state))
react_builder.add_edge(START, "travel_advisor")

react_graph = react_builder.compile()
```

## Usage Example
Here's how to run the multi-agent network and visualize the handoffs:

```python
from langchain_core.messages import convert_to_messages

# Helper function to display agent responses
def pretty_print_messages(update):
    for node_name, node_update in update.items():
        print(f"\n=== Update from {node_name} ===")
        for m in convert_to_messages(node_update["messages"]):
            role = m["role"].upper()
            print(f"[{role}]: {m['content']}")
        print("=" * 50)

# Run the graph with a sample query
user_query = {"messages": [("user", "Suggest a warm destination in the Caribbean and recommend some luxury hotels there")]}

print("Running multi-agent conversation...")
for update in graph.stream(user_query, subgraphs=True):
    pretty_print_messages(update)
```

## Advanced Example: Multi-Agent Team
Create a more complex network with additional specialized agents:

```python
from langgraph.graph import StateGraph, START, END

# Create additional specialized agents
@tool
def get_activities(location: str):
    """Get activity recommendations for a location."""
    activities = {
        "Aruba": ["Windsurfing at Palm Beach", "Snorkeling at Baby Beach"],
        "Turks and Caicos": ["Diving at Bight Reef", "Kayaking at Chalk Sound"]
    }
    return activities.get(location, ["No activities found"])

# Activities advisor agent
activities_advisor_tools = [
    get_activities,
    make_handoff_tool(agent_name="travel_advisor"),
    make_handoff_tool(agent_name="hotel_advisor")
]
activities_advisor = create_react_agent(
    model=ChatAnthropic(model="claude-3-5-sonnet-latest"),
    tools=activities_advisor_tools,
    prompt="You are an activities expert. Help users find fun things to do at their destination."
)

# Update travel and hotel advisors to include handoff to activities
travel_advisor_tools.append(make_handoff_tool(agent_name="activities_advisor"))
hotel_advisor_tools.append(make_handoff_tool(agent_name="activities_advisor"))

# Recreate the agents with updated tools
travel_advisor_agent = create_react_agent(
    model=ChatAnthropic(model="claude-3-5-sonnet-latest"),
    tools=travel_advisor_tools,
    prompt="You are a travel expert. Transfer activity queries to activities_advisor."
)

hotel_advisor_agent = create_react_agent(
    model=ChatAnthropic(model="claude-3-5-sonnet-latest"),
    tools=hotel_advisor_tools,
    prompt="You are a hotel expert. Transfer activity queries to activities_advisor."
)

# Build the expanded network
team_builder = StateGraph(MessagesState)
team_builder.add_node("travel_advisor", lambda state: travel_advisor_agent.invoke(state))
team_builder.add_node("hotel_advisor", lambda state: hotel_advisor_agent.invoke(state))
team_builder.add_node("activities_advisor", lambda state: activities_advisor.invoke(state))
team_builder.add_edge(START, "travel_advisor")

team_graph = team_builder.compile()
```

## Benefits
- Dynamic routing of queries to appropriate domain experts
- Maintains conversation history across agent handoffs
- Scales easily by adding new specialized agents
- Enables complex problem-solving through collaborative agent teams
- Reduces hallucination by delegating to agents with specific expertise

## Considerations
- Proper prompt engineering is crucial for effective handoffs
- Consider adding validation to ensure tools are being called correctly
- Monitor token usage as multi-agent conversations can be more expensive
- Implement error handling for failed handoffs or tool calls
- Test thoroughly to ensure agents know when to delegate vs. answer directly