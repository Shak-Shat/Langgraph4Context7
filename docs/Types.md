# LangGraph Type System

## Overview
This guide documents the core types used in LangGraph for controlling execution flow, managing state, and implementing advanced features like streaming, interruption, and caching.

## Key Concepts
- Control flow primitives (Command, Send)
- Streaming modes and output management
- Interruption for human-in-the-loop workflows
- State snapshots and execution management
- Retry and caching policies

## Prerequisites
- Python typing knowledge
- Understanding of graph execution flow
- Familiarity with LangGraph state management

## Implementation

### Special Constants

```python
# Special value to indicate graph should interrupt on all nodes
All = Literal['*']

# Stream modes control how the stream method emits outputs
StreamMode = Literal['values', 'updates', 'debug', 'messages', 'custom']

# Callable for writing custom stream output
StreamWriter = Callable[[Any], None]
```

### Retry and Cache Policies

```python
# Configuration for retrying nodes
class RetryPolicy(NamedTuple):
    initial_interval: float = 0.5  # Seconds before first retry
    backoff_factor: float = 2.0    # Multiplier for interval increase
    max_interval: float = 128.0    # Maximum seconds between retries
    max_attempts: int = 3          # Max attempts before giving up
    jitter: bool = True            # Add random jitter between retries
    retry_on: Union[Type[Exception], Sequence[Type[Exception]], 
              Callable[[Exception], bool]] = default_retry_on

# Configuration for caching node results
class CachePolicy(NamedTuple):
    # Cache configuration parameters
    pass
```

### State Management Types

```python
# Snapshot of graph state at the beginning of a step
class StateSnapshot(NamedTuple):
    values: Union[dict[str, Any], Any]  # Current values of channels
    next: tuple[str, ...]               # Nodes to execute in each task
    config: RunnableConfig              # Config used for this snapshot
    metadata: Optional[CheckpointMetadata]  # Associated metadata
    created_at: Optional[str]           # Timestamp of creation
    parent_config: Optional[RunnableConfig]  # Config for parent snapshot
    tasks: tuple[PregelTask, ...]       # Tasks to execute
```

### Control Flow Primitives

#### Send Class
```python
class Send:
    """A message or packet to send to a specific node in the graph.
    
    Used within a StateGraph's conditional edges to dynamically invoke 
    a node with a custom state at the next step.
    """
    
    def __init__(self, node: str, arg: Any) -> None:
        """Initialize a new Send instance.
        
        Parameters:
            node (str): The target node name to send the message to.
            arg (Any): The state or message to send to the target node.
        """
        self.node = node
        self.arg = arg
```

#### Command Class
```python
@dataclass
class Command(Generic[N], ToolOutputMixin):
    """Commands to update graph state and control flow.
    
    Parameters:
        graph (Optional[str], default: None): Graph to send the command to.
            None: current graph (default)
            Command.PARENT: closest parent graph
        update (Optional[Any], default: None): Update to apply to graph state.
        resume (Optional[Union[Any, dict[str, Any]]], default: None): 
            Value to resume execution with after interrupt.
        goto (Union[Send, Sequence[Union[Send, str]], str], default: ()):
            - Name of node to navigate to next
            - Sequence of node names to navigate to
            - Send object with input
            - Sequence of Send objects
    """
    # Implementation details omitted for brevity
```

### Human-in-the-Loop Functionality

#### Interrupt Function
```python
def interrupt(value: Any) -> Any:
    """Interrupt graph execution with a resumable exception.
    
    Enables human-in-the-loop workflows by pausing execution and 
    surfacing a value to the client. The value can communicate context
    or request input required to resume execution.
    
    Parameters:
        value (Any): Value to surface to the client on interruption.
        
    Returns:
        Any: On subsequent invocations within the same node, returns
            the value provided during resumption.
            
    Raises:
        GraphInterrupt: On first invocation, halts execution.
    """
    # Implementation details omitted for brevity
```

## Usage Example

### Human-in-the-Loop Workflow with Interrupt
```python
import uuid
from typing import Optional
from typing_extensions import TypedDict

from langgraph.checkpoint.memory import MemorySaver
from langgraph.constants import START
from langgraph.graph import StateGraph
from langgraph.types import interrupt, Command

class State(TypedDict):
    """The graph state."""
    foo: str
    human_value: Optional[str]
    """Human value will be updated using an interrupt."""

def node(state: State):
    answer = interrupt(
        # This value will be sent to the client
        # as part of the interrupt information.
        "what is your age?"
    )
    print(f"> Received an input from the interrupt: {answer}")
    return {"human_value": answer}

builder = StateGraph(State)
builder.add_node("node", node)
builder.add_edge(START, "node")

# A checkpointer must be enabled for interrupts to work!
checkpointer = MemorySaver()
graph = builder.compile(checkpointer=checkpointer)

config = {
    "configurable": {
        "thread_id": uuid.uuid4(),
    }
}

# Run graph until interrupt
for chunk in graph.stream({"foo": "abc"}, config):
    print(chunk)
# {'__interrupt__': (Interrupt(value='what is your age?', resumable=True, ns=['node:62e598fa-8653-9d6d-2046-a70203020e37'], when='during'),)}

# Resume execution with provided value
for chunk in graph.stream(Command(resume="some input from a human!!!"), config):
    print(chunk)
# Received an input from the interrupt: some input from a human!!!
# {'node': {'human_value': 'some input from a human!!!'}}
```

### Map-Reduce Pattern with Send
```python
from typing import Annotated
import operator
from typing_extensions import TypedDict
from langgraph.types import Send
from langgraph.graph import END, START, StateGraph

class OverallState(TypedDict):
    subjects: list[str]
    jokes: Annotated[list[str], operator.add]

def continue_to_jokes(state: OverallState):
    return [Send("generate_joke", {"subject": s}) for s in state['subjects']]

builder = StateGraph(OverallState)
builder.add_node("generate_joke", lambda state: {"jokes": [f"Joke about {state['subject']}"]})
builder.add_conditional_edges(START, continue_to_jokes)
builder.add_edge("generate_joke", END)
graph = builder.compile()

# Invoking with two subjects results in a joke for each
result = graph.invoke({"subjects": ["cats", "dogs"]})
# {'subjects': ['cats', 'dogs'], 'jokes': ['Joke about cats', 'Joke about dogs']}
```

## Benefits
- Fine-grained control over graph execution flow
- Support for complex multi-agent interactions
- Human-in-the-loop capabilities
- Robust error handling and retry mechanisms
- Flexible state management across execution spans

## Considerations
- Requires a checkpointer for interrupt functionality
- Remember proper order of resume values for multiple interrupts
- Command navigation paths must reference valid graph nodes
- Type annotations are critical for properly typed workflows
