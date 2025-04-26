# Multi-Turn Conversations in Multi-Agent Applications

## Overview
This document demonstrates how to enable multi-turn conversations in a LangGraph-based multi-agent application. By leveraging tools, state workflows, and commands, agents can seamlessly manage multi-turn inputs while handing off tasks dynamically.

## Key Concepts
- **Multi-Turn Conversations:** Enables agents to interact with users across multiple steps while recalling past state.
- **Interrupt:** Allows agents to prompt users for input and proceed after receiving responses.
- **Agent Handoffs:** Tasks can be dynamically transferred between connected agents, enabling collaborative outcomes.


## Implementation

### 1. Define Tools for Agents
```python
import random
from langchain_core.tools import tool
from typing_extensions import Literal

@tool
def get_travel_recommendations():
    """Provide recommendations for travel destinations."""
    return random.choice(["Aruba", "Turks and Caicos"])

@tool
def get_hotel_recommendations(location: Literal["Aruba", "Turks and Caicos"]):
    """Suggest hotels for travel destinations."""
    return {
        "Aruba": [
            "The Ritz-Carlton, Aruba (Palm Beach)",
            "Bucuti & Tara Beach Resort (Eagle Beach)",
        ],
        "Turks and Caicos": ["Grace Bay Club", "COMO Parrot Cay"],
    }[location]

@tool(return_direct=True)
def transfer_to_hotel_advisor():
    """Hand off task to the hotel advisor agent."""
    return "Successfully transferred to hotel advisor."

@tool(return_direct=True)
def transfer_to_travel_advisor():
    """Hand off task to the travel advisor agent."""
    return "Successfully transferred to travel advisor."
```

### 2. Define Agents
```python
from langgraph.prebuilt import create_react_agent
from langchain_anthropic import ChatAnthropic

model = ChatAnthropic(model="claude-3-5-sonnet-latest")

# Travel Advisor Agent
travel_advisor_tools = [
    get_travel_recommendations,
    transfer_to_hotel_advisor,
]
travel_advisor = create_react_agent(
    model,
    travel_advisor_tools,
    state_modifier=(
        "You are a travel expert providing destination recommendations. "
        "If hotel recommendations are needed, transfer to hotel advisor."
    ),
)

# Hotel Advisor Agent
hotel_advisor_tools = [
    get_hotel_recommendations,
    transfer_to_travel_advisor,
]
hotel_advisor = create_react_agent(
    model,
    hotel_advisor_tools,
    state_modifier=(
        "You are a hotel expert providing hotel recommendations for destinations. "
        "If travel destinations are required, transfer to travel advisor."
    ),
)
```

### 3. Define Workflow
```python
from langgraph.graph import add_messages
from langgraph.func import entrypoint, task, interrupt
from langchain_core.messages import AIMessage
import uuid

# Helper to generate unique IDs
def string_to_uuid(input_string):
    return str(uuid.uuid5(uuid.NAMESPACE_URL, input_string))

# Define the workflow
@entrypoint()
def multi_turn_graph(messages, previous):
    previous = previous or []
    messages = add_messages(previous, messages)
    call_active_agent = call_travel_advisor

    while True:
        agent_messages = call_active_agent(messages).result()
        messages = add_messages(messages, agent_messages)

        # Detect last AI message and check for tool calls
        ai_msg = next(m for m in reversed(agent_messages) if isinstance(m, AIMessage))
        if not ai_msg.tool_calls:
            user_input = interrupt(value="Ready for user input.")
            human_message = {
                "role": "user",
                "content": user_input,
                "id": string_to_uuid(user_input),
            }
            messages = add_messages(messages, [human_message])
            continue

        # Handle tool calls
        tool_call = ai_msg.tool_calls[-1]
        if tool_call["name"] == "transfer_to_hotel_advisor":
            call_active_agent = call_hotel_advisor
        elif tool_call["name"] == "transfer_to_travel_advisor":
            call_active_agent = call_travel_advisor
        else:
            raise ValueError(f"Unexpected tool call: {tool_call['name']}")

    return entrypoint.final(value=agent_messages[-1], save=messages)

# Define agent task calls
@task
def call_travel_advisor(messages):
    response = travel_advisor.invoke({"messages": messages})
    return response["messages"]

@task
def call_hotel_advisor(messages):
    response = hotel_advisor.invoke({"messages": messages})
    return response["messages"]
```

### 4. Test Multi-Turn Conversations
Example conversation:
```python
from langgraph.types import Command

thread_config = {"configurable": {"thread_id": uuid.uuid4()}}

inputs = [
    {"role": "user", "content": "I want to go to the Caribbean.", "id": str(uuid.uuid4())},
    Command(resume="Can you suggest a hotel?"),
    Command(resume="What activities can I do near the hotel?"),
]

for idx, user_input in enumerate(inputs):
    print(f"--- Conversation Turn {idx + 1} ---")
    for update in multi_turn_graph.stream(
        user_input, config=thread_config, stream_mode="updates"
    ):
        for node_id, value in update.items():
            if isinstance(value, list) and value:
                last_message = value[-1]
                if isinstance(last_message, dict) or last_message.type != "ai":
                    continue
                print(f"{node_id}: {last_message.content}")
```

## Benefits
- **Multi-Agent Collaboration:** Facilitates seamless user interactions between specialized agents.
- **State Management:** Maintains conversation and task history dynamically.
- **Scalability:** Handles complex multi-turn workflows efficiently.

## Considerations
- Test interrupt handling for various types of responses.
- Secure sensitive keys and configurations at all stages.
- Validate agent handoffs and tool calls for each workflow node.

---

This completes the transformation of `How to add multi-turn conversation in a multi-agent application (functional API).md`. Let me know how you'd like to proceed!