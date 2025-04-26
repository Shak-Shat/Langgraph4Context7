# Tool Call Review in LangGraph

## Overview
This guide demonstrates how to implement human-in-the-loop (HIL) review for agent tool calls in LangGraph applications. The system pauses execution before critical tool calls, allowing human users to review, approve, modify, or provide feedback on the agent's intended actions.

## Key Concepts
- Interruptions for human review at strategic points in graph execution
- Three interaction patterns: approving tool calls, modifying tool calls, and providing natural language feedback
- Command-based resumption with different action paths based on human input
- State management for preserving conversation context during interruptions

## Prerequisites
```python
%pip install --quiet -U langgraph langchain_anthropic
import os
```

## Implementation

### Setting Up the Basic Graph Structure
```python
from typing_extensions import TypedDict, Literal
from langgraph.graph import StateGraph, START, END, MessagesState
from langgraph.checkpoint.memory import MemorySaver
from langgraph.types import Command, interrupt
from langchain_anthropic import ChatAnthropic
from langchain_core.tools import tool
from langchain_core.messages import AIMessage
from IPython.display import Image, display


@tool
def weather_search(city: str):
    """Search for the weather"""
    print("----")
    print(f"Searching for: {city}")
    print("----")
    return "Sunny!"


model = ChatAnthropic(model_name="claude-3-5-sonnet-latest").bind_tools(
    [weather_search]
)


class State(MessagesState):
    """Simple state."""


def call_llm(state):
    return {"messages": [model.invoke(state["messages"])]}


def human_review_node(state) -> Command[Literal["call_llm", "run_tool"]]:
    last_message = state["messages"][-1]
    tool_call = last_message.tool_calls[-1]

    # this is the value we'll be providing via Command(resume=<human_review>)
    human_review = interrupt(
        {
            "question": "Is this correct?",
            # Surface tool calls for review
            "tool_call": tool_call,
        }
    )

    review_action = human_review["action"]
    review_data = human_review.get("data")

    # if approved, call the tool
    if review_action == "continue":
        return Command(goto="run_tool")

    # update the AI message AND call tools
    elif review_action == "update":
        updated_message = {
            "role": "ai",
            "content": last_message.content,
            "tool_calls": [
                {
                    "id": tool_call["id"],
                    "name": tool_call["name"],
                    # This the update provided by the human
                    "args": review_data,
                }
            ],
            # This is important - this needs to be the same as the message you replacing!
            # Otherwise, it will show up as a separate message
            "id": last_message.id,
        }
        return Command(goto="run_tool", update={"messages": [updated_message]})

    # provide feedback to LLM
    elif review_action == "feedback":
        # NOTE: we're adding feedback message as a ToolMessage
        # to preserve the correct order in the message history
        # (AI messages with tool calls need to be followed by tool call messages)
        tool_message = {
            "role": "tool",
            # This is our natural language feedback
            "content": review_data,
            "name": tool_call["name"],
            "tool_call_id": tool_call["id"],
        }
        return Command(goto="call_llm", update={"messages": [tool_message]})


def run_tool(state):
    new_messages = []
    tools = {"weather_search": weather_search}
    tool_calls = state["messages"][-1].tool_calls
    for tool_call in tool_calls:
        tool = tools[tool_call["name"]]
        result = tool.invoke(tool_call["args"])
        new_messages.append(
            {
                "role": "tool",
                "name": tool_call["name"],
                "content": result,
                "tool_call_id": tool_call["id"],
            }
        )
    return {"messages": new_messages}


def route_after_llm(state) -> Literal[END, "human_review_node"]:
    if len(state["messages"][-1].tool_calls) == 0:
        return END
    else:
        return "human_review_node"


builder = StateGraph(State)
builder.add_node(call_llm)
builder.add_node(run_tool)
builder.add_node(human_review_node)
builder.add_edge(START, "call_llm")
builder.add_conditional_edges("call_llm", route_after_llm)
builder.add_edge("run_tool", "call_llm")

# Set up memory
memory = MemorySaver()

# Add
graph = builder.compile(checkpointer=memory)

## Usage Examples

### Example 1: Direct Response (No Tool Call)
When the LLM response doesn't involve any tool calls, the graph proceeds directly to the end:

```python
# Input
initial_input = {"messages": [{"role": "user", "content": "hi!"}]}

# Thread
thread = {"configurable": {"thread_id": "1"}}

# Run the graph until the first interruption
for event in graph.stream(initial_input, thread, stream_mode="updates"):
    print(event)
    print("\n")
```

Output:
```
{'call_llm': {'messages': [AIMessage(content="Hello! I'm here to help you. I can assist you with checking the weather in different cities using the weather search tool. Would you like to know the weather for a specific city? Just let me know which city you're interested in!", additional_kwargs={}, response_metadata={'id': 'msg_01XHvA3ZWpsq4PdyiruWFLBs', 'model': 'claude-3-5-sonnet-20241022', 'stop_reason': 'end_turn', 'stop_sequence': None, 'usage': {'input_tokens': 374, 'output_tokens': 52}}, id='run-c3ff5fea-0135-4d66-8ec1-f8ed6a88356b-0', usage_metadata={'input_tokens': 374, 'output_tokens': 52, 'total_tokens': 426, 'input_token_details': {}})]}}
```

### Example 2: Approving a Tool Call
When the agent makes a tool call, execution pauses for human review. The human can approve the call without changes:

```python
# Input
initial_input = {"messages": [{"role": "user", "content": "what's the weather in sf?"}]}

# Thread
thread = {"configurable": {"thread_id": "2"}}

# Run the graph until the first interruption
for event in graph.stream(initial_input, thread, stream_mode="updates"):
    print(event)
    print("\n")
```

Output (before approval):
```
{'call_llm': {'messages': [AIMessage(content=[{'text': "I'll help you check the weather in San Francisco.", 'type': 'text'}, {'id': 'toolu_01Kn67GmQAA3BEF1cfYdNW3c', 'input': {'city': 'sf'}, 'name': 'weather_search', 'type': 'tool_use'}], additional_kwargs={}, response_metadata={'id': 'msg_013eJXUAEA2ANvYLkDUQFRPo', 'model': 'claude-3-5-sonnet-20241022', 'stop_reason': 'tool_use', 'stop_sequence': None, 'usage': {'input_tokens': 379, 'output_tokens': 65}}, id='run-e8174b94-f681-4688-967f-a32295412f91-0', tool_calls=[{'name': 'weather_search', 'args': {'city': 'sf'}, 'id': 'toolu_01Kn67GmQAA3BEF1cfYdNW3c', 'type': 'tool_call'}], usage_metadata={'input_tokens': 379, 'output_tokens': 65, 'total_tokens': 444, 'input_token_details': {}})]}}

{'__interrupt__': (Interrupt(value={'question': 'Is this correct?', 'tool_call': {'name': 'weather_search', 'args': {'city': 'sf'}, 'id': 'toolu_01Kn67GmQAA3BEF1cfYdNW3c', 'type': 'tool_call'}}, resumable=True, ns=['human_review_node:be252162-5b29-0a98-1ed2-c807c1fc64c6'], when='during'),)}
```

Continue with approval:
```python
# To approve, resume with "continue" action
for event in graph.stream(
    Command(resume={"action": "continue"}),
    thread,
    stream_mode="updates",
):
    print(event)
    print("\n")
```

Output (after approval):
```
{'human_review_node': None}

----
Searching for: sf
----

{'run_tool': {'messages': [{'role': 'tool', 'name': 'weather_search', 'content': 'Sunny!', 'tool_call_id': 'toolu_01Kn67GmQAA3BEF1cfYdNW3c'}]}}

{'call_llm': {'messages': [AIMessage(content="According to the search, it's sunny in San Francisco today!", additional_kwargs={}, response_metadata={'id': 'msg_01FJTbC8oK5fkD73rUBmAtUx', 'model': 'claude-3-5-sonnet-20241022', 'stop_reason': 'end_turn', 'stop_sequence': None, 'usage': {'input_tokens': 457, 'output_tokens': 17}}, id='run-c21af72d-3cc5-4b74-bb7c-fbeb8f88bd6d-0', usage_metadata={'input_tokens': 457, 'output_tokens': 17, 'total_tokens': 474, 'input_token_details': {}})]}}
```

### Example 3: Modifying a Tool Call
The human can modify the parameters of a tool call before execution:

```python
# Input
initial_input = {"messages": [{"role": "user", "content": "what's the weather in sf?"}]}

# Thread
thread = {"configurable": {"thread_id": "3"}}

# Run the graph until the first interruption
for event in graph.stream(initial_input, thread, stream_mode="updates"):
    print(event)
    print("\n")
```

Continue with modification:
```python
# Modify the tool call arguments and continue
for event in graph.stream(
    Command(resume={"action": "update", "data": {"city": "San Francisco, USA"}}),
    thread,
    stream_mode="updates",
):
    print(event)
    print("\n")
```

Output (after modification):
```
{'human_review_node': {'messages': [{'role': 'ai', 'content': [{'text': "I'll help you check the weather in San Francisco.", 'type': 'text'}, {'id': 'toolu_013eUXow3jwM6eekcDJdrjDa', 'input': {'city': 'sf'}, 'name': 'weather_search', 'type': 'tool_use'}], 'tool_calls': [{'id': 'toolu_013eUXow3jwM6eekcDJdrjDa', 'name': 'weather_search', 'args': {'city': 'San Francisco, USA'}}], 'id': 'run-13df3982-ce6d-4fe2-9e5c-ea6ce30a63e4-0'}]}}

----
Searching for: San Francisco, USA
----

{'run_tool': {'messages': [{'role': 'tool', 'name': 'weather_search', 'content': 'Sunny!', 'tool_call_id': 'toolu_013eUXow3jwM6eekcDJdrjDa'}]}}

{'call_llm': {'messages': [AIMessage(content="According to the search, it's sunny in San Francisco right now!", additional_kwargs={}, response_metadata={'id': 'msg_01QssVtxXPqr8NWjYjTaiHqN', 'model': 'claude-3-5-sonnet-20241022', 'stop_reason': 'end_turn', 'stop_sequence': None, 'usage': {'input_tokens': 460, 'output_tokens': 18}}, id='run-8ab865c8-cc9e-4300-8e1d-9eb673e8445c-0', usage_metadata={'input_tokens': 460, 'output_tokens': 18, 'total_tokens': 478, 'input_token_details': {}})]}}
```

### Example 4: Providing Feedback to the Agent
Instead of executing or modifying the tool call, the human can provide natural language feedback to the agent:

```python
# Input
initial_input = {"messages": [{"role": "user", "content": "what's the weather in sf?"}]}

# Thread
thread = {"configurable": {"thread_id": "4"}}

# Run the graph until the first interruption
for event in graph.stream(initial_input, thread, stream_mode="updates"):
    print(event)
    print("\n")
```

Continue with feedback:
```python
# Provide natural language feedback
for event in graph.stream(
    Command(
        resume={
            "action": "feedback",
            "data": "User requested changes: use <city, country> format for location",
        }
    ),
    thread,
    stream_mode="updates",
):
    print(event)
    print("\n")
```

Output (first round of feedback):
```
{'human_review_node': {'messages': [{'role': 'tool', 'content': 'User requested changes: use <city, country> format for location', 'name': 'weather_search', 'tool_call_id': 'toolu_01QxXNTCasnNLQCGAiVoNUBe'}]}}

{'call_llm': {'messages': [AIMessage(content=[{'text': 'Let me try again with the full city name.', 'type': 'text'}, {'id': 'toolu_01WBGTKBWusaPNZYJi5LKmeQ', 'input': {'city': 'San Francisco, USA'}, 'name': 'weather_search', 'type': 'tool_use'}], additional_kwargs={}, response_metadata={'id': 'msg_0141KCdx6KhJmWXyYwAYGvmj', 'model': 'claude-3-5-sonnet-20241022', 'stop_reason': 'tool_use', 'stop_sequence': None, 'usage': {'input_tokens': 468, 'output_tokens': 68}}, id='run-60c8267a-52c7-4b6e-87ca-16aa3bd6266b-0', tool_calls=[{'name': 'weather_search', 'args': {'city': 'San Francisco, USA'}, 'id': 'toolu_01WBGTKBWusaPNZYJi5LKmeQ', 'type': 'tool_call'}], usage_metadata={'input_tokens': 468, 'output_tokens': 68, 'total_tokens': 536, 'input_token_details': {}})]}}

{'__interrupt__': (Interrupt(value={'question': 'Is this correct?', 'tool_call': {'name': 'weather_search', 'args': {'city': 'San Francisco, USA'}, 'id': 'toolu_01WBGTKBWusaPNZYJi5LKmeQ', 'type': 'tool_call'}}, resumable=True, ns=['human_review_node:621fc4a9-bbf1-9a99-f50b-3bf91675234e'], when='during'),)}
```

Now approve the corrected tool call:
```python
# Approve the corrected tool call
for event in graph.stream(
    Command(resume={"action": "continue"}), thread, stream_mode="updates"
):
    print(event)
    print("\n")
```

Output (final execution):
```
{'human_review_node': None}

----
Searching for: San Francisco, USA
----

{'run_tool': {'messages': [{'role': 'tool', 'name': 'weather_search', 'content': 'Sunny!', 'tool_call_id': 'toolu_01WBGTKBWusaPNZYJi5LKmeQ'}]}}

{'call_llm': {'messages': [AIMessage(content='The weather in San Francisco is sunny!', additional_kwargs={}, response_metadata={'id': 'msg_01JrfZd8SYyH51Q8rhZuaC3W', 'model': 'claude-3-5-sonnet-20241022', 'stop_reason': 'end_turn', 'stop_sequence': None, 'usage': {'input_tokens': 549, 'output_tokens': 12}}, id='run-09a198b2-79fa-484d-9d9d-f12432978488-0', usage_metadata={'input_tokens': 549, 'output_tokens': 12, 'total_tokens': 561, 'input_token_details': {}})]}}
```

## Benefits
- Enhanced control over agent actions before execution
- Multiple interaction patterns for different review scenarios
- Human-in-the-loop provides safety guardrails
- Maintains coherent conversation flow even with interruptions
- Combines automation efficiency with human judgment

## Considerations
- Choose the appropriate review method based on the context
- Ensure clear user interface for reviewing tool calls
- Preserve message history structure when adding feedback
- Consider when to interrupt for review vs. allowing automatic execution
