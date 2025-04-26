# Plan-and-Execute Agent Architecture

## Overview
This guide demonstrates how to create a "plan-and-execute" style agent using LangGraph. This architecture first develops a multi-step plan, then executes each step sequentially, with the ability to revisit and modify the plan based on execution results.

## Key Concepts
- Multi-step planning before execution
- Step-by-step task completion
- Dynamic plan revision based on execution results
- Separation of planning and execution models

## Prerequisites
- LangGraph and LangChain packages
- OpenAI API access
- Tavily API for search functionality (optional)
- Understanding of agent architectures

## Implementation

### Setup Dependencies and Tools
```python
from langchain_community.tools.tavily_search import TavilySearchResults
from langchain import hub
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent
from pydantic import BaseModel, Field
from typing import Annotated, List, Tuple, Literal, Union
from typing_extensions import TypedDict
import operator
from langchain_core.prompts import ChatPromptTemplate
from langgraph.graph import StateGraph, START, END
```

### Define the Tools
```python
# Create a search tool for execution
tools = [TavilySearchResults(max_results=3)]
```

### Create the Execution Agent
```python
# Use a ReAct-style agent for step execution
llm = ChatOpenAI(model="gpt-4-turbo-preview")
prompt = "You are a helpful assistant."
agent_executor = create_react_agent(llm, tools, prompt=prompt)
```

### Define the State Structure
```python
class PlanExecute(TypedDict):
    input: str
    plan: List[str]
    past_steps: Annotated[List[Tuple], operator.add]
    response: str
```

### Implement Plan Creation Models

#### Plan Model
```python
class Plan(BaseModel):
    """Plan to follow in future"""
    steps: List[str] = Field(
        description="different steps to follow, should be in sorted order"
    )

planner_prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            """For the given objective, come up with a simple step by step plan. \
This plan should involve individual tasks, that if executed correctly will yield the correct answer. Do not add any superfluous steps. \
The result of the final step should be the final answer. Make sure that each step has all the information needed - do not skip steps.""",
        ),
        ("placeholder", "{messages}"),
    ]
)

planner = planner_prompt | ChatOpenAI(
    model="gpt-4o", temperature=0
).with_structured_output(Plan)
```

#### Re-Planning Model
```python
class Response(BaseModel):
    """Response to user."""
    response: str

class Act(BaseModel):
    """Action to perform."""
    action: Union[Response, Plan] = Field(
        description="Action to perform. If you want to respond to user, use Response. "
        "If you need to further use tools to get the answer, use Plan."
    )

replanner_prompt = ChatPromptTemplate.from_template(
    """For the given objective, come up with a simple step by step plan. \
This plan should involve individual tasks, that if executed correctly will yield the correct answer. Do not add any superfluous steps. \
The result of the final step should be the final answer. Make sure that each step has all the information needed - do not skip steps.

Your objective was this:
{input}

Your original plan was this:
{plan}

You have currently done the follow steps:
{past_steps}

Update your plan accordingly. If no more steps are needed and you can return to the user, then respond with that. Otherwise, fill out the plan. Only add steps to the plan that still NEED to be done. Do not return previously done steps as part of the plan."""
)

replanner = replanner_prompt | ChatOpenAI(
    model="gpt-4o", temperature=0
).with_structured_output(Act)
```

### Create Graph Nodes

#### Execute Step
```python
async def execute_step(state: PlanExecute):
    plan = state["plan"]
    plan_str = "\n".join(f"{i+1}. {step}" for i, step in enumerate(plan))
    task = plan[0]
    task_formatted = f"""For the following plan:
{plan_str}\n\nYou are tasked with executing step {1}, {task}."""
    agent_response = await agent_executor.ainvoke(
        {"messages": [("user", task_formatted)]}
    )
    return {
        "past_steps": [(task, agent_response["messages"][-1].content)],
    }
```

#### Plan Step
```python
async def plan_step(state: PlanExecute):
    plan = await planner.ainvoke({"messages": [("user", state["input"])]})
    return {"plan": plan.steps}
```

#### Replan Step
```python
async def replan_step(state: PlanExecute):
    output = await replanner.ainvoke(state)
    if isinstance(output.action, Response):
        return {"response": output.action.response}
    else:
        return {"plan": output.action.steps}
```

#### Decision Function
```python
def should_end(state: PlanExecute):
    if "response" in state and state["response"]:
        return END
    else:
        return "agent"
```

### Build the Graph
```python
workflow = StateGraph(PlanExecute)

# Add nodes
workflow.add_node("planner", plan_step)
workflow.add_node("agent", execute_step)
workflow.add_node("replan", replan_step)

# Define workflow paths
workflow.add_edge(START, "planner")
workflow.add_edge("planner", "agent")
workflow.add_edge("agent", "replan")
workflow.add_conditional_edges(
    "replan",
    should_end,
    ["agent", END],
)

# Compile the graph
app = workflow.compile()
```

## Usage Example
```python
# Configure recursion limit to prevent infinite loops
config = {"recursion_limit": 50}

# Define the input question
inputs = {"input": "what is the hometown of the mens 2024 Australia open winner?"}

# Execute the workflow
async for event in app.astream(inputs, config=config):
    for k, v in event.items():
        if k != "__end__":
            print(v)
```

## Benefits
- Explicit long-term planning capability
- Ability to use different models for planning vs. execution
- Greater control over complex multi-step tasks
- Improved handling of tasks requiring structured approaches

## Considerations
- Sequential execution can increase total processing time
- Consider implementing DAG-based task management for parallel operations
- Plan quality depends heavily on the planning model's capabilities
- May require more computational resources than simpler agent architectures
