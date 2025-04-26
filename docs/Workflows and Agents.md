# Workflows and Agents

## Metadata
- **url**: https://langchain-ai.github.io/langgraph/tutorials/workflows/

## Overview
LangGraph provides infrastructure for building both structured workflows and autonomous agents. This guide explains the difference between these approaches and demonstrates implementation patterns for each using LangGraph's capabilities.

## Key Concepts
- **Workflows**: Systems where LLMs and tools are orchestrated through predefined code paths
- **Agents**: Systems where LLMs dynamically direct their own processes and tool usage, maintaining control over task execution
- **Building Blocks**: Augmented LLMs with structured outputs and tool calling capabilities
- **Common Patterns**:
  - Prompt Chaining: Sequential LLM calls where each processes the output of the previous one
  - Parallelization: LLMs working simultaneously on subtasks with aggregated outputs
  - Routing: Classifying input and directing it to specialized followup tasks
  - Tool Use (Agents): Dynamic selection and execution of tools by LLMs

## Prerequisites
- Python with LangGraph and LangChain installed
- Access to LLM APIs that support structured outputs and tool calling (e.g., Anthropic Claude)
- Basic understanding of state management in applications

## Implementation

### 1. Setting up the LLM
```python
import os
from langchain_anthropic import ChatAnthropic

# Set your API key
os.environ["ANTHROPIC_API_KEY"] = "your-api-key"

# Initialize the LLM
llm = ChatAnthropic(model="claude-3-5-sonnet-latest")
```

### 2. Prompt Chaining Implementation
```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

# Define state structure
class State(TypedDict):
    topic: str
    joke: str
    improved_joke: str
    final_joke: str

# Define node functions
def generate_joke(state: State):
    msg = llm.invoke(f"Write a short joke about {state['topic']}")
    return {"joke": msg.content}

def check_punchline(state: State):
    if "?" in state["joke"] or "!" in state["joke"]:
        return "Fail"
    return "Pass"

def improve_joke(state: State):
    msg = llm.invoke(f"Make this joke funnier by adding wordplay: {state['joke']}")
    return {"improved_joke": msg.content}

def polish_joke(state: State):
    msg = llm.invoke(f"Add a surprising twist to this joke: {state['improved_joke']}")
    return {"final_joke": msg.content}

# Build and connect the graph
workflow = StateGraph(State)
workflow.add_node("generate_joke", generate_joke)
workflow.add_node("improve_joke", improve_joke)
workflow.add_node("polish_joke", polish_joke)
workflow.add_edge(START, "generate_joke")
workflow.add_conditional_edges(
    "generate_joke", check_punchline, {"Fail": "improve_joke", "Pass": END}
)
workflow.add_edge("improve_joke", "polish_joke")
workflow.add_edge("polish_joke", END)

# Compile and run
chain = workflow.compile()
result = chain.invoke({"topic": "cats"})
```

### 3. Parallelization Implementation
```python
# Define state with multiple parallel outputs
class State(TypedDict):
    topic: str
    joke: str
    story: str
    poem: str
    combined_output: str

# Define parallel node functions
def call_llm_1(state: State):
    msg = llm.invoke(f"Write a joke about {state['topic']}")
    return {"joke": msg.content}

def call_llm_2(state: State):
    msg = llm.invoke(f"Write a story about {state['topic']}")
    return {"story": msg.content}

def call_llm_3(state: State):
    msg = llm.invoke(f"Write a poem about {state['topic']}")
    return {"poem": msg.content}

def aggregator(state: State):
    combined = f"Here's a story, joke, and poem about {state['topic']}!\n\n"
    combined += f"STORY:\n{state['story']}\n\n"
    combined += f"JOKE:\n{state['joke']}\n\n"
    combined += f"POEM:\n{state['poem']}"
    return {"combined_output": combined}

# Build graph with parallel paths
parallel_builder = StateGraph(State)
parallel_builder.add_node("call_llm_1", call_llm_1)
parallel_builder.add_node("call_llm_2", call_llm_2)
parallel_builder.add_node("call_llm_3", call_llm_3)
parallel_builder.add_node("aggregator", aggregator)
parallel_builder.add_edge(START, "call_llm_1")
parallel_builder.add_edge(START, "call_llm_2")
parallel_builder.add_edge(START, "call_llm_3")
parallel_builder.add_edge("call_llm_1", "aggregator")
parallel_builder.add_edge("call_llm_2", "aggregator")
parallel_builder.add_edge("call_llm_3", "aggregator")
parallel_builder.add_edge("aggregator", END)
```

### 4. Agent (Tool Use) Implementation
```python
from langgraph.graph import MessagesState
from langchain_core.messages import SystemMessage, HumanMessage, ToolMessage
from typing_extensions import Literal

# Define tools
@tool
def add(a: int, b: int) -> int:
    return a + b

@tool
def multiply(a: int, b: int) -> int:
    return a * b

@tool
def divide(a: int, b: int) -> float:
    return a / b

# Set up tools and augmented LLM
tools = [add, multiply, divide]
tools_by_name = {tool.name: tool for tool in tools}
llm_with_tools = llm.bind_tools(tools)

# Define agent nodes
def llm_call(state: MessagesState):
    return {
        "messages": [
            llm_with_tools.invoke(
                [
                    SystemMessage(
                        content="You are a helpful assistant tasked with performing arithmetic."
                    )
                ]
                + state["messages"]
            )
        ]
    }

def tool_node(state: dict):
    result = []
    for tool_call in state["messages"][-1].tool_calls:
        tool = tools_by_name[tool_call["name"]]
        observation = tool.invoke(tool_call["args"])
        result.append(ToolMessage(content=observation, tool_call_id=tool_call["id"]))
    return {"messages": result}

# Routing function
def should_continue(state: MessagesState) -> Literal["environment", END]:
    last_message = state["messages"][-1]
    if last_message.tool_calls:
        return "Action"
    return END

# Build agent graph
agent_builder = StateGraph(MessagesState)
agent_builder.add_node("llm_call", llm_call)
agent_builder.add_node("environment", tool_node)
agent_builder.add_edge(START, "llm_call")
agent_builder.add_conditional_edges(
    "llm_call",
    should_continue,
    {
        "Action": "environment",
        END: END,
    },
)
agent_builder.add_edge("environment", "llm_call")
agent = agent_builder.compile()
```

### 5. Using Pre-built Agents
```python
from langgraph.prebuilt import create_react_agent

# Create a pre-built agent with one line
pre_built_agent = create_react_agent(llm, tools=tools)

# Invoke the agent
messages = [HumanMessage(content="Add 3 and 4.")]
result = pre_built_agent.invoke({"messages": messages})
```

## Benefits

### 1. Persistence
- **Human-in-the-Loop**: Support for interruption and approval of actions
- **Memory Management**: Both short-term (conversational) and long-term memory capabilities

### 2. Streaming
- Multiple options for streaming workflow/agent outputs and intermediate states
- Real-time visibility into processing steps

### 3. Deployment and Debugging
- Easy on-ramp for deployment, observability, and evaluation
- Integrated debugging tools for complex workflows and agent behaviors

## When to Use Each Pattern

- **Prompt Chaining**: When tasks can be cleanly decomposed into fixed subtasks
- **Parallelization**: When subtasks can be run simultaneously, or when multiple perspectives are needed
- **Routing**: For complex tasks with distinct categories that are better handled separately
- **Agents**: When dynamic tool selection and execution are required based on task context
