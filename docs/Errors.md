# Errors



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/reference/errors/
- **html**: API reference
Library
Errors¶
 GraphRecursionError ¶

Bases: RecursionError

Raised when the graph has exhausted the maximum number of steps.

This prevents infinite loops. To increase the maximum number of steps, run your graph with a config specifying a higher recursion_limit.

Troubleshooting Guides:

GRAPH_RECURSION_LIMIT

Examples:

graph = builder.compile()
graph.invoke(
    {"messages": [("user", "Hello, world!")]},
    # The config is the second positional argument
    {"recursion_limit": 1000},
)

 InvalidUpdateError ¶

Bases: Exception

Raised when attempting to update a channel with an invalid set of updates.

Troubleshooting Guides:

INVALID_CONCURRENT_GRAPH_UPDATE
INVALID_GRAPH_NODE_RETURN_VALUE
 GraphInterrupt ¶

Bases: GraphBubbleUp

Raised when a subgraph is interrupted, suppressed by the root graph. Never raised directly, or surfaced to the user.

 NodeInterrupt ¶

Bases: GraphInterrupt

Raised by a node to interrupt execution.

 GraphDelegate ¶

Bases: GraphBubbleUp

Raised when a graph is delegated (for distributed mode).

 EmptyInputError ¶

Bases: Exception

Raised when graph receives an empty input.

 TaskNotFound ¶

Bases: Exception

Raised when the executor is unable to find a task (for distributed mode).

 CheckpointNotLatest ¶

Bases: Exception

Raised when the checkpoint is not the latest version (for distributed mode).

Comments
