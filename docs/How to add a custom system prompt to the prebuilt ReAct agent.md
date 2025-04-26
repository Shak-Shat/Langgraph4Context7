# Adding a Custom System Prompt to the Prebuilt ReAct Agent

## Overview
This guide demonstrates how to customize the behavior of LangGraph's prebuilt ReAct agent by adding a custom system prompt, enabling you to control the agent's response style, language, and operational context.

## Key Concepts
- **Custom System Prompt**: Modify the system message to control agent behavior and communication style
- **Prebuilt ReAct Agent**: Ready-to-use agent implementation with built-in reasoning capabilities
- **LangChain Tools**: Function-based tools that can be called by the agent

## Prerequisites
- LangGraph and LangChain OpenAI packages installed
- OpenAI API key configured
- Basic understanding of ReAct agents and tools

```python
# Install required packages
pip install -U langgraph langchain-openai

# Set up API key
import os
import getpass

def _set_env(var: str):
    if not os.environ.get(var):
        os.environ[var] = getpass.getpass(f"{var}: ")

_set_env("OPENAI_API_KEY")
```

## Implementation

### Defining Custom Tools
First, create the tools that your agent will use:

```python
from typing import Literal
from langchain_core.tools import tool

@tool
def get_weather(city: Literal["nyc", "sf"]) -> str:
    """Use this to get weather information."""
    if city == "nyc":
        return "It might be cloudy in nyc"
    elif city == "sf":
        return "It's always sunny in sf"
    else:
        raise AssertionError("Unknown city")

tools = [get_weather]
```

### Creating a ReAct Agent with Custom Prompt
Use the `create_react_agent` function with your custom system prompt:

```python
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent

# Initialize the language model
model = ChatOpenAI(model="gpt-4o", temperature=0)

# Define your custom system prompt
prompt = "Respond in Italian"

# Create the ReAct agent with the custom prompt
graph = create_react_agent(model, tools=tools, prompt=prompt)
```

### Using Multiple Customizations
You can combine multiple customizations in your system prompt:

```python
# More advanced system prompt with multiple directives
complex_prompt = """
You are a helpful assistant with the following characteristics:
1. You always respond in Italian
2. You keep responses brief and to the point
3. You present weather information in a poetic style
4. You always greet the user appropriately based on time of day
"""

advanced_graph = create_react_agent(model, tools=tools, prompt=complex_prompt)
```

## Usage Example
Interact with the agent by providing user messages and streaming the responses:

```python
# Define the input message
inputs = {"messages": [("user", "What's the weather in NYC?")]} 

# Helper function to print the streaming output
def print_stream(stream):
    for s in stream:
        message = s["messages"][-1]
        if isinstance(message, tuple):
            print(message)
        else:
            message.pretty_print()

# Run the agent and stream the output
print_stream(graph.stream(inputs, stream_mode="values"))
```

Expected output:

```
================================ Human Message ================================
What's the weather in NYC?

================================ Ai Message ==================================
Tool Calls:
  get_weather (call_b02uzBRrIm2uciJa8zDXCDxT)
 Call ID: call_b02uzBRrIm2uciJa8zDXCDxT
  Args:
    city: nyc

================================= Tool Message ================================
Name: get_weather
It might be cloudy in nyc

================================ Ai Message ==================================
A New York potrebbe essere nuvoloso.
```

## Benefits
- Customize agent behavior without modifying its underlying architecture
- Control response language, tone, and format through simple prompt changes
- Integrate specialized domain knowledge or context for targeted applications
- Maintain the reasoning capabilities of the ReAct framework while adapting output style

## Considerations
- System prompts should be clear and specific to avoid confusing the agent
- Test custom prompts to ensure they don't conflict with tool usage instructions
- Complex directives may occasionally be ignored in favor of completing the primary task
- Different language models may respond differently to the same system prompt

---

This transformed document maintains a concise, knowledge-base-friendly format, retaining core technical details, implementation examples, and practical usage instructions. Let me know if you need refinements!
