# Structured Output for Tool-Calling Agents in LangGraph

## Overview
This document explains how to ensure that tool-calling agents return their output in a consistent structured format. Structured output is essential when integrating an agent into a larger software system where downstream components expect data in specific formats. We present two approaches to achieve structured output: binding the output schema as a tool, and using a dedicated structured output LLM.

## Key Concepts
- Structured output in agent responses
- Tool-binding approach (single LLM)
- Two-LLM approach with dedicated structured output
- Conditional routing based on tool calls
- Pydantic models for response schemas

## Prerequisites
```python
from pydantic import BaseModel, Field
from typing import Literal
from langchain_core.tools import tool
from langchain_anthropic import ChatAnthropic
from langgraph.graph import MessagesState, StateGraph, END
from langgraph.prebuilt import ToolNode
from langchain_core.messages import HumanMessage
```

## Implementation

### Defining Response Schema and Tools
First, define the structured output format and any tools your agent will use:

```python
class WeatherResponse(BaseModel):
    """Response schema for weather information"""
    temperature: float = Field(description="The temperature in fahrenheit")
    wind_directon: str = Field(description="The direction of the wind in abbreviated form")
    wind_speed: float = Field(description="The speed of the wind in km/h")

# Inherit 'messages' key from MessagesState
class AgentState(MessagesState):
    # Final structured response from the agent
    final_response: WeatherResponse

@tool
def get_weather(city: Literal["nyc", "sf"]):
    """Use this to get weather information."""
    if city == "nyc":
        return "It is cloudy in NYC, with 5 mph winds in the North-East direction and a temperature of 70 degrees"
    elif city == "sf":
        return "It is 75 degrees and sunny in SF, with 3 mph winds in the South-East direction"
    else:
        raise AssertionError("Unknown city")

tools = [get_weather]
model = ChatAnthropic(model="claude-3-opus-20240229")
```

### Approach 1: Binding Output Schema as a Tool
This approach uses a single LLM but binds the output schema as an additional tool:

```python
# Add the WeatherResponse as a tool
tools = [get_weather, WeatherResponse]

# Force the model to use tools
model_with_response_tool = model.bind_tools(tools, tool_choice="any")

# Define the model node
def call_model(state: AgentState):
    response = model_with_response_tool.invoke(state["messages"])
    return {"messages": [response]}

# Define the response formatter node
def respond(state: AgentState):
    # Extract structured data from the tool call
    weather_tool_call = state["messages"][-1].tool_calls[0]
    response = WeatherResponse(**weather_tool_call["args"])
    
    # Add a tool message for the WeatherResponse tool call
    tool_message = {
        "type": "tool",
        "content": "Here is your structured response",
        "tool_call_id": weather_tool_call["id"],
    }
    
    return {"final_response": response, "messages": [tool_message]}

# Routing function
def should_continue(state: AgentState):
    messages = state["messages"]
    last_message = messages[-1]
    # If there is only one tool call and it is the response tool call, respond to the user
    if (len(last_message.tool_calls) == 1 and 
        last_message.tool_calls[0]["name"] == "WeatherResponse"):
        return "respond"
    # Otherwise continue to the tools node
    else:
        return "continue"
```

### Approach 2: Using a Second LLM for Structured Output
This approach uses two LLMs - one for the agent and one for structured output:

```python
# Prepare the models
model_with_tools = model.bind_tools(tools)
model_with_structured_output = model.with_structured_output(WeatherResponse)

# Define the model node (same as before)
def call_model(state: AgentState):
    response = model_with_tools.invoke(state["messages"])
    return {"messages": [response]}

# Define the structured output response node
def respond(state: AgentState):
    # Call the structured output model with the last tool message
    response = model_with_structured_output.invoke(
        [HumanMessage(content=state["messages"][-2].content)]
    )
    return {"final_response": response}

# Routing function
def should_continue(state: AgentState):
    messages = state["messages"]
    last_message = messages[-1]
    # If there is no tool call, respond to the user
    if not last_message.tool_calls:
        return "respond"
    # Otherwise continue to the tools node
    else:
        return "continue"
```

### Building the Graph (Common for Both Approaches)
The graph structure is the same for both approaches:

```python
# Define the graph
workflow = StateGraph(AgentState)

# Add nodes
workflow.add_node("agent", call_model)
workflow.add_node("respond", respond)
workflow.add_node("tools", ToolNode(tools))

# Set entrypoint
workflow.set_entry_point("agent")

# Add conditional edges
workflow.add_conditional_edges(
    "agent",
    should_continue,
    {
        "continue": "tools",
        "respond": "respond",
    },
)

# Add regular edges
workflow.add_edge("tools", "agent")
workflow.add_edge("respond", END)

# Compile the graph
graph = workflow.compile()
```

## Usage Examples

### Running the Graph
Invoke the graph with a user query:

```python
answer = graph.invoke(
    input={"messages": [("human", "what's the weather in SF?")]}
)["final_response"]

# Result: 
# WeatherResponse(temperature=75.0, wind_directon='SE', wind_speed=3.0)
```

## Benefits and Considerations

### Approach 1: Binding Output Schema as a Tool
**Benefits:**
- Requires only one LLM, reducing cost and latency
- Simpler implementation with fewer model calls

**Considerations:**
- Not guaranteed that the LLM will call the correct tool when needed
- Requires careful routing logic if multiple tools are called
- May need to set `tool_choice="any"` to force tool selection

### Approach 2: Using a Second LLM for Structured Output
**Benefits:**
- More reliable structured output (when `.with_structured_output` works as expected)
- Clearer separation of concerns between agent logic and response formatting

**Considerations:**
- Additional LLM call increases cost and latency
- The agent LLM has no information about the desired output schema
- May fail to collect necessary information for the structured response
