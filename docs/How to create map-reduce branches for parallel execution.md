# Map-Reduce Pattern for Parallel Execution in LangGraph

## Overview
This document explains how to implement the map-reduce pattern in LangGraph for parallel task processing. This pattern enables breaking a complex task into smaller sub-tasks that can be processed independently, then aggregating the results. Map-reduce is particularly valuable when the number of sub-tasks isn't known in advance and different states need to be passed to parallel processing nodes.

## Key Concepts
- Map-reduce pattern for task decomposition and parallel processing
- The Send API for dynamic distribution of states to nodes
- Conditional edges for managing dynamic workflows
- Annotated state fields for result aggregation
- Structured output with Pydantic models

## Prerequisites
```python
import operator
from typing import Annotated
from typing_extensions import TypedDict
from langchain_anthropic import ChatAnthropic
from langgraph.types import Send
from langgraph.graph import END, StateGraph, START
from pydantic import BaseModel, Field
```

## Implementation

### Defining State Models
First, define the state models for the main graph and sub-task processing:

```python
# Main graph state
class OverallState(TypedDict):
    topic: str
    subjects: list
    # Using operator.add for aggregating results from parallel nodes
    jokes: Annotated[list, operator.add]
    best_selected_joke: str

# State for individual joke generation
class JokeState(TypedDict):
    subject: str
```

### Creating Pydantic Models for Structured Output
Define models for structured outputs from LLM calls:

```python
class Subjects(BaseModel):
    subjects: list[str]

class Joke(BaseModel):
    joke: str

class BestJoke(BaseModel):
    id: int = Field(description="Index of the best joke, starting with 0", ge=0)
```

### Implementing Node Functions
Define the functions that will be used as nodes in the graph:

```python
# Generate a list of subjects based on a topic
def generate_topics(state: OverallState):
    prompt = subjects_prompt.format(topic=state["topic"])
    response = model.with_structured_output(Subjects).invoke(prompt)
    return {"subjects": response.subjects}

# Generate a joke for a given subject
def generate_joke(state: JokeState):
    prompt = joke_prompt.format(subject=state["subject"])
    response = model.with_structured_output(Joke).invoke(prompt)
    return {"jokes": [response.joke]}

# Select the best joke from all generated jokes
def best_joke(state: OverallState):
    jokes = "\n\n".join(state["jokes"])
    prompt = best_joke_prompt.format(topic=state["topic"], jokes=jokes)
    response = model.with_structured_output(BestJoke).invoke(prompt)
    return {"best_selected_joke": state["jokes"][response.id]}
```

### Creating the Distribution Function
The key to map-reduce is the function that distributes tasks to parallel nodes:

```python
# Function to map subjects to joke generation nodes
def continue_to_jokes(state: OverallState):
    # Return a list of Send objects, one for each subject
    # Each Send directs a specific state to a node
    return [Send("generate_joke", {"subject": s}) for s in state["subjects"]]
```

### Building the Graph
Construct the graph with the appropriate nodes and edges:

```python
# Create the state graph
graph = StateGraph(OverallState)

# Add nodes
graph.add_node("generate_topics", generate_topics)
graph.add_node("generate_joke", generate_joke)
graph.add_node("best_joke", best_joke)

# Add edges
graph.add_edge(START, "generate_topics")
graph.add_conditional_edges("generate_topics", continue_to_jokes, ["generate_joke"])
graph.add_edge("generate_joke", "best_joke")
graph.add_edge("best_joke", END)

# Compile the graph
app = graph.compile()
```

## Usage Example
Run the graph with a topic to generate jokes and select the best one:

```python
# Stream the results to see each step
for s in app.stream({"topic": "animals"}):
    print(s)

# Output:
# {'generate_topics': {'subjects': ['Lions', 'Elephants', 'Penguins', 'Dolphins']}}
# {'generate_joke': {'jokes': ["Why don't elephants use computers? They're afraid of the mouse!"]}}
# {'generate_joke': {'jokes': ["Why don't dolphins use smartphones? Because they're afraid of phishing!"]}}
# {'generate_joke': {'jokes': ["Why don't penguins in Britain? Because they're afraid of Wales!"]}}
# {'generate_joke': {'jokes': ["Why don't lions like fast food? Because they can't catch it!"]}}
# {'best_joke': {'best_selected_joke': "Why don't dolphins use smartphones? Because they're afraid of phishing!"}}
```

## Benefits
- Efficient parallel processing of sub-tasks
- Dynamic workflow management without predefined edge count
- Ability to process an unknown number of items
- Independent sub-task execution with separate states
- Result aggregation using annotated state fields

## Considerations
- Proper state management is crucial for effective result aggregation
- The map operation requires careful state design to distribute correctly
- The reduce operation typically uses operator annotations to combine results
- Structured output helps ensure consistent formatting across parallel tasks
- Consider the concurrency model of your runtime environment
