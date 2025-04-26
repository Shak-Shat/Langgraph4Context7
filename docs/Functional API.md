# LangGraph Functional API

## Overview
The Functional API provides a way to add LangGraph's core capabilities—persistence, memory, human-in-the-loop interactions, and streaming—to your applications with minimal changes to existing code. Unlike graph-based approaches that require restructuring code into explicit pipelines or DAGs, the Functional API works with standard Python control flow constructs like if statements, loops, and function calls.

## Key Concepts
- **@entrypoint**: Marks a function as the starting point of a workflow, managing execution flow and state
- **@task**: Represents a discrete unit of work that can be executed asynchronously within an entrypoint
- **Checkpointing**: Enables persistence by saving execution state for later resumption
- **Determinism**: Ensures workflows can be properly resumed after interruptions
- **Human-in-the-loop**: Allows pausing execution for human input or intervention

## Prerequisites
```python
# Required imports
from langgraph.func import entrypoint, task
from langgraph.types import interrupt
from langgraph.checkpoint.memory import MemorySaver
```

## Implementation

### Basic Structure
The Functional API is built around two primary decorators:

```python
@entrypoint(checkpointer=MemorySaver())
def workflow(input_data):
    # Workflow logic
    result = some_task(input_data).result()
    return result

@task
def some_task(data):
    # Task logic
    return processed_data
```

### Example: Simple Workflow with Human Review
```python
import time

@task
def write_essay(topic: str) -> str:
    """Write an essay about the given topic."""
    # Simulate a long-running task
    time.sleep(1)
    return f"An essay about topic: {topic}"

@entrypoint(checkpointer=MemorySaver())
def workflow(topic: str) -> dict:
    """A simple workflow that writes an essay and asks for a review."""
    essay = write_essay(topic).result()
    
    # Interrupt for human review
    is_approved = interrupt({
        "essay": essay,
        "action": "Please approve/reject the essay",
    })

    return {
        "essay": essay,
        "is_approved": is_approved,
    }
```

## Entrypoint Decorator

### Definition and Basic Usage
An entrypoint encapsulates workflow logic and manages execution flow, including handling long-running tasks and interrupts:

```python
@entrypoint(checkpointer=MemorySaver())
def my_workflow(some_input: dict) -> int:
    # Workflow logic
    return result
```

### Injectable Parameters
You can request access to additional parameters that will be injected at runtime:

```python
from typing import Any
from langgraph.checkpoint import BaseStore
from langgraph.io import StreamWriter
from langchain_core.runnables import RunnableConfig

@entrypoint(checkpointer=MemorySaver())
def workflow(input_data: dict, *, 
             previous: Any = None,       # Access previous state
             store: BaseStore = None,    # For long-term memory
             writer: StreamWriter = None, # For streaming custom data
             config: RunnableConfig = None # For runtime configuration
            ) -> dict:
    # Use the injected parameters
    return result
```

### Execution Methods
You can execute an entrypoint using several methods:

```python
# Define a thread configuration
config = {
    "configurable": {
        "thread_id": "some_thread_id"
    }
}

# Synchronous execution
result = my_workflow.invoke(some_input, config)

# Stream results
for event in my_workflow.stream(some_input, config, stream_mode="updates"):
    print(event)
```

### Resuming After Interruption
```python
from langgraph.types import Command

# Resume with human input
my_workflow.invoke(Command(resume=human_input), config)

# Resume after an error
my_workflow.invoke(None, config)
```

### State Management
When an entrypoint is defined with a checkpointer, it stores information between successive invocations on the same thread:

```python
@entrypoint(checkpointer=MemorySaver())
def counter(increment: int, *, previous: Any = None) -> int:
    previous = previous or 0
    return previous + increment

# Usage across multiple invocations
config = {"configurable": {"thread_id": "counter-thread"}}
counter.invoke(1, config)  # 1 (previous was None)
counter.invoke(2, config)  # 3 (previous was 1 from previous invocation)
```

### Separating Return and Saved Values
Use `entrypoint.final` to decouple what's returned from what's saved in the checkpoint:

```python
@entrypoint(checkpointer=MemorySaver())
def workflow(data: dict, *, previous: Any = None) -> entrypoint.final[dict, int]:
    previous = previous or 0
    # Return the data to the caller but save the count in the checkpoint
    return entrypoint.final(value=data, save=previous + 1)
```

## Task Decorator

### Definition and Purpose
Tasks represent discrete units of work that can be executed asynchronously:

```python
@task
def slow_computation(input_value):
    # Long-running operation
    time.sleep(2)
    return result
```

### Execution Patterns
Tasks return immediately with a future object, which you can resolve:

```python
# Get result synchronously
future = slow_computation(input_value)
result = future.result()

# Or concurrently with multiple tasks
futures = [slow_computation(value) for value in values]
results = [f.result() for f in futures]
```

### When to Use Tasks
Tasks are particularly useful in these scenarios:
- For checkpointing long-running operations
- To encapsulate randomness in human-in-the-loop workflows
- For parallel execution of I/O-bound operations
- To improve observability and monitoring
- For operations that may need to be retried

## Serialization and Determinism

### Serialization Requirements
- Entrypoint inputs/outputs must be JSON-serializable
- Task outputs must be JSON-serializable

### Ensuring Determinism
Encapsulate randomness inside tasks to ensure consistent behavior when resuming:

```python
import random

# Good: Randomness in task
@task
def generate_random():
    return random.randint(1, 100)

@entrypoint(checkpointer=MemorySaver())
def workflow():
    random_value = generate_random().result()
    user_input = interrupt(f"Random value is {random_value}")
    return {"random": random_value, "user_input": user_input}

# Bad: Randomness outside task
@entrypoint(checkpointer=MemorySaver())
def workflow_bad():
    random_value = random.randint(1, 100)  # Will generate new value on resume!
    user_input = interrupt(f"Random value is {random_value}")
    return {"random": random_value, "user_input": user_input}
```

## Common Patterns and Examples

### Multiple Inputs Using Dictionary
```python
@entrypoint(checkpointer=MemorySaver())
def my_workflow(inputs: dict) -> int:
    value = inputs["value"]
    another_value = inputs["another_value"]
    # Process inputs
    return result

# Usage
my_workflow.invoke({"value": 1, "another_value": 2})
```

### Parallel Execution
```python
@task
def add_one(number: int) -> int:
    return number + 1

@entrypoint(checkpointer=MemorySaver())
def process_numbers(numbers: list[int]) -> list[int]:
    futures = [add_one(i) for i in numbers]
    return [f.result() for f in futures]

# Usage
result = process_numbers.invoke([1, 2, 3, 4, 5])  # Returns [2, 3, 4, 5, 6]
```

### Calling Subgraphs
```python
from langgraph.graph import StateGraph

# Create a graph using Graph API
builder = StateGraph()
# ... configure graph
some_graph = builder.compile()

@entrypoint(checkpointer=MemorySaver())
def workflow(input_data: dict) -> dict:
    # Call a graph defined using the Graph API
    result = some_graph.invoke(input_data)
    return {"processed": result}
```

### Calling Other Entrypoints
```python
@entrypoint()  # Will inherit checkpointer from parent
def sub_workflow(data: dict) -> dict:
    return {"processed": data["value"] * 2}

@entrypoint(checkpointer=MemorySaver())
def main_workflow(data: dict) -> dict:
    # Call another entrypoint
    result = sub_workflow.invoke({"value": data["value"]})
    return {"final": result["processed"] + 1}

# Usage
main_workflow.invoke({"value": 10})  # Returns {"final": 21}
```

### Streaming Custom Data
```python
@entrypoint(checkpointer=MemorySaver())
def workflow(data: dict, *, writer: StreamWriter = None) -> dict:
    writer("Starting workflow")  # Write to custom stream
    result = process_data(data["value"]).result()
    writer("Finished processing")  # Write more data
    return {"result": result}

# Consume the stream
for chunk in workflow.stream(
    {"value": 42}, 
    stream_mode=["custom", "updates"],
    config={"configurable": {"thread_id": "stream-demo"}}
):
    print(chunk)
```

## Advanced Use Cases

### Retry Policy for Tasks
```python
import random
from langgraph.types import RetryPolicy

retry_policy = RetryPolicy(retry_on=ValueError)

@task(retry=retry_policy)
def potentially_failing_task():
    # Task that might fail but should be retried
    if random.random() < 0.5:
        raise ValueError("Temporary failure")
    return "Success"
```

### Resuming After Errors
```python
@task
def slow_task():
    # A time-consuming operation
    time.sleep(5)
    return "Slow task completed"

@task
def risky_task():
    # An operation that might fail
    if random.random() < 0.3:
        raise ValueError("Something went wrong")
    return "Risky task completed"

@entrypoint(checkpointer=MemorySaver())
def workflow(data: dict) -> dict:
    slow_result = slow_task().result()
    try:
        risky_result = risky_task().result()  # Might fail
    except Exception as e:
        # Log error, let caller decide to resume
        raise
    return {"completed": True, "results": [slow_result, risky_result]}

# First attempt might fail
config = {"configurable": {"thread_id": "error-resume-demo"}}
try:
    workflow.invoke({"input": "data"}, config)
except Exception:
    # Fix underlying issue, then resume without rerunning slow_task
    workflow.invoke(None, config)
```

### Human-in-the-Loop Workflows
```python
@task
def generate_response(query: str) -> str:
    # Generate an initial response
    return f"Initial response to: {query}"

@task
def generate_revised_response(query: str, original_response: str, feedback: str) -> str:
    # Generate a revised response based on feedback
    return f"Revised response to {query} based on feedback: {feedback}"

@entrypoint(checkpointer=MemorySaver())
def agent_workflow(query: str) -> dict:
    response = generate_response(query).result()
    # Ask for human approval before proceeding
    human_feedback = interrupt({
        "response": response,
        "query": query,
        "request": "Please review this response"
    })
    
    if human_feedback.get("approved", False):
        return {"final_response": response}
    else:
        # Generate new response with feedback
        revised = generate_revised_response(
            query, response, human_feedback.get("comments", "")
        ).result()
        return {"final_response": revised}
```

## Comparison with Graph API

| Feature | Functional API | Graph API |
|---------|---------------|-----------|
| Control flow | Standard Python constructs | Explicit graph structure |
| State management | Function-scoped | Shared across graph |
| Checkpointing | Task results in existing checkpoint | New checkpoint after each superstep |
| Visualization | Not supported | Graph visualization |
| Code complexity | Usually less code | More explicit structure |

## Best Practices

### Code Organization
- Keep tasks focused on single responsibilities
- Structure entrypoints to handle coordination and flow control
- Use meaningful names that reflect the purpose of each task and entrypoint

### Performance and Reliability
- Encapsulate side effects in tasks to prevent duplicate execution
- Place all randomness and non-deterministic operations in tasks
- Design task functions to be idempotent when possible
- Use dictionary inputs for entrypoints that need multiple parameters
- Consider parallel execution for I/O-bound tasks
- Maintain clear error handling and resumption strategies

### Debugging and Testing
- Test tasks individually before integrating them into workflows
- Use the streaming interface to monitor execution progress
- Add logging within tasks to track execution flow
- Write unit tests for deterministic task behavior
- Use simple thread IDs during development for easier tracing
