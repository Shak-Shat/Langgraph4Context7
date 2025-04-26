# Building a ReAct Agent with LangGraph Functional API

## Overview
This document explains how to implement a ReAct agent from scratch using the LangGraph Functional API. A ReAct agent is a tool-calling agent that iteratively consults a language model, executes tools based on the model's recommendations, and processes results until a final answer is reached. The Functional API provides a more lightweight implementation approach compared to the traditional StateGraph approach.

## Key Concepts
- ReAct pattern for reasoning and acting
- Functional API with tasks and entrypoints
- Tool calling integration with language models
- Parallel tool execution
- Thread-level persistence for conversation memory

## Prerequisites
```python
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langchain_core.messages import ToolMessage
from langgraph.func import entrypoint, task
from langgraph.graph.message import add_messages
from langgraph.checkpoint.memory import MemorySaver
```

## Implementation

### Defining Tools and Model
First, create the tools your agent will use and configure your language model:

```python
@tool
def get_weather(location: str):
    """Call to get the weather from a specific location."""
    # This is a placeholder for the actual implementation
    if any([city in location.lower() for city in ["sf", "san francisco"]]):
        return "It's sunny!"
    elif "boston" in location.lower():
        return "It's rainy!"
    else:
        return f"I am not sure what the weather is in {location}"

tools = [get_weather]
model = ChatOpenAI(model="gpt-4o-mini")

# Create a lookup dictionary for easier tool access
tools_by_name = {tool.name: tool for tool in tools}
```

### Creating Task Functions
Define the core tasks for your agent:

```python
@task
def call_model(messages):
    """Call model with a sequence of messages."""
    response = model.bind_tools(tools).invoke(messages)
    return response

@task
def call_tool(tool_call):
    """Execute a tool based on the tool call information."""
    tool = tools_by_name[tool_call["name"]]
    observation = tool.invoke(tool_call["args"])
    return ToolMessage(content=observation, tool_call_id=tool_call["id"])
```

### Basic Agent Implementation
Create the main entrypoint that orchestrates the tasks:

```python
@entrypoint()
def agent(messages):
    llm_response = call_model(messages).result()
    
    while True:
        if not llm_response.tool_calls:
            break

        # Execute tools in parallel
        tool_result_futures = [
            call_tool(tool_call) for tool_call in llm_response.tool_calls
        ]
        tool_results = [fut.result() for fut in tool_result_futures]

        # Append to message list
        messages = add_messages(messages, [llm_response, *tool_results])

        # Call model again
        llm_response = call_model(messages).result()

    return llm_response
```

### Adding Thread-Level Persistence
Enhance the agent with conversation memory:

```python
# Create a checkpointer for persistence
checkpointer = MemorySaver()

@entrypoint(checkpointer=checkpointer)
def agent(messages, previous):
    # Merge previous conversation with new messages
    if previous is not None:
        messages = add_messages(previous, messages)

    llm_response = call_model(messages).result()
    
    while True:
        if not llm_response.tool_calls:
            break

        # Execute tools in parallel
        tool_result_futures = [
            call_tool(tool_call) for tool_call in llm_response.tool_calls
        ]
        tool_results = [fut.result() for fut in tool_result_futures]

        # Append to message list
        messages = add_messages(messages, [llm_response, *tool_results])

        # Call model again
        llm_response = call_model(messages).result()

    # Generate final response and save messages for next turn
    messages = add_messages(messages, llm_response)
    return entrypoint.final(value=llm_response, save=messages)
```

## Usage Examples

### Basic Agent Invocation
Run the agent with a user message:

```python
user_message = {"role": "user", "content": "What's the weather in san francisco?"}

for step in agent.stream([user_message]):
    for task_name, message in step.items():
        if task_name == "agent":
            continue
        print(f"\n{task_name}:")
        message.pretty_print()

# Output will show the model calling the weather tool and returning the result
```

### Conversational Agent with Persistence
For maintaining conversation context across turns:

```python
# Define a thread ID for the conversation
config = {"configurable": {"thread_id": "1"}}

# First turn
user_message = {"role": "user", "content": "What's the weather in san francisco?"}
response = agent.invoke([user_message], config)

# Follow-up question
follow_up = {"role": "user", "content": "How does it compare to Boston, MA?"}
response = agent.invoke([follow_up], config)
```

## Benefits
- Simplified implementation through functional programming paradigm
- Built-in parallel tool execution
- Easy to add conversation memory
- Lightweight alternative to StateGraph approach
- Clear separation of task responsibilities
- Intuitive code structure for simple agents

## Considerations
- Choose an appropriate checkpointer based on your persistence needs
- Configure thread IDs consistently to maintain conversation context
- Parallel tool execution improves efficiency but may not be desired in all cases
- The Functional API is best suited for simpler agent patterns
