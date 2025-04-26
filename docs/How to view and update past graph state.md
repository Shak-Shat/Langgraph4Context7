# Time Travel: Viewing and Updating Past Graph State

## Overview
This guide demonstrates how to access and modify the state of a LangGraph at different points in time using checkpointing, enabling powerful capabilities like rewinding execution, branching alternate paths, and implementing human-in-the-loop interventions.

## Key Concepts
- Accessing past states using get_state with checkpoint configurations
- Modifying past states with update_state to create branching paths
- Resuming execution from any saved state point
- Creating alternate execution paths by modifying state values
- Supporting human review of critical operations before they occur

## Prerequisites
```python
# Install required packages
%pip install -U langgraph langchain_openai

# Set API keys
import os
import getpass

def _set_env(var: str):
    if not os.environ.get(var):
        os.environ[var] = getpass.getpass(f"{var}: ")

_set_env("OPENAI_API_KEY")
```

## Implementation

### Define Tools and Agent
```python
# Set up the tools
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langgraph.graph import MessagesState, START
from langgraph.prebuilt import ToolNode
from langgraph.graph import END, StateGraph
from langgraph.checkpoint.memory import MemorySaver

@tool
def play_song_on_spotify(song: str):
    """Play a song on Spotify"""
    # Call the spotify API ...
    return f"Successfully played {song} on Spotify!"

@tool
def play_song_on_apple(song: str):
    """Play a song on Apple Music"""
    # Call the apple music API ...
    return f"Successfully played {song} on Apple Music!"

tools = [play_song_on_apple, play_song_on_spotify]
tool_node = ToolNode(tools)

# Set up the model
model = ChatOpenAI(model="gpt-4o-mini")
model = model.bind_tools(tools, parallel_tool_calls=False)

# Define decision function
def should_continue(state):
    messages = state["messages"]
    last_message = messages[-1]
    # If there is no function call, then we finish
    if not last_message.tool_calls:
        return "end"
    # Otherwise if there is, we continue
    else:
        return "continue"

# Define the function that calls the model
def call_model(state):
    messages = state["messages"]
    response = model.invoke(messages)
    # We return a list, because this will get added to the existing list
    return {"messages": [response]}
```

### Create Graph Structure
```python
# Define a new graph
workflow = StateGraph(MessagesState)

# Define the two nodes we will cycle between
workflow.add_node("agent", call_model)
workflow.add_node("action", tool_node)

# Set the entrypoint as `agent`
workflow.add_edge(START, "agent")

# Add conditional edges
workflow.add_conditional_edges(
    "agent",
    should_continue,
    {
        "continue": "action",
        "end": END,
    },
)

# Add edge from action back to agent
workflow.add_edge("action", "agent")

# Set up memory
memory = MemorySaver()

# Compile graph with checkpointing
app = workflow.compile(checkpointer=memory)
```

## Usage Examples

### Running the Agent
```python
from langchain_core.messages import HumanMessage

config = {"configurable": {"thread_id": "1"}}
input_message = HumanMessage(content="Can you play Taylor Swift's most popular song?")
for event in app.stream({"messages": [input_message]}, config, stream_mode="values"):
    event["messages"][-1].pretty_print()
```

### Replaying a Past State
```python
# Access a specific past state (example assumes we've stored past states in all_states)
to_replay = all_states[2]  # State right before a tool call
print(to_replay.values)    # View state values
print(to_replay.next)      # See next node ('action')

# Replay from this exact point
for event in app.stream(None, to_replay.config):
    for v in event.values():
        print(v)
```

### Branching a Past State
```python
# Get last message in the state
last_message = to_replay.values["messages"][-1]

# Modify tool name to use Spotify instead of Apple
last_message.tool_calls[0]["name"] = "play_song_on_spotify"

# Update state with modified message
branch_config = app.update_state(
    to_replay.config,
    {"messages": [last_message]},
)

# Run with the branched state
for event in app.stream(None, branch_config):
    for v in event.values():
        print(v)
```

### Changing Execution Path
```python
from langchain_core.messages import AIMessage

# Get last message in the state
last_message = to_replay.values["messages"][-1]

# Create completely different message with same ID
new_message = AIMessage(
    content="It's quiet hours so I can't play any music right now!", 
    id=last_message.id
)

# Update state with new message
branch_config = app.update_state(
    to_replay.config,
    {"messages": [new_message]},
)

# View the updated state
branch_state = app.get_state(branch_config)
print(branch_state.values)  # Shows the modified message
print(branch_state.next)    # Shows no next step - execution complete
```

## Benefits
- Enables human review and approval of critical agent actions
- Supports what-if exploration of different execution paths
- Provides undo/redo capabilities for agent workflows
- Allows for manual correction of agent behavior
- Facilitates debugging by examining specific execution states

## Considerations
- State modifications should maintain data consistency and expected formats
- Proper checkpoint IDs must be used to access the intended states
- Branch configurations should be stored if multiple branches need to be maintained
- Memory usage increases with more saved states
- Modifying past states can create divergent execution paths that may be hard to track
