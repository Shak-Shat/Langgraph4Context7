# Multi-Agent Multi-Turn Conversation Application

## Overview
This guide demonstrates how to implement multi-turn conversational capabilities in multi-agent applications using LangGraph. The approach enables fluid back-and-forth dialogue between users and different specialized agents, with seamless handoffs between agents.

## Key Concepts
- **Multi-Turn Conversations**: Enable continuous dialogue across multiple interactions
- **Agent Handoffs**: Mechanism for agents to delegate tasks to other specialized agents
- **Interruption Handling**: Collection of user input dynamically during workflow execution
- **State Management**: Maintaining conversation context across multiple agent interactions

## Prerequisites
- LangGraph and LangChain Anthropic packages installed
- Anthropic API key configured
- Basic understanding of LangGraph concepts (StateGraph, nodes, edges)

```python
# Install required packages
pip install -U langgraph langchain-anthropic

# Set up environment variables
import os
import getpass

def _set_env(var: str):
    if not os.environ.get(var):
        os.environ[var] = getpass.getpass(f"{var}: ")

_set_env("ANTHROPIC_API_KEY")
```

## Implementation

### Defining Agent Tools
First, create specialized tools for each agent, including handoff capabilities:

```python
import random
from typing import Annotated, Literal
from langchain_core.tools import tool
from langgraph.types import Command, InjectedState

# Travel advisor tool
@tool
def get_travel_recommendations():
    """Get recommendations for warm travel destinations."""
    return random.choice(["aruba", "turks and caicos"])

# Hotel advisor tool
@tool
def get_hotel_recommendations(location: Literal["aruba", "turks and caicos"]):
    """Get hotel recommendations for a given destination."""
    return {
        "aruba": ["The Ritz-Carlton, Aruba", "Bucuti & Tara Beach Resort"],
        "turks and caicos": ["Grace Bay Club", "COMO Parrot Cay"],
    }[location]

# Handoff tool factory
def make_handoff_tool(*, agent_name: str):
    """Create a tool that facilitates agent handoffs."""
    @tool(f"transfer_to_{agent_name}")
    def handoff_to_agent(state: Annotated[dict, InjectedState], tool_call_id: str):
        return Command(
            goto=agent_name,
            graph=Command.PARENT,
            update={"messages": state["messages"] + [{"role": "tool", "content": f"Handoff to {agent_name}"}]},
        )
    return handoff_to_agent
```

### Creating Specialized Agents
Create agents with different expertise and capabilities:

```python
from langchain_anthropic import ChatAnthropic
from langgraph.prebuilt import create_react_agent
from langchain_core.messages import MessagesState

# Initialize the model
model = ChatAnthropic(model="claude-3-5-sonnet-latest")

# Travel advisor agent
travel_advisor_tools = [
    get_travel_recommendations, 
    make_handoff_tool(agent_name="hotel_advisor")
]
travel_advisor = create_react_agent(
    model, 
    travel_advisor_tools, 
    prompt="You are a travel expert providing destination recommendations. If users ask about hotels, transfer to the hotel_advisor."
)

# Hotel advisor agent
hotel_advisor_tools = [
    get_hotel_recommendations, 
    make_handoff_tool(agent_name="travel_advisor")
]
hotel_advisor = create_react_agent(
    model, 
    hotel_advisor_tools, 
    prompt="You are a hotel expert providing accommodation recommendations. If users ask about destinations, transfer to the travel_advisor."
)
```

### Human Interaction Node
Define a node to handle user input during the conversation:

```python
from langgraph.types import interrupt

def human_node(state: MessagesState) -> Command:
    # Interrupt the workflow to get user input
    user_input = interrupt(value="Ready for user input.")
    
    # Determine which agent triggered the human node
    active_agent = state["metadata"]["langgraph_triggers"][0].split(":")[1]
    
    # Update the state with user input and route back to the active agent
    return Command(
        update={"messages": [{"role": "human", "content": user_input}]},
        goto=active_agent
    )
```

### Building the Conversation Graph
Connect all components into a complete workflow:

```python
import uuid
from langgraph.graph import StateGraph, START

# Create the state graph
builder = StateGraph(MessagesState)

# Add nodes for each agent and human interaction
builder.add_node("travel_advisor", lambda state: travel_advisor.invoke(state))
builder.add_node("hotel_advisor", lambda state: hotel_advisor.invoke(state))
builder.add_node("human", human_node)

# Define the starting point
builder.add_edge(START, "travel_advisor")

# Define how agents connect to the human node
builder.add_edge("travel_advisor", "human")
builder.add_edge("hotel_advisor", "human")

# Compile the graph
graph = builder.compile()
```

## Usage Example
Here's how to run and test a multi-turn conversation that involves both agents:

```python
from langgraph.types import Command

# Initialize conversation with first user message
inputs = {"messages": [{"role": "user", "content": "I want to go somewhere warm in the Caribbean."}]}

# Generate a unique thread ID
thread_id = str(uuid.uuid4())

# First turn
print("User: I want to go somewhere warm in the Caribbean.")
for update in graph.stream(inputs, config={"thread_id": thread_id}, stream_mode="updates"):
    if "interrupt" in update:
        print("\nSystem: Ready for user input")
        break
    if "travel_advisor" in update:
        messages = update["travel_advisor"].get("messages", [])
        if messages and messages[-1]["role"] == "ai":
            print(f"\nTravel Advisor: {messages[-1]['content']}")

# Second turn
print("\nUser: Could you recommend hotels there?")
for update in graph.stream(
    Command(resume="Could you recommend hotels there?"),
    config={"thread_id": thread_id},
    stream_mode="updates"
):
    if "interrupt" in update:
        print("\nSystem: Ready for user input")
        break
    if "hotel_advisor" in update:
        messages = update["hotel_advisor"].get("messages", [])
        if messages and messages[-1]["role"] == "ai":
            print(f"\nHotel Advisor: {messages[-1]['content']}")

# Third turn
print("\nUser: Can you suggest activities near the hotel?")
for update in graph.stream(
    Command(resume="Can you suggest activities near the hotel?"),
    config={"thread_id": thread_id},
    stream_mode="updates"
):
    if "interrupt" in update:
        print("\nSystem: Ready for user input")
        break
    if "hotel_advisor" in update or "travel_advisor" in update:
        agent = "Hotel Advisor" if "hotel_advisor" in update else "Travel Advisor"
        messages = update.get(list(update.keys())[0], {}).get("messages", [])
        if messages and messages[-1]["role"] == "ai":
            print(f"\n{agent}: {messages[-1]['content']}")
```

## Benefits
- Enables natural conversational flow across multiple specialized agents
- Maintains conversation context throughout the entire interaction
- Allows for dynamic routing based on user needs and agent capabilities
- Provides clear separation of concerns between different expert agents

## Considerations
- Consider adding error handling for unexpected user inputs
- Implement logging to track handoffs between agents
- Ensure proper prompt engineering to guide agent behavior during handoffs
- Design the system to handle cases where an agent might not know when to hand off
