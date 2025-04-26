# How to define input/output schema for your graph



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/how-tos/input_output_schema/
- **html**: Home
Guides
How-to Guides
LangGraph
State Management
How to define input/output schema for your graph¶

Prerequisites

This guide assumes familiarity with the following:

Multiple Schemas
State Graph

By default, StateGraph operates with a single schema, and all nodes are expected to communicate using that schema. However, it's also possible to define distinct input and output schemas for a graph.

When distinct schemas are specified, an internal schema will still be used for communication between nodes. The input schema ensures that the provided input matches the expected structure, while the output schema filters the internal data to return only the relevant information according to the defined output schema.

In this example, we'll see how to define distinct input and output schema.

Setup¶

First, let's install the required packages

%%capture --no-stderr

%pip install -U langgraph


Set up LangSmith for LangGraph development

Sign up for LangSmith to quickly spot issues and improve the performance of your LangGraph projects. LangSmith lets you use trace data to debug, test, and monitor your LLM apps built with LangGraph — read more about how to get started here.

Define and use the graph¶
from langgraph.graph import StateGraph, START, END

from typing_extensions import TypedDict





# Define the schema for the input

class InputState(TypedDict):

    question: str





# Define the schema for the output

class OutputState(TypedDict):

    answer: str





# Define the overall schema, combining both input and output

class OverallState(InputState, OutputState):

    pass





# Define the node that processes the input and generates an answer

def answer_node(state: InputState):

    # Example answer and an extra key

    return {"answer": "bye", "question": state["question"]}





# Build the graph with input and output schemas specified

builder = StateGraph(OverallState, input=InputState, output=OutputState)

builder.add_node(answer_node)  # Add the answer node

builder.add_edge(START, "answer_node")  # Define the starting edge

builder.add_edge("answer_node", END)  # Define the ending edge

graph = builder.compile()  # Compile the graph



# Invoke the graph with an input and print the result

print(graph.invoke({"question": "hi"}))


API Reference: StateGraph | START | END

{'answer': 'bye'}

Notice that the output of invoke only includes the output schema.

Comments
