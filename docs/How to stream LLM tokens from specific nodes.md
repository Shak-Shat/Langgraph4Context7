# How to stream LLM tokens from specific nodes



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/how-tos/streaming-specific-nodes/
- **html**: Home
Guides
How-to Guides
LangGraph
Streaming
How to stream LLM tokens from specific nodes¶

Prerequisites

This guide assumes familiarity with the following:

Streaming
Chat Models

A common use case when streaming LLM tokens is to only stream them from specific nodes. To do so, you can use stream_mode="messages" and filter the outputs by the langgraph_node field in the streamed metadata:

from langgraph.graph import StateGraph

from langchain_openai import ChatOpenAI



model = ChatOpenAI()



def node_a(state: State):

    model.invoke(...)

    ...



def node_b(state: State):

    model.invoke(...)

    ...



graph = (

    StateGraph(State)

    .add_node(node_a)

    .add_node(node_b)

    ...

    .compile()



for msg, metadata in graph.stream(

    inputs,

    stream_mode="messages"

):

    # stream from 'node_a'

    if metadata["langgraph_node"] == "node_a":

        print(msg)


Streaming from a specific LLM invocation

If you need to instead filter streamed LLM tokens to a specific LLM invocation, check out this guide

Setup¶

First we need to install the packages required

%%capture --no-stderr

%pip install --quiet -U langgraph langchain_openai

import getpass

import os





def _set_env(var: str):

    if not os.environ.get(var):

        os.environ[var] = getpass.getpass(f"{var}: ")





_set_env("OPENAI_API_KEY")

Example¶
from typing import TypedDict

from langgraph.graph import START, StateGraph, MessagesState

from langchain_openai import ChatOpenAI



model = ChatOpenAI(model="gpt-4o-mini")





class State(TypedDict):

    topic: str

    joke: str

    poem: str





def write_joke(state: State):

    topic = state["topic"]

    joke_response = model.invoke(

        [{"role": "user", "content": f"Write a joke about {topic}"}]

    )

    return {"joke": joke_response.content}





def write_poem(state: State):

    topic = state["topic"]

    poem_response = model.invoke(

        [{"role": "user", "content": f"Write a short poem about {topic}"}]

    )

    return {"poem": poem_response.content}





graph = (

    StateGraph(State)

    .add_node(write_joke)

    .add_node(write_poem)

    # write both the joke and the poem concurrently

    .add_edge(START, "write_joke")

    .add_edge(START, "write_poem")

    .compile()

)


API Reference: START | StateGraph | ChatOpenAI

for msg, metadata in graph.stream(

    {"topic": "cats"},

    stream_mode="messages",

):

    if msg.content and metadata["langgraph_node"] == "write_poem":

        print(msg.content, end="|", flush=True)

In| shadows| soft|,| they| quietly| creep|,|  

|Wh|isk|ered| wonders|,| in| dreams| they| leap|.|  

|With| eyes| like| lantern|s|,| bright| and| wide|,|  

|Myst|eries| linger| where| they| reside|.|  



|P|aws| that| pat|ter| on| silent| floors|,|  

|Cur|led| in| sun|be|ams|,| they| seek| out| more|.|  

|A| flick| of| a| tail|,| a| leap|,| a| p|ounce|,|  

|In| their| playful| world|,| we| can't| help| but| bounce|.|  



|Guard|ians| of| secrets|,| with| gentle| grace|,|  

|Each| little| me|ow|,| a| warm| embrace|.|  

|Oh|,| the| joy| that| they| bring|,| so| pure| and| true|,|  

|In| the| heart| of| a| cat|,| there's| magic| anew|.|  |


Comments
