# Editing Graph State

## Overview
This guide demonstrates how to manually edit graph state in LangGraph applications, enabling human-in-the-loop interaction at specific points in workflow execution. You'll learn how to pause execution at predefined breakpoints, modify the state, and resume execution with the updated values.

## Key Concepts
- **Graph State**: The current data and context being passed between nodes in a LangGraph
- **Breakpoints**: Predefined points where graph execution pauses for inspection or intervention
- **State Updates**: Modifications to the state during execution pauses
- **Thread Management**: Using thread IDs to track specific execution instances
- **Resumption**: Continuing graph execution from the modified state

## Prerequisites
- LangGraph package installed
- Understanding of StateGraph fundamentals
- Familiarity with checkpointing in LangGraph

```python
# Install required packages
pip install -U langgraph langchain-anthropic

# Set up environment variables if needed
import os
os.environ["ANTHROPIC_API_KEY"] = "your-api-key"
```

## Implementation

### Creating a Graph with Breakpoints
First, define a graph with designated breakpoints where execution will pause:

```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver

# Define the state structure
class State(TypedDict):
    input: str

# Define simple node functions
def step_1(state):
    print("---Step 1---")
    return {"input": state["input"] + " - processed by step 1"}

def step_2(state):
    print("---Step 2---")
    return {"input": state["input"] + " - processed by step 2"}

def step_3(state):
    print("---Step 3---")
    return {"input": state["input"] + " - processed by step 3"}

# Build the graph
builder = StateGraph(State)
builder.add_node("step_1", step_1)
builder.add_node("step_2", step_2)
builder.add_node("step_3", step_3)
builder.add_edge(START, "step_1")
builder.add_edge("step_1", "step_2")
builder.add_edge("step_2", "step_3")
builder.add_edge("step_3", END)

# Set up memory for checkpoints and add breakpoint before step_2
memory = MemorySaver()
graph = builder.compile(
    checkpointer=memory, 
    interrupt_before=["step_2"]  # This creates a breakpoint before step_2
)
```

### Running the Graph to a Breakpoint
Execute the graph until it reaches the first breakpoint:

```python
# Initial input state
initial_input = {"input": "hello world"}

# Thread configuration for state tracking
thread_config = {"configurable": {"thread_id": "thread-1"}}

# Run graph until breakpoint
print("Running until breakpoint...")
for event in graph.stream(initial_input, thread_config, stream_mode="values"):
    print(f"State: {event}")

# The graph will pause before executing step_2
```

### Inspecting and Editing the State
Once the graph is paused, you can inspect and modify the current state:

```python
# Get the current state at the breakpoint
current_state = graph.get_state(thread_config).values
print("\nCurrent state at breakpoint:")
print(current_state)

# Update the state with new values
print("\nUpdating state...")
graph.update_state(thread_config, {"input": "hello universe"})

# Verify the updated state
updated_state = graph.get_state(thread_config).values
print("\nUpdated state:")
print(updated_state)
```

### Resuming Execution
Continue execution with the updated state from the breakpoint:

```python
# Resume execution from the breakpoint
print("\nResuming execution with updated state...")
for event in graph.stream(None, thread_config, stream_mode="values"):
    print(f"State: {event}")

# Execution will complete with the modified state
```

### Applying State Editing to Agent Workflows
Use state editing in more complex agent-based workflows with tool calling:

```python
from langchain_anthropic import ChatAnthropic
from langchain_core.tools import tool
from langgraph.graph import MessagesState
from langgraph.prebuilt import ToolNode

# Define a search tool
@tool
def search(query: str) -> str:
    """Search for information online.
    
    Args:
        query: The search query
        
    Returns:
        Search results
    """
    # In a real application, this would call a search API
    return f"Results for '{query}': Sample information about {query}."

# Create a tool node
tools = [search]
tool_node = ToolNode(tools)

# Set up the language model
model = ChatAnthropic(model="claude-3-haiku")

# Define the agent logic
def should_continue(state):
    """Determine if we should continue to tools or end."""
    messages = state["messages"]
    last_message = messages[-1]
    
    # Check if the last message has tool calls
    if hasattr(last_message, "tool_calls") and last_message.tool_calls:
        return "continue"
    return "end"

def call_model(state):
    """Call the model with the current messages."""
    messages = state["messages"]
    response = model.invoke(messages)
    return {"messages": [response]}

# Build the agent workflow
workflow = StateGraph(MessagesState)
workflow.add_node("agent", call_model)
workflow.add_node("action", tool_node)
workflow.add_edge(START, "agent")
workflow.add_conditional_edges(
    "agent", 
    should_continue, 
    {"continue": "action", "end": END}
)
workflow.add_edge("action", "agent")

# Set up memory and add breakpoint before tool execution
memory = MemorySaver()
agent_app = workflow.compile(
    checkpointer=memory, 
    interrupt_before=["action"]  # Add breakpoint before tools run
)
```

### Modifying Tool Calls in an Agent Workflow
Execute the agent workflow and modify tool calls at the breakpoint:

```python
# Thread configuration
thread_config = {"configurable": {"thread_id": "agent-thread-1"}}

# Initial user message
initial_messages = [{"role": "user", "content": "Search for information about climate change"}]

# Run until the breakpoint (before tool execution)
print("Running agent until tool execution...")
for event in agent_app.stream({"messages": initial_messages}, thread_config, stream_mode="values"):
    if "messages" in event and event["messages"]:
        last_message = event["messages"][-1]
        print(f"Agent: {last_message.content}")

# Get current state with tool calls
state = agent_app.get_state(thread_config).values
current_message = state["messages"][-1]

# Display the current tool call
print("\nCurrent tool call:")
if hasattr(current_message, "tool_calls") and current_message.tool_calls:
    print(f"Tool: {current_message.tool_calls[0]['name']}")
    print(f"Arguments: {current_message.tool_calls[0]['args']}")

# Modify the tool call arguments
if hasattr(current_message, "tool_calls") and current_message.tool_calls:
    # Change the search query to be more specific
    current_message.tool_calls[0]["args"]["query"] = "recent effects of climate change in polar regions"
    
    # Update the state with the modified message
    agent_app.update_state(thread_config, {"messages": state["messages"]})
    
    print("\nUpdated tool call:")
    print(f"Tool: {current_message.tool_calls[0]['name']}")
    print(f"Arguments: {current_message.tool_calls[0]['args']}")

# Resume execution with the modified tool call
print("\nResuming agent with modified tool call...")
for event in agent_app.stream(None, thread_config, stream_mode="values"):
    if "messages" in event and event["messages"]:
        last_message = event["messages"][-1]
        print(f"Agent: {last_message.content}")
```

## Usage Example
Here's a practical example of using state editing for content moderation:

```python
from langchain_anthropic import ChatAnthropic
from langgraph.graph import StateGraph, MessagesState
from langgraph.checkpoint.memory import MemorySaver

# Define a simple content generation workflow
model = ChatAnthropic(model="claude-3-haiku")

def generate_content(state):
    """Generate content based on user request."""
    messages = state["messages"]
    response = model.invoke(messages)
    return {"messages": [response]}

# Build the graph with a breakpoint for review
workflow = StateGraph(MessagesState)
workflow.add_node("generate", generate_content)
workflow.add_edge(START, "generate")
workflow.add_edge("generate", END)

# Add memory and breakpoint for human review
memory = MemorySaver()
moderation_app = workflow.compile(
    checkpointer=memory,
    interrupt_before=["generate"]  # Review prompt before generation
)

# Thread config
thread_config = {"configurable": {"thread_id": "moderation-1"}}

# Initial request that might need moderation
initial_messages = [{"role": "user", "content": "Write a story about hackers"}]

# Run until content generation
for event in moderation_app.stream({"messages": initial_messages}, thread_config, stream_mode="values"):
    print("Paused for moderation review")

# Get current state for review
state = moderation_app.get_state(thread_config).values
print(f"Message to review: {state['messages'][0]['content']}")

# Modify the prompt for safety
modified_messages = [{"role": "user", "content": "Write a story about ethical hackers who help companies improve security"}]
moderation_app.update_state(thread_config, {"messages": modified_messages})

# Resume with the safer prompt
for event in moderation_app.stream(None, thread_config, stream_mode="values"):
    if "messages" in event and event["messages"]:
        print(f"Generated content: {event['messages'][-1].content[:100]}...")
```

## Benefits
- **Human Oversight**: Enables human review and intervention in automated workflows
- **Debugging**: Easily inspect and modify state during development and testing
- **Corrective Actions**: Fix errors or modify behavior without restarting the entire workflow
- **Guided Execution**: Steer agents toward more effective or appropriate actions
- **Security Control**: Review sensitive operations before they execute

## Considerations
- **State Consistency**: Ensure your modifications maintain the expected state structure
- **Type Safety**: Modified values should match the state schema defined for the graph
- **Breakpoint Placement**: Choose strategic interruption points that allow meaningful intervention
- **Performance Impact**: Adding many breakpoints may affect execution speed
- **UX Design**: Consider the user experience when designing interfaces for state editing
