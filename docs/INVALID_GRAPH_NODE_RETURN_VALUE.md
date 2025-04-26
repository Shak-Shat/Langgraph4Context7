# INVALID_GRAPH_NODE_RETURN_VALUE



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/troubleshooting/errors/INVALID_GRAPH_NODE_RETURN_VALUE/
- **html**: Home
Resources
Troubleshooting
INVALID_GRAPH_NODE_RETURN_VALUE¶

A LangGraph StateGraph received a non-dict return type from a node. Here's an example:

class State(TypedDict):

    some_key: str



def bad_node(state: State):

    # Should return an dict with a value for "some_key", not a list

    return ["whoops"]



builder = StateGraph(State)

builder.add_node(bad_node)

...



graph = builder.compile()


Invoking the above graph will result in an error like this:

graph.invoke({ "some_key": "someval" });

InvalidUpdateError: Expected dict, got ['whoops']

For troubleshooting, visit: https://python.langchain.com/docs/troubleshooting/errors/INVALID_GRAPH_NODE_RETURN_VALUE


Nodes in your graph must return an dict containing one or more keys defined in your state.

Troubleshooting¶

The following may help resolve this error:

If you have complex logic in your node, make sure all code paths return an appropriate dict for your defined state.
Comments
