# Implementing Agent Handoffs

## Overview
This guide demonstrates how to implement handoffs between specialized agents in LangGraph applications. You'll learn how to create modular agent workflows where control and state are seamlessly passed between different agents with distinct capabilities.

## Key Concepts
- **Handoff Mechanism**: The process of transferring control from one agent to another
- **Command Object**: Special return type that combines routing instructions with state updates
- **Agent Specialization**: Creating purpose-built agents for specific tasks or domains
- **Dynamic Routing**: Conditionally determining the next agent based on the current state
- **State Preservation**: Maintaining context when transferring between agents

## Prerequisites
- LangGraph and LangChain packages installed
- Understanding of StateGraph concepts
- Familiarity with tool definition and usage in LangGraph

```python
# Install required packages
pip install -U langgraph langchain-anthropic

# Set up environment variables
import os
os.environ["ANTHROPIC_API_KEY"] = "your-api-key"
```

## Implementation

### Creating Agent Handoffs with Command
First, let's implement a basic handoff between two specialized calculator agents:

```python
from langchain_anthropic import ChatAnthropic
from langgraph.graph import MessagesState, StateGraph, START, END
from langgraph.types import Command
from langchain_core.tools import tool

# Initialize language model
model = ChatAnthropic(model="claude-3-haiku")

# Define handoff tools
@tool
def transfer_to_multiplication_expert():
    """Ask the multiplication expert for help with multiplication tasks."""
    return "Transferring to multiplication expert"

@tool
def transfer_to_addition_expert():
    """Ask the addition expert for help with addition tasks."""
    return "Transferring to addition expert"

# Addition expert agent
def addition_expert(state: MessagesState):
    """Agent specialized in addition operations."""
    system_prompt = """You are an addition expert. You excel at:
    1. Adding numbers together
    2. Solving equations involving addition
    
    If a task requires multiplication, use the transfer_to_multiplication_expert tool.
    Always solve addition problems completely before transferring.
    """
    
    # Include system prompt with messages
    messages = [{"role": "system", "content": system_prompt}] + state["messages"]
    
    # Use the model with the transfer tool
    ai_msg = model.bind_tools([transfer_to_multiplication_expert]).invoke(messages)
    
    # Check if handoff tool was called
    if hasattr(ai_msg, "tool_calls") and ai_msg.tool_calls:
        tool_msg = {"role": "tool", "content": "Transferred to multiplication expert"}
        # Return Command to route to multiplication expert
        return Command(goto="multiplication_expert", update={"messages": [ai_msg, tool_msg]})
    
    # No handoff, just return the response
    return {"messages": [ai_msg]}

# Multiplication expert agent
def multiplication_expert(state: MessagesState):
    """Agent specialized in multiplication operations."""
    system_prompt = """You are a multiplication expert. You excel at:
    1. Multiplying numbers together
    2. Solving equations involving multiplication
    
    If a task requires addition, use the transfer_to_addition_expert tool.
    Always solve multiplication problems completely before transferring.
    """
    
    # Include system prompt with messages
    messages = [{"role": "system", "content": system_prompt}] + state["messages"]
    
    # Use the model with the transfer tool
    ai_msg = model.bind_tools([transfer_to_addition_expert]).invoke(messages)
    
    # Check if handoff tool was called
    if hasattr(ai_msg, "tool_calls") and ai_msg.tool_calls:
        tool_msg = {"role": "tool", "content": "Transferred to addition expert"}
        # Return Command to route to addition expert
        return Command(goto="addition_expert", update={"messages": [ai_msg, tool_msg]})
    
    # No handoff, just return the response
    return {"messages": [ai_msg]}

# Build the graph
builder = StateGraph(MessagesState)
builder.add_node("addition_expert", addition_expert)
builder.add_node("multiplication_expert", multiplication_expert)

# Define the initial agent (addition expert)
builder.add_edge(START, "addition_expert")

# Both experts can reach END
builder.add_edge("addition_expert", END)
builder.add_edge("multiplication_expert", END)

# Compile the graph
math_graph = builder.compile()
```

### Creating Reusable Handoff Tools
Let's create a more generic approach with reusable handoff tools:

```python
from typing_extensions import Annotated
from typing import Dict, List

# Tool factory for creating handoff tools
def make_handoff_tool(agent_name: str):
    """Creates a tool that hands off to a specific agent.
    
    Args:
        agent_name: The name of the agent to hand off to
        
    Returns:
        A tool function that creates a Command for handoff
    """
    @tool(f"transfer_to_{agent_name}")
    def handoff_to_agent(state: Annotated[dict, "Current state"], tool_call_id: str = None):
        """Transfer the conversation to another agent.
        
        Args:
            state: Current conversation state (automatically injected)
            tool_call_id: ID of the tool call (automatically provided)
            
        Returns:
            Command to transfer control to the target agent
        """
        # Create a tool result message
        tool_message = {
            "role": "tool", 
            "content": f"Transferred to {agent_name}",
            "tool_call_id": tool_call_id
        }
        
        # Command to transfer to target agent, preserving messages
        return Command(
            goto=agent_name,  # Target agent
            graph=Command.PARENT,  # Target is in the parent graph
            update={
                "messages": state["messages"] + [tool_message]
            }
        )
    
    return handoff_to_agent

# Helper function to create a specialized agent
def make_agent(model, tools, system_prompt):
    """Creates an agent with specific tools and system prompt.
    
    Args:
        model: Language model to use
        tools: List of tools available to this agent
        system_prompt: System prompt that defines the agent's role
        
    Returns:
        Agent function that can be added as a node
    """
    def agent_fn(state: MessagesState):
        messages = [{"role": "system", "content": system_prompt}] + state["messages"]
        ai_msg = model.bind_tools(tools).invoke(messages)
        
        # Check if any handoff tools were called
        if hasattr(ai_msg, "tool_calls") and ai_msg.tool_calls:
            for tool_call in ai_msg.tool_calls:
                if tool_call["name"].startswith("transfer_to_"):
                    # This is a handoff - tool will return a Command
                    target_agent = tool_call["name"].replace("transfer_to_", "")
                    tool_result = next(t for t in tools if t.name == tool_call["name"])(
                        state=state,
                        tool_call_id=tool_call["id"]
                    )
                    return tool_result
        
        # No handoff, just return the AI message
        return {"messages": [ai_msg]}
    
    return agent_fn
```

### Building a Multi-Agent System with Handoffs
Now let's build a complete system with arithmetic tools and specialized agents:

```python
# Define the calculation tools
@tool
def add(a: int, b: int) -> int:
    """Add two numbers together.
    
    Args:
        a: First number
        b: Second number
        
    Returns:
        Sum of the two numbers
    """
    return a + b

@tool
def multiply(a: int, b: int) -> int:
    """Multiply two numbers together.
    
    Args:
        a: First number
        b: Second number
        
    Returns:
        Product of the two numbers
    """
    return a * b

# Create the specialized agents with their tools
addition_expert = make_agent(
    model,
    [add, make_handoff_tool("multiplication_expert")],
    system_prompt="""You are an addition expert. Solve addition problems using the add tool.
    For multiplication tasks, transfer to the multiplication expert.
    Always show your work when solving problems."""
)

multiplication_expert = make_agent(
    model,
    [multiply, make_handoff_tool("addition_expert")],
    system_prompt="""You are a multiplication expert. Solve multiplication problems using the multiply tool.
    For addition tasks, transfer to the addition expert.
    Always show your work when solving problems."""
)

# Build the graph
builder = StateGraph(MessagesState)
builder.add_node("addition_expert", addition_expert)
builder.add_node("multiplication_expert", multiplication_expert)
builder.add_edge(START, "addition_expert")  # Start with addition expert
builder.add_edge("addition_expert", END)
builder.add_edge("multiplication_expert", END)
math_system = builder.compile()
```

### Executing the Multi-Agent System
Let's run the system with a task that requires both agents:

```python
from langchain_core.messages import HumanMessage

# Function to pretty print message exchanges
def pretty_print_messages(state):
    if "messages" in state:
        for msg in state["messages"]:
            role = msg.get("role", "")
            content = msg.get("content", "")
            if role and content:
                print(f"{role.upper()}: {content[:100]}..." if len(content) > 100 else f"{role.upper()}: {content}")
        print("-" * 50)

# Initial message requiring both addition and multiplication
initial_message = HumanMessage(content="What is (42 + 18) * 5?")

# Run the graph and observe the handoffs
print("Running math problem with agent handoffs:")
for chunk in math_system.stream({"messages": [initial_message]}, stream_mode="values"):
    pretty_print_messages(chunk)
```

## Usage Example
Here's a complete example of a decision-making system with specialized agents:

```python
from langchain_anthropic import ChatAnthropic
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.types import Command
from langchain_core.messages import HumanMessage, SystemMessage

# Initialize model
model = ChatAnthropic(model="claude-3-haiku")

# Define specialized agents
def finance_expert(state: MessagesState):
    """Agent specialized in financial analysis."""
    system_message = SystemMessage(content="""You are a financial expert. You excel at:
    - Financial analysis and planning
    - Investment recommendations
    - Budget optimization
    
    If a question is about legal matters, use the handoff_to_legal tool.""")
    
    # Combine messages with system prompt
    messages = [system_message] + state["messages"]
    
    # Define the handoff tool
    handoff_tools = [{
        "type": "function",
        "function": {
            "name": "handoff_to_legal",
            "description": "Transfer this conversation to the legal expert",
            "parameters": {"type": "object", "properties": {}}
        }
    }]
    
    # Call the model
    response = model.invoke(messages, tools=handoff_tools)
    
    # Check for handoff
    if hasattr(response, "tool_calls") and response.tool_calls:
        # Handoff to legal
        return Command(
            goto="legal_expert",
            update={"messages": state["messages"] + [response]}
        )
    
    # No handoff
    return {"messages": state["messages"] + [response]}

def legal_expert(state: MessagesState):
    """Agent specialized in legal advice."""
    system_message = SystemMessage(content="""You are a legal expert. You excel at:
    - Legal analysis and compliance
    - Regulatory guidance
    - Contract review
    
    If a question is about finances, use the handoff_to_finance tool.""")
    
    # Combine messages with system prompt
    messages = [system_message] + state["messages"]
    
    # Define the handoff tool
    handoff_tools = [{
        "type": "function",
        "function": {
            "name": "handoff_to_finance",
            "description": "Transfer this conversation to the financial expert",
            "parameters": {"type": "object", "properties": {}}
        }
    }]
    
    # Call the model
    response = model.invoke(messages, tools=handoff_tools)
    
    # Check for handoff
    if hasattr(response, "tool_calls") and response.tool_calls:
        # Handoff to finance
        return Command(
            goto="finance_expert",
            update={"messages": state["messages"] + [response]}
        )
    
    # No handoff
    return {"messages": state["messages"] + [response]}

# Build the graph
builder = StateGraph(MessagesState)
builder.add_node("finance_expert", finance_expert)
builder.add_node("legal_expert", legal_expert)
builder.add_edge(START, "finance_expert")
builder.add_edge("finance_expert", END)
builder.add_edge("legal_expert", END)
advisory_system = builder.compile()

# Test the system
financial_query = HumanMessage(content="What's the best way to structure my stock portfolio?")
legal_query = HumanMessage(content="What regulations should I be aware of when starting a business?")
hybrid_query = HumanMessage(content="What are the tax implications of my investment strategy?")

# Run with a query that may require handoff
result = advisory_system.invoke({"messages": [hybrid_query]})
```

## Benefits
- **Specialization**: Each agent can focus on a narrow domain of expertise
- **Modularity**: Easier maintenance and updates to individual agent components
- **Flexibility**: Dynamic routing based on user needs and agent capabilities
- **Context Preservation**: State is maintained across handoffs for continuity
- **Scalability**: Easily add new specialized agents to the system

## Considerations
- **Handoff Criteria**: Define clear rules for when handoffs should occur
- **Circular Handoffs**: Prevent agents from continuously handing off to each other
- **State Management**: Ensure all relevant context is preserved during handoffs
- **System Prompt Design**: Clear instructions help agents know when to use handoff tools
- **Monitoring**: Track the flow of control to ensure efficient routing
