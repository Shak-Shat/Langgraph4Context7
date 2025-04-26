# Multi-Agent Network Implementation with Functional API

## Overview
A guide to implementing a multi-agent network architecture where agents can communicate with each other (many-to-many connections) and decide which agent to call next, using LangGraph's functional API.

## Key Concepts
- Multi-agent systems with fully connected networks (each agent can talk to any other agent)
- Agent handoffs through direct transfer tools
- Task-based agent implementation using functional API
- Dynamic agent selection based on tool calls

## Prerequisites
```python
%pip install -U langgraph langchain-anthropic
import os
```

## Implementation

### Tool Definition
```python
import random
from typing_extensions import Literal
from langchain_core.tools import tool


@tool
def get_travel_recommendations():
    """Get recommendation for travel destinations"""
    return random.choice(["aruba", "turks and caicos"])


@tool
def get_hotel_recommendations(location: Literal["aruba", "turks and caicos"]):
    """Get hotel recommendations for a given destination."""
    return {
        "aruba": [
            "The Ritz-Carlton, Aruba (Palm Beach)"
            "Bucuti & Tara Beach Resort (Eagle Beach)"
        ],
        "turks and caicos": ["Grace Bay Club", "COMO Parrot Cay"],
    }[location]


@tool(return_direct=True)
def transfer_to_hotel_advisor():
    """Ask hotel advisor agent for help."""
    return "Successfully transferred to hotel advisor"


@tool(return_direct=True)
def transfer_to_travel_advisor():
    """Ask travel advisor agent for help."""
    return "Successfully transferred to travel advisor"
```

### Agent Definition
```python
from langchain_core.messages import AIMessage
from langchain_anthropic import ChatAnthropic
from langgraph.prebuilt import create_react_agent
from langgraph.graph import add_messages
from langgraph.func import entrypoint, task

model = ChatAnthropic(model="claude-3-5-sonnet-latest")

# Define travel advisor ReAct agent
travel_advisor_tools = [
    get_travel_recommendations,
    transfer_to_hotel_advisor,
]
travel_advisor = create_react_agent(
    model,
    travel_advisor_tools,
    state_modifier=(
        "You are a general travel expert that can recommend travel destinations (e.g. countries, cities, etc). "
        "If you need hotel recommendations, ask 'hotel_advisor' for help. "
        "You MUST include human-readable response before transferring to another agent."
    ),
)


@task
def call_travel_advisor(messages):
    # You can also add additional logic like changing the input to the agent / output from the agent, etc.
    # NOTE: we're invoking the ReAct agent with the full history of messages in the state
    response = travel_advisor.invoke({"messages": messages})
    return response["messages"]


# Define hotel advisor ReAct agent
hotel_advisor_tools = [get_hotel_recommendations, transfer_to_travel_advisor]
hotel_advisor = create_react_agent(
    model,
    hotel_advisor_tools,
    state_modifier=(
        "You are a hotel expert that can provide hotel recommendations for a given destination. "
        "If you need help picking travel destinations, ask 'travel_advisor' for help."
        "You MUST include human-readable response before transferring to another agent."
    ),
)


@task
def call_hotel_advisor(messages):
    response = hotel_advisor.invoke({"messages": messages})
    return response["messages"]
```

### Workflow Definition
```python
@entrypoint()
def workflow(messages):
    messages = add_messages([], messages)

    call_active_agent = call_travel_advisor
    while True:
        agent_messages = call_active_agent(messages).result()
        messages = add_messages(messages, agent_messages)
        ai_msg = next(m for m in reversed(agent_messages) if isinstance(m, AIMessage))
        if not ai_msg.tool_calls:
            break

        tool_call = ai_msg.tool_calls[-1]
        if tool_call["name"] == "transfer_to_travel_advisor":
            call_active_agent = call_travel_advisor
        elif tool_call["name"] == "transfer_to_hotel_advisor":
            call_active_agent = call_hotel_advisor
        else:
            raise ValueError(f"Expected transfer tool, got '{tool_call['name']}'")

    return messages
```

### Display Helper
```python
from langchain_core.messages import convert_to_messages


def pretty_print_messages(update):
    if isinstance(update, tuple):
        ns, update = update
        # skip parent graph updates in the printouts
        if len(ns) == 0:
            return

        graph_id = ns[-1].split(":")[0]
        print(f"Update from subgraph {graph_id}:")
        print("\n")

    for node_name, node_update in update.items():
        print(f"Update from node {node_name}:")
        print("\n")

        for m in convert_to_messages(node_update["messages"]):
            m.pretty_print()
        print("\n")
```

## Usage Example
```python
for chunk in workflow.stream(
    [
        {
            "role": "user",
            "content": "i wanna go somewhere warm in the caribbean. pick one destination and give me hotel recommendations",
        }
    ],
    subgraphs=True,
):
    pretty_print_messages(chunk)
```

Example output:
```
Update from subgraph call_travel_advisor:


Update from node agent:


==================================[1m Ai Message [0m==================================

[{'text': "I'll help you find a warm Caribbean destination and then get some hotel recommendations for you.\n\nLet me first get some destination recommendations for the Caribbean region.", 'type': 'text'}, {'id': 'toolu_015vT8PkPq1VXvjrDvSpWUwJ', 'input': {}, 'name': 'get_travel_recommendations', 'type': 'tool_use'}]

Tool Calls:
  get_travel_recommendations (toolu_015vT8PkPq1VXvjrDvSpWUwJ)
 Call ID: toolu_015vT8PkPq1VXvjrDvSpWUwJ
  Args:


Update from subgraph call_travel_advisor:


Update from node tools:


=================================[1m Tool Message [0m=================================
Name: get_travel_recommendations

turks and caicos


Update from subgraph call_travel_advisor:


Update from node agent:


==================================[1m Ai Message [0m==================================

[{'text': "Based on the recommendation, I suggest Turks and Caicos! This beautiful British Overseas Territory is known for its stunning white-sand beaches, crystal-clear turquoise waters, and year-round warm weather. Grace Bay Beach in Providenciales is consistently ranked among the world's best beaches. The islands offer excellent snorkeling, diving, and water sports opportunities, plus a relaxed Caribbean atmosphere.\n\nNow, let me connect you with our hotel advisor to get some specific hotel recommendations for Turks and Caicos.", 'type': 'text'}, {'id': 'toolu_01JY7pNNWFuaWoe9ymxFYiPV', 'input': {}, 'name': 'transfer_to_hotel_advisor', 'type': 'tool_use'}]

Tool Calls:
  transfer_to_hotel_advisor (toolu_01JY7pNNWFuaWoe9ymxFYiPV)
 Call ID: toolu_01JY7pNNWFuaWoe9ymxFYiPV
  Args:


Update from subgraph call_travel_advisor:


Update from node tools:


=================================[1m Tool Message [0m=================================
Name: transfer_to_hotel_advisor

Successfully transferred to hotel advisor


Update from subgraph call_hotel_advisor:


Update from node agent:


==================================[1m Ai Message [0m==================================

[{'text': 'Let me get some hotel recommendations for Turks and Caicos:', 'type': 'text'}, {'id': 'toolu_0129ELa7jFocn16bowaGNapg', 'input': {'location': 'turks and caicos'}, 'name': 'get_hotel_recommendations', 'type': 'tool_use'}]

Tool Calls:
  get_hotel_recommendations (toolu_0129ELa7jFocn16bowaGNapg)
 Call ID: toolu_0129ELa7jFocn16bowaGNapg
  Args:
    location: turks and caicos


Update from subgraph call_hotel_advisor:


Update from node tools:


=================================[1m Tool Message [0m=================================
Name: get_hotel_recommendations

["Grace Bay Club", "COMO Parrot Cay"]


Update from subgraph call_hotel_advisor:


Update from node agent:


==================================[1m Ai Message [0m==================================

Here are two excellent hotel options in Turks and Caicos:

1. Grace Bay Club: This luxury resort is located on the world-famous Grace Bay Beach. It offers all-oceanfront suites, exceptional dining options, and personalized service. The resort features adult-only and family-friendly sections, making it perfect for any type of traveler.

2. COMO Parrot Cay: This exclusive private island resort offers the ultimate luxury escape. It's known for its pristine beach, world-class spa, and holistic wellness programs. The resort provides an intimate, secluded experience with top-notch amenities and service.

Would you like more specific information about either of these properties or would you like to explore hotels in another destination?
```

## Benefits
- Decentralized agent control with dynamic handoffs
- Composable and flexible design through functional API
- Clean separation between agent definition and workflow logic
- Easy to extend with new agents and tools

## Considerations
- Must handle transfer mechanics properly (return_direct=True for transfer tools)
- Need appropriate fallback strategies if agent transfer fails
- Consider message history management with multiple agents
- Ensure agent prompts clearly specify when to transfer to other agents
