# Human-in-the-Loop in LangGraph

## Overview
Human-in-the-loop (HITL) functionality in LangGraph enables the integration of human input into automated workflows. This pattern allows for human decisions, validation, or corrections at key stages of execution, enhancing reliability and accuracy in LLM-based applications where errors may have significant consequences.

## Key Concepts
- Interrupt mechanism for pausing and resuming graph execution
- Command primitive for controlling state and flow after human input
- Design patterns for common human interaction scenarios
- State management during interruption and resumption

## Prerequisites
- LangGraph setup with a properly configured checkpointer
- Understanding of graph execution flow
- Knowledge of state management in LangGraph

## Implementation

### Basic Interrupt Pattern

The `interrupt` function is the core mechanism enabling human-in-the-loop workflows:

```python
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import MemorySaver

def human_node(state):
    value = interrupt(
        # Any JSON serializable value to surface to the human
        {
            "text_to_revise": state["some_text"]
        }
    )
    
    # Update state with human's input
    return {
        "some_text": value
    }

# Create graph with required checkpointer
checkpointer = MemorySaver()
graph = graph_builder.compile(checkpointer=checkpointer)

# Run graph until interrupt occurs
thread_config = {"configurable": {"thread_id": "some_id"}}
graph.invoke(input_data, config=thread_config)

# Resume graph with human input
graph.invoke(Command(resume="Human edited text"), config=thread_config)
```

### Design Patterns

#### Approve or Reject Pattern

```python
from typing import Literal
from langgraph.types import interrupt, Command

def human_approval(state) -> Command[Literal["proceed_node", "alternate_node"]]:
    is_approved = interrupt(
        {
            "question": "Is this correct?",
            "llm_output": state["llm_output"]
        }
    )
    
    if is_approved:
        return Command(goto="proceed_node")
    else:
        return Command(goto="alternate_node")

# Add to graph and connect to relevant nodes
graph_builder.add_node("human_approval", human_approval)
graph = graph_builder.compile(checkpointer=checkpointer)

# Resume with approval or rejection
thread_config = {"configurable": {"thread_id": "some_id"}}
graph.invoke(Command(resume=True), config=thread_config)  # Approve
# Or
# graph.invoke(Command(resume=False), config=thread_config)  # Reject
```

#### Review and Edit State Pattern

```python
from langgraph.types import interrupt

def human_editing(state):
    result = interrupt(
        {
            "task": "Review the output from the LLM and make necessary edits",
            "llm_generated_summary": state["llm_generated_summary"]
        }
    )
    
    # Update state with edited content
    return {
        "llm_generated_summary": result["edited_text"] 
    }

# Resume with edited content
graph.invoke(
    Command(resume={"edited_text": "The improved text"}), 
    config=thread_config
)
```

#### Review Tool Calls Pattern

```python
def human_review_node(state) -> Command[Literal["call_llm", "run_tool"]]:
    human_review = interrupt(
        {
            "question": "Is this tool call correct?",
            "tool_call": tool_call
        }
    )
    
    review_action, review_data = human_review
    
    if review_action == "continue":
        return Command(goto="run_tool")
    elif review_action == "update":
        updated_msg = get_updated_msg(review_data)
        return Command(goto="run_tool", update={"messages": [updated_message]})
    elif review_action == "feedback":
        feedback_msg = get_feedback_msg(review_data)
        return Command(goto="call_llm", update={"messages": [feedback_msg]})
```

#### Multi-turn Conversation Pattern

```python
from langgraph.types import interrupt

def human_input(state):
    human_message = interrupt("human_input")
    return {
        "messages": [
            {
                "role": "human",
                "content": human_message
            }
        ]
    }

def agent(state):
    # Agent logic processing messages
    # ...

# Connect nodes for conversation flow
graph_builder.add_node("human_input", human_input)
graph_builder.add_edge("human_input", "agent")
graph = graph_builder.compile(checkpointer=checkpointer)

# Resume with human message
graph.invoke(
    Command(resume="Hello, can you help me with something?"),
    config=thread_config
)
```

#### Input Validation Pattern

```python
from langgraph.types import interrupt

def human_node_with_validation(state):
    """Human node that validates input."""
    question = "What is your age?"
    
    while True:
        answer = interrupt(question)
        
        # Validate answer
        if not isinstance(answer, int) or answer < 0:
            question = f"'{answer}' is not a valid age. What is your age?"
            answer = None
            continue
        else:
            # Valid answer, proceed
            break
    
    print(f"The human is {answer} years old.")
    return {
        "age": answer
    }
```

### The Command Primitive

The Command primitive enables control over resumption after an interrupt:

```python
# Simple resumption with value
graph.invoke(Command(resume="User input"), thread_config)

# Resumption with state update
graph.invoke(
    Command(
        resume="Yes, proceed",  # Required when resuming from an interrupt
        update={"additional_info": "Extra context"}
    ), 
    thread_config
)

# Resumption with navigation control
graph.invoke(
    Command(
        resume="User choice",
        goto="specific_node"  # Direct flow to specific node
    ),
    thread_config
)
```

### Retrieving Interrupt Information

When using `invoke` or `ainvoke`, you need to use `get_state` to access interrupt information:

```python
# Run graph to interrupt
result = graph.invoke(inputs, thread_config)

# Get graph state containing interrupt details
state = graph.get_state(thread_config)
print(state.values)  # Current state values
print(state.tasks)   # Pending tasks including interrupt information

# Resume with appropriate input
graph.invoke(Command(resume="User response"), thread_config)
```

## Usage Example

```python
from typing import TypedDict, Optional
from langgraph.graph import StateGraph
from langgraph.checkpoint.memory import MemorySaver
from langgraph.types import interrupt, Command

# Define state type
class ReviewState(TypedDict):
    content: str
    approved: Optional[bool]
    feedback: Optional[str]

# Create nodes
def generate_content(state: ReviewState):
    # Generate content with an LLM or other process
    return {"content": "This is AI-generated content for review"}

def human_review(state: ReviewState):
    # Request human review
    review_result = interrupt({
        "content_to_review": state["content"],
        "instructions": "Please review this content and approve or provide feedback"
    })
    
    # Process review result
    if review_result["approved"]:
        return {"approved": True}
    else:
        return {"approved": False, "feedback": review_result["feedback"]}

def finalize(state: ReviewState):
    # Process based on approval status
    if state["approved"]:
        return {"status": "Content published"}
    else:
        # Regenerate with feedback
        return {"status": "Feedback received", "revision_needed": True}

# Build graph
builder = StateGraph(ReviewState)
builder.add_node("generate", generate_content)
builder.add_node("review", human_review)
builder.add_node("finalize", finalize)

# Add edges
builder.add_edge("generate", "review")
builder.add_edge("review", "finalize")

# Compile with checkpointer
checkpointer = MemorySaver()
graph = builder.compile(checkpointer=checkpointer)

# Run until interrupt
thread_id = "review_session_123"
thread_config = {"configurable": {"thread_id": thread_id}}
graph.invoke({}, config=thread_config)

# Resume with human review
graph.invoke(
    Command(resume={
        "approved": False, 
        "feedback": "Please make the tone more conversational"
    }), 
    config=thread_config
)
```

## Benefits
- Enhanced reliability through human verification
- Flexibility in workflow design with approval gates
- Improved output quality through expert editing
- Ability to handle edge cases through human intervention
- Support for complex interaction patterns like multi-turn conversations

## Considerations

### Execution Flow
When resuming from an interrupt, execution restarts from the beginning of the node containing the interrupt:

```python
def node(state):
    print("This code runs TWICE when resumed")  # Re-executes on resume
    value = interrupt("Need input")
    print("This code runs ONCE when resumed")   # Runs only after interrupt resolution
```

### Side Effects
Code with side effects should be placed after the interrupt to avoid duplication:

```python
# BAD: API call will execute twice
def problematic_node(state):
    api_call()  # Called again when resumed
    value = interrupt("Need input")
    
# GOOD: API call executes only after interrupt is resolved
def better_node(state):
    value = interrupt("Need input")
    api_call()  # Called only once after resume
```

### Multiple Interrupts
When using multiple interrupts within a node, values are matched by index order:

```python
def multiple_interrupt_node(state):
    name = interrupt("What's your name?")
    age = interrupt("What's your age?")
    return {"name": name, "age": age}

# First resume provides name
graph.invoke(Command(resume="Alice"), config)
# Second resume provides age
graph.invoke(Command(resume=30), config)
```

### Subgraph Execution
When interrupts occur in subgraphs, both the subgraph node and parent node restart execution from their beginning when resumed.