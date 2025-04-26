# Updating Graph State from Tools

## Overview
This guide demonstrates how to update a LangGraph's state directly from tools using the Command object, enabling tools to modify both state values and message history.

## Key Concepts
- Tools can update graph state by returning a Command object with update instructions
- State updates can include modifying custom keys and adding messages
- Dynamic prompts can be created based on updated state information
- Personalized responses can be generated using state values updated by tools

## Prerequisites
```python
# Install required packages
%pip install -U langgraph langchain-openai

# Set API keys
import os
import getpass

def _set_if_undefined(var: str):
    if not os.environ.get(var):
        os.environ[var] = getpass.getpass(f"Please provide your {var}")

_set_if_undefined("OPENAI_API_KEY")
```

## Implementation

### Define Sample Data
```python
USER_INFO = [
    {"user_id": "1", "name": "Bob Dylan", "location": "New York, NY"},
    {"user_id": "2", "name": "Taylor Swift", "location": "Beverly Hills, CA"},
]

USER_ID_TO_USER_INFO = {info["user_id"]: info for info in USER_INFO}
```

### Define State and Tool
```python
from langgraph.prebuilt.chat_agent_executor import AgentState
from langgraph.types import Command
from langchain_core.tools import tool
from langchain_core.tools.base import InjectedToolCallId
from langchain_core.messages import ToolMessage
from langchain_core.runnables import RunnableConfig

from typing_extensions import Any, Annotated

class State(AgentState):
    # updated by the tool
    user_info: dict[str, Any]

@tool
def lookup_user_info(
    tool_call_id: Annotated[str, InjectedToolCallId], config: RunnableConfig
):
    """Use this to look up user information to better assist them with their questions."""
    user_id = config.get("configurable", {}).get("user_id")
    if user_id is None:
        raise ValueError("Please provide user ID")

    if user_id not in USER_ID_TO_USER_INFO:
        raise ValueError(f"User '{user_id}' not found")

    user_info = USER_ID_TO_USER_INFO[user_id]
    return Command(
        update={
            # update the state keys
            "user_info": user_info,
            # update the message history
            "messages": [
                ToolMessage(
                    "Successfully looked up user information", tool_call_id=tool_call_id
                )
            ],
        }
    )
```

### Define Dynamic Prompt
```python
def prompt(state: State):
    user_info = state.get("user_info")
    if user_info is None:
        return state["messages"]

    system_msg = (
        f"User name is {user_info['name']}. User lives in {user_info['location']}"
    )
    return [{"role": "system", "content": system_msg}] + state["messages"]
```

### Create the Agent
```python
from langgraph.prebuilt import create_react_agent
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o")

agent = create_react_agent(
    model,
    # pass the tool that can update state
    [lookup_user_info],
    state_schema=State,
    # pass dynamic prompt function
    prompt=prompt,
)
```

## Usage Example
```python
# Run the agent for Bob Dylan
for chunk in agent.stream(
    {"messages": [("user", "hi, what should i do this weekend?")]},
    # provide user ID in the config
    {"configurable": {"user_id": "1"}},
):
    print(chunk)
    print("\n")

# Run the agent for Taylor Swift
for chunk in agent.stream(
    {"messages": [("user", "hi, what should i do this weekend?")]},
    {"configurable": {"user_id": "2"}},
):
    print(chunk)
    print("\n")
```

## Benefits
- Enables tools to directly modify both state and message history
- Allows for dynamic personalization based on retrieved information
- Simplifies state management by handling updates within tools
- Creates more contextually aware agent responses
- Reduces the need for separate state update nodes in the graph

## Considerations
- Tool-based state updates must be handled by either prebuilt components or custom nodes
- Error handling for state updates should be implemented in the tools
- Careful planning of state schema is needed to ensure proper typing
- State updates from tools should maintain data consistency
- Config values must be properly passed to tools that require them
