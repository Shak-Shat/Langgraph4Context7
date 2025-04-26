# LangGraph Prebuilt Components

## Overview
LangGraph provides several prebuilt components to simplify agent construction and operation. These components enable rapid development of agents with tool-calling capabilities, validation of structured outputs, and proper handling of state across conversations.

## Key Components

### ReAct Agent Creator
`create_react_agent` is a utility that builds a complete agent graph with tool-calling capabilities.

### Tool Node
`ToolNode` executes tools called by an LLM, supporting both single and parallel tool execution patterns.

### Validation Node
`ValidationNode` validates structured outputs against Pydantic schemas without executing tools.

### State and Store Injection
`InjectedState` and `InjectedStore` annotations enable tools to access graph state and persistent storage.

## Implementation

### Create ReAct Agent
```python
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent
from datetime import datetime

# Define a simple tool
def check_weather(location: str, at_time: datetime | None = None) -> str:
    '''Return the weather forecast for the specified location.'''
    return f"It's always sunny in {location}"

# Create agent with the tool
tools = [check_weather]
model = ChatOpenAI(model="gpt-4o")
graph = create_react_agent(model, tools=tools)

# Use the agent
inputs = {"messages": [("user", "what is the weather in sf")]}
for s in graph.stream(inputs, stream_mode="values"):
    message = s["messages"][-1]
    if isinstance(message, tuple):
        print(message)
    else:
        message.pretty_print()
```

### Using System Prompts
```python
# Simple system prompt
system_prompt = "You are a helpful bot named Fred."
graph = create_react_agent(model, tools, prompt=system_prompt)

# Complex prompt with templates
from langchain_core.prompts import ChatPromptTemplate
prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful bot named Fred."),
    ("placeholder", "{messages}"),
    ("user", "Remember, always be polite!"),
])

graph = create_react_agent(model, tools, prompt=prompt)
```

### Custom State Schema
```python
from typing_extensions import TypedDict
from langgraph.managed import IsLastStep
from langchain_core.messages import BaseMessage
from typing import Annotated
from langgraph.graph.message import add_messages

# Define custom state
prompt = ChatPromptTemplate.from_messages([
    ("system", "Today is {today}"),
    ("placeholder", "{messages}"),
])

class CustomState(TypedDict):
    today: str
    messages: Annotated[list[BaseMessage], add_messages]
    is_last_step: IsLastStep

graph = create_react_agent(model, tools, state_schema=CustomState, prompt=prompt)
```

### Conversation Memory
```python
from langgraph.checkpoint.memory import MemorySaver

# Add memory to persist conversation history
graph = create_react_agent(model, tools, checkpointer=MemorySaver())
config = {"configurable": {"thread_id": "thread-1"}}

# First message
inputs = {"messages": [("user", "What's the weather in SF?")]}
for s in graph.stream(inputs, config, stream_mode="values"):
    message = s["messages"][-1]
    # Display message...

# Follow-up question uses conversation history
inputs2 = {"messages": [("user", "Cool, so then should i go biking today?")]}
for s in graph.stream(inputs2, config, stream_mode="values"):
    message = s["messages"][-1]
    # Display message...
```

### Cross-Thread Store
```python
from langgraph.prebuilt import InjectedStore
from langgraph.store.base import BaseStore
from typing import Annotated

# Define tool that saves to store
def save_memory(memory: str, *, config: dict, store: Annotated[BaseStore, InjectedStore()]) -> str:
    '''Save the given memory for the current user.'''
    user_id = config.get("configurable", {}).get("user_id")
    namespace = ("memories", user_id)
    store.put(namespace, f"memory_{len(store.search(namespace))}", {"data": memory})
    return f"Saved memory: {memory}"

# Access stored information in prompt
def prepare_model_inputs(state: dict, config: dict, store: BaseStore):
    # Retrieve user memories and add them to system message
    user_id = config.get("configurable", {}).get("user_id")
    namespace = ("memories", user_id)
    memories = [m.value["data"] for m in store.search(namespace)]
    system_msg = f"User memories: {', '.join(memories)}"
    return [{"role": "system", "content": system_msg}] + state["messages"]

# Set up agent with store
from langgraph.store.memory import InMemoryStore
store = InMemoryStore()
graph = create_react_agent(
    model, 
    [save_memory], 
    prompt=prepare_model_inputs, 
    store=store, 
    checkpointer=MemorySaver()
)
```

### Using ToolNode Directly
```python
from langchain_anthropic import ChatAnthropic
from langchain_core.tools import tool
from langgraph.graph import StateGraph
from langgraph.prebuilt import ToolNode, tools_condition
from langgraph.graph.message import add_messages
from typing import Annotated
from typing_extensions import TypedDict

# Define a tool
@tool
def divide(a: float, b: float) -> int:
    """Return a / b."""
    return a / b

# Create model and tools
llm = ChatAnthropic(model="claude-3-haiku-20240307")
tools = [divide]

# Define state
class State(TypedDict):
    messages: Annotated[list, add_messages]

# Build graph manually
graph_builder = StateGraph(State)
graph_builder.add_node("tools", ToolNode(tools))
graph_builder.add_node("chatbot", lambda state: {"messages": llm.bind_tools(tools).invoke(state['messages'])})
graph_builder.add_edge("tools", "chatbot")
graph_builder.add_conditional_edges("chatbot", tools_condition)
graph_builder.set_entry_point("chatbot")
graph = graph_builder.compile()
```

### Using ValidationNode
```python
from typing import Literal, Annotated
from typing_extensions import TypedDict
from langchain_anthropic import ChatAnthropic
from pydantic import BaseModel, field_validator
from langgraph.graph import END, START, StateGraph
from langgraph.prebuilt import ValidationNode
from langgraph.graph.message import add_messages

# Define a schema with validation
class SelectNumber(BaseModel):
    a: int
    
    @field_validator("a")
    def a_must_be_meaningful(cls, v):
        if v != 37:
            raise ValueError("Only 37 is allowed")
        return v

# Build graph with validation
builder = StateGraph(Annotated[list, add_messages])
llm = ChatAnthropic(model="claude-3-5-haiku-latest").bind_tools([SelectNumber])
builder.add_node("model", llm)
builder.add_node("validation", ValidationNode([SelectNumber]))
builder.add_edge(START, "model")

# Decision function for validation
def should_validate(state: list) -> Literal["validation", "__end__"]:
    if state[-1].tool_calls:
        return "validation"
    return END

# Decision function for reprompting
def should_reprompt(state: list) -> Literal["model", "__end__"]:
    for msg in state[::-1]:
        # None of the tool calls were errors
        if msg.type == "ai":
            return END
        if msg.additional_kwargs.get("is_error"):
            return "model"
    return END

# Connect edges
builder.add_conditional_edges("model", should_validate)
builder.add_conditional_edges("validation", should_reprompt)
graph = builder.compile()
```

### Injecting State and Store into Tools
```python
from typing import List, Any
from typing_extensions import Annotated, TypedDict
from langchain_core.messages import BaseMessage, AIMessage
from langchain_core.tools import tool
from langgraph.prebuilt import InjectedState, InjectedStore, ToolNode
from langgraph.store.memory import InMemoryStore

# Define tools that use state and store
class AgentState(TypedDict):
    messages: List[BaseMessage]
    foo: str

@tool
def state_tool(x: int, state: Annotated[dict, InjectedState]) -> str:
    '''Do something with state.'''
    if len(state["messages"]) > 2:
        return state["foo"] + str(x)
    else:
        return "not enough messages"

@tool
def foo_tool(x: int, foo: Annotated[str, InjectedState("foo")]) -> str:
    '''Do something else with state.'''
    return foo + str(x + 1)

# Set up store
store = InMemoryStore()
store.put(("values",), "foo", {"bar": 2})

@tool
def store_tool(x: int, my_store: Annotated[Any, InjectedStore()]) -> str:
    '''Do something with store.'''
    stored_value = my_store.get(("values",), "foo").value["bar"]
    return stored_value + x

# Create node with these tools
node = ToolNode([state_tool, foo_tool, store_tool])
```

## Benefits
- Rapid agent construction with minimal boilerplate code
- Built-in conversation memory and cross-conversation storage
- Structured output validation with automatic error handling
- Flexible prompting mechanisms for customization
- Support for complex multi-turn conversations
- Easy integration of external tools and resources

## Considerations
- Performance implications of parallel tool execution
- Proper error handling for failed tool calls
- Schema validation requirements for structured outputs
- Thread and user management for persistent storage
- Timeout handling for long-running operations
- Security concerns with injected state and store access