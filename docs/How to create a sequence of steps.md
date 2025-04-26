# How to create a sequence of steps



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/how-tos/sequence/
- **html**: Home
Guides
How-to Guides
LangGraph
Graph API Basics
How to create a sequence of steps¶

Prerequisites

This guide assumes familiarity with the following:

How to define and update graph state

This guide demonstrates how to construct a simple sequence of steps. We will demonstrate:

How to build a sequential graph
Built-in short-hand for constructing similar graphs.
Summary¶

To add a sequence of nodes, we use the .add_node and .add_edge methods of our graph:

from langgraph.graph import START, StateGraph



graph_builder = StateGraph(State)



# Add nodes

graph_builder.add_node(step_1)

graph_builder.add_node(step_2)

graph_builder.add_node(step_3)



# Add edges

graph_builder.add_edge(START, "step_1")

graph_builder.add_edge("step_1", "step_2")

graph_builder.add_edge("step_2", "step_3")


API Reference: START | StateGraph

We can also use the built-in shorthand .add_sequence:

graph_builder = StateGraph(State).add_sequence([step_1, step_2, step_3])

graph_builder.add_edge(START, "step_1")


Why split application steps into a sequence with LangGraph?
Setup¶

First, let's install langgraph:

%%capture --no-stderr

%pip install -U langgraph


Set up LangSmith for better debugging

Sign up for LangSmith to quickly spot issues and improve the performance of your LangGraph projects. LangSmith lets you use trace data to debug, test, and monitor your LLM aps built with LangGraph — read more about how to get started in the docs.

Build the graph¶

Let's demonstrate a simple usage example. We will create a sequence of three steps:

Populate a value in a key of the state
Update the same value
Populate a different value
Define state¶

Let's first define our state. This governs the schema of the graph, and can also specify how to apply updates. See this guide for more detail.

In our case, we will just keep track of two values:

from typing_extensions import TypedDict





class State(TypedDict):

    value_1: str

    value_2: int

Define nodes¶

Our nodes are just Python functions that read our graph's state and make updates to it. The first argument to this function will always be the state:

def step_1(state: State):

    return {"value_1": "a"}





def step_2(state: State):

    current_value_1 = state["value_1"]

    return {"value_1": f"{current_value_1} b"}





def step_3(state: State):

    return {"value_2": 10}


Note

Note that when issuing updates to the state, each node can just specify the value of the key it wishes to update.

By default, this will overwrite the value of the corresponding key. You can also use reducers to control how updates are processed— for example, you can append successive updates to a key instead. See this guide for more detail.

Define graph¶

We use StateGraph to define a graph that operates on this state.

We will then use add_node and add_edge to populate our graph and define its control flow.

from langgraph.graph import START, StateGraph



graph_builder = StateGraph(State)



# Add nodes

graph_builder.add_node(step_1)

graph_builder.add_node(step_2)

graph_builder.add_node(step_3)



# Add edges

graph_builder.add_edge(START, "step_1")

graph_builder.add_edge("step_1", "step_2")

graph_builder.add_edge("step_2", "step_3")


API Reference: START | StateGraph

Specifying custom names

You can specify custom names for nodes using .add_node:

graph_builder.add_node("my_node", step_1)


Note that:

.add_edge takes the names of nodes, which for functions defaults to node.__name__.
We must specify the entry point of the graph. For this we add an edge with the START node.
The graph halts when there are no more nodes to execute.

We next compile our graph. This provides a few basic checks on the structure of the graph (e.g., identifying orphaned nodes). If we were adding persistence to our application via a checkpointer, it would also be passed in here.

graph = graph_builder.compile()


LangGraph provides built-in utilities for visualizing your graph. Let's inspect our sequence. See this guide for detail on visualization.

from IPython.display import Image, display



display(Image(graph.get_graph().draw_mermaid_png()))


Usage¶

Let's proceed with a simple invocation:

graph.invoke({"value_1": "c"})

{'value_1': 'a b', 'value_2': 10}


Note that:

We kicked off invocation by providing a value for a single state key. We must always provide a value for at least one key.
The value we passed in was overwritten by the first node.
The second node updated the value.
The third node populated a different value.
Built-in shorthand¶

Prerequisites

.add_sequence requires langgraph>=0.2.46

LangGraph includes a built-in shorthand .add_sequence for convenience:

graph_builder = StateGraph(State).add_sequence([step_1, step_2, step_3])

graph_builder.add_edge(START, "step_1")



graph = graph_builder.compile()



graph.invoke({"value_1": "c"})

{'value_1': 'a b', 'value_2': 10}

Comments
