# How to add node retry policies



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/how-tos/node-retries/
- **html**: Home
Guides
How-to Guides
LangGraph
Controllability
How to add node retry policies¶

Prerequisites

This guide assumes familiarity with the following:

LangGraph Glossary

There are many use cases where you may wish for your node to have a custom retry policy, for example if you are calling an API, querying a database, or calling an LLM, etc.

Setup¶

First, let's install the required packages and set our API keys

%%capture --no-stderr

%pip install -U langgraph langchain_anthropic langchain_community

import getpass

import os





def _set_env(var: str):

    if not os.environ.get(var):

        os.environ[var] = getpass.getpass(f"{var}: ")





_set_env("ANTHROPIC_API_KEY")


Set up LangSmith for LangGraph development

Sign up for LangSmith to quickly spot issues and improve the performance of your LangGraph projects. LangSmith lets you use trace data to debug, test, and monitor your LLM apps built with LangGraph — read more about how to get started here.

In order to configure the retry policy, you have to pass the retry parameter to the add_node. The retry parameter takes in a RetryPolicy named tuple object. Below we instantiate a RetryPolicy object with the default parameters:

from langgraph.pregel import RetryPolicy



RetryPolicy()

RetryPolicy(initial_interval=0.5, backoff_factor=2.0, max_interval=128.0, max_attempts=3, jitter=True, retry_on=<function default_retry_on at 0x78b964b89940>)


By default, the retry_on parameter uses the default_retry_on function, which retries on any exception except for the following:

ValueError
TypeError
ArithmeticError
ImportError
LookupError
NameError
SyntaxError
RuntimeError
ReferenceError
StopIteration
StopAsyncIteration
OSError

In addition, for exceptions from popular http request libraries such as requests and httpx it only retries on 5xx status codes.

Passing a retry policy to a node¶

Lastly, we can pass RetryPolicy objects when we call the add_node function. In the example below we pass two different retry policies to each of our nodes:

import operator

import sqlite3

from typing import Annotated, Sequence

from typing_extensions import TypedDict



from langchain_anthropic import ChatAnthropic

from langchain_core.messages import BaseMessage



from langgraph.graph import END, StateGraph, START

from langchain_community.utilities import SQLDatabase

from langchain_core.messages import AIMessage



db = SQLDatabase.from_uri("sqlite:///:memory:")



model = ChatAnthropic(model_name="claude-2.1")





class AgentState(TypedDict):

    messages: Annotated[Sequence[BaseMessage], operator.add]





def query_database(state):

    query_result = db.run("SELECT * FROM Artist LIMIT 10;")

    return {"messages": [AIMessage(content=query_result)]}





def call_model(state):

    response = model.invoke(state["messages"])

    return {"messages": [response]}





# Define a new graph

builder = StateGraph(AgentState)

builder.add_node(

    "query_database",

    query_database,

    retry=RetryPolicy(retry_on=sqlite3.OperationalError),

)

builder.add_node("model", call_model, retry=RetryPolicy(max_attempts=5))

builder.add_edge(START, "model")

builder.add_edge("model", "query_database")

builder.add_edge("query_database", END)



graph = builder.compile()


API Reference: ChatAnthropic | BaseMessage | END | StateGraph | START | SQLDatabase | AIMessage

Comments
