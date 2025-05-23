# GRAPH_RECURSION_LIMIT



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/troubleshooting/errors/GRAPH_RECURSION_LIMIT/
- **html**: Home
Resources
Troubleshooting
GRAPH_RECURSION_LIMIT¶

Your LangGraph StateGraph reached the maximum number of steps before hitting a stop condition. This is often due to an infinite loop caused by code like the example below:

class State(TypedDict):

    some_key: str



builder = StateGraph(State)

builder.add_node("a", ...)

builder.add_node("b", ...)

builder.add_edge("a", "b")

builder.add_edge("b", "a")

...



graph = builder.compile()


However, complex graphs may hit the default limit naturally.

Troubleshooting¶
If you are not expecting your graph to go through many iterations, you likely have a cycle. Check your logic for infinite loops.
If you have a complex graph, you can pass in a higher recursion_limit value into your config object when invoking your graph like this:
graph.invoke({...}, {"recursion_limit": 100})

Comments
