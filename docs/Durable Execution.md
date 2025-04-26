# Durable Execution



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/concepts/durable_execution/
- **html**: Home
Guides
Concepts
LangGraph
Durable Execution¶

Durable execution is a technique in which a process or workflow saves its progress at key points, allowing it to pause and later resume exactly where it left off. This is particularly useful in scenarios that require human-in-the-loop, where users can inspect, validate, or modify the process before continuing, and in long-running tasks that might encounter interruptions or errors (e.g., calls to an LLM timing out). By preserving completed work, durable execution enables a process to resume without reprocessing previous steps -- even after a significant delay (e.g., a week later).

LangGraph's built-in persistence layer provides durable execution for workflows, ensuring that the state of each execution step is saved to a durable store. This capability guarantees that if a workflow is interrupted -- whether by a system failure or for human-in-the-loop interactions -- it can be resumed from its last recorded state.

Tip

If you are using LangGraph with a checkpointer, you already have durable execution enabled. You can pause and resume workflows at any point, even after interruptions or failures. To make the most of durable execution, ensure that your workflow is designed to be deterministic and idempotent and wrap any side effects or non-deterministic operations inside tasks. You can use tasks from both the StateGraph (Graph API) and the Functional API.

Requirements¶

To leverage durable execution in LangGraph, you need to:

Enable persistence in your workflow by specifying a checkpointer that will save workflow progress.
Specify a thread identifier when executing a workflow. This will track the execution history for a particular instance of the workflow.
Wrap any non-deterministic operations (e.g., random number generation) or operations with side effects (e.g., file writes, API calls) inside tasks to ensure that when a workflow is resumed, these operations are not repeated for the particular run, and instead their results are retrieved from the persistence layer. For more information, see Determinism and Consistent Replay.
Determinism and Consistent Replay¶

When you resume a workflow run, the code does NOT resume from the same line of code where execution stopped; instead, it will identify an appropriate starting point from which to pick up where it left off. This means that the workflow will replay all steps from the starting point until it reaches the point where it was stopped.

As a result, when you are writing a workflow for durable execution, you must wrap any non-deterministic operations (e.g., random number generation) and any operations with side effects (e.g., file writes, API calls) inside tasks or nodes.

To ensure that your workflow is deterministic and can be consistently replayed, follow these guidelines:

Avoid Repeating Work: If a node contains multiple operations with side effects (e.g., logging, file writes, or network calls), wrap each operation in a separate task. This ensures that when the workflow is resumed, the operations are not repeated, and their results are retrieved from the persistence layer.
Encapsulate Non-Deterministic Operations: Wrap any code that might yield non-deterministic results (e.g., random number generation) inside tasks or nodes. This ensures that, upon resumption, the workflow follows the exact recorded sequence of steps with the same outcomes.
Use Idempotent Operations: When possible ensure that side effects (e.g., API calls, file writes) are idempotent. This means that if an operation is retried after a failure in the workflow, it will have the same effect as the first time it was executed. This is particularly important for operations that result in data writes. In the event that a task starts but fails to complete successfully, the workflow's resumption will re-run the task, relying on recorded outcomes to maintain consistency. Use idempotency keys or verify existing results to avoid unintended duplication, ensuring a smooth and predictable workflow execution.

For some examples of pitfalls to avoid, see the Common Pitfalls section in the functional API, which shows how to structure your code using tasks to avoid these issues. The same principles apply to the StateGraph (Graph API).

Using tasks in nodes¶

If a node contains multiple operations, you may find it easier to convert each operation into a task rather than refactor the operations into individual nodes.

Original
With task
from typing import NotRequired

from typing_extensions import TypedDict

import uuid



from langgraph.checkpoint.memory import MemorySaver

from langgraph.graph import StateGraph, START, END

import requests



# Define a TypedDict to represent the state

class State(TypedDict):

    url: str

    result: NotRequired[str]



def call_api(state: State):

    """Example node that makes an API request."""

    result = requests.get(state['url']).text[:100]  # Side-effect

    return {

        "result": result

    }



# Create a StateGraph builder and add a node for the call_api function

builder = StateGraph(State)

builder.add_node("call_api", call_api)



# Connect the start and end nodes to the call_api node

builder.add_edge(START, "call_api")

builder.add_edge("call_api", END)



# Specify a checkpointer

checkpointer = MemorySaver()



# Compile the graph with the checkpointer

graph = builder.compile(checkpointer=checkpointer)



# Define a config with a thread ID.

thread_id = uuid.uuid4()

config = {"configurable": {"thread_id": thread_id}}



# Invoke the graph

graph.invoke({"url": "https://www.example.com"}, config)

Resuming Workflows¶

Once you have enabled durable execution in your workflow, you can resume execution for the following scenarios:

Pausing and Resuming Workflows: Use the interrupt function to pause a workflow at specific points and the Command primitive to resume it with updated state. See Human-in-the-Loop for more details.
Recovering from Failures: Automatically resume workflows from the last successful checkpoint after an exception (e.g., LLM provider outage). This involves executing the workflow with the same thread identifier by providing it with a None as the input value (see this example with the functional API).
Starting Points for Resuming Workflows¶
If you're using a StateGraph (Graph API), the starting point is the beginning of the node where execution stopped.
If you're making a subgraph call inside a node, the starting point will be the parent node that called the subgraph that was halted. Inside the subgraph, the starting point will be the specific node where execution stopped.
If you're using the Functional API, the starting point is the beginning of the entrypoint where execution stopped.
Comments
