# How to add thread-level persistence (functional API)



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/how-tos/persistence-functional/
- **html**: Home
Guides
How-to Guides
LangGraph
Persistence
How to add thread-level persistence (functional API)¶

Prerequisites

This guide assumes familiarity with the following:

Functional API
Persistence
Memory
Chat Models

Many AI applications need memory to share context across multiple interactions on the same thread (e.g., multiple turns of a conversation). In LangGraph functional API, this kind of memory can be added to any entrypoint() workflow using thread-level persistence.

When creating a LangGraph workflow, you can set it up to persist its results by using a checkpointer:

Create an instance of a checkpointer:

from langgraph.checkpoint.memory import MemorySaver



checkpointer = MemorySaver()       


Pass checkpointer instance to the entrypoint() decorator:

from langgraph.func import entrypoint



@entrypoint(checkpointer=checkpointer)

def workflow(inputs)

    ...


Optionally expose previous parameter in the workflow function signature:

@entrypoint(checkpointer=checkpointer)

def workflow(

    inputs,

    *,

    # you can optionally specify `previous` in the workflow function signature

    # to access the return value from the workflow as of the last execution

    previous

):

    previous = previous or []

    combined_inputs = previous + inputs

    result = do_something(combined_inputs)

    ...


Optionally choose which values will be returned from the workflow and which will be saved by the checkpointer as previous:

@entrypoint(checkpointer=checkpointer)

def workflow(inputs, *, previous):

    ...

    result = do_something(...)

    return entrypoint.final(value=result, save=combine(inputs, result))


This guide shows how you can add thread-level persistence to your workflow.

Note

If you need memory that is shared across multiple conversations or users (cross-thread persistence), check out this how-to guide.

Note

If you need to add thread-level persistence to a StateGraph, check out this how-to guide.

Setup¶

First we need to install the packages required

%%capture --no-stderr

%pip install --quiet -U langgraph langchain_anthropic


Next, we need to set API key for Anthropic (the LLM we will use).

import getpass

import os





def _set_env(var: str):

    if not os.environ.get(var):

        os.environ[var] = getpass.getpass(f"{var}: ")





_set_env("ANTHROPIC_API_KEY")


Set up LangSmith for LangGraph development

Sign up for LangSmith to quickly spot issues and improve the performance of your LangGraph projects. LangSmith lets you use trace data to debug, test, and monitor your LLM apps built with LangGraph — read more about how to get started here.

Example: simple chatbot with short-term memory¶

We will be using a workflow with a single task that calls a chat model.

Let's first define the model we'll be using:

from langchain_anthropic import ChatAnthropic



model = ChatAnthropic(model="claude-3-5-sonnet-latest")


API Reference: ChatAnthropic

Now we can define our task and workflow. To add in persistence, we need to pass in a Checkpointer to the entrypoint() decorator.

from langchain_core.messages import BaseMessage

from langgraph.graph import add_messages

from langgraph.func import entrypoint, task

from langgraph.checkpoint.memory import MemorySaver





@task

def call_model(messages: list[BaseMessage]):

    response = model.invoke(messages)

    return response





checkpointer = MemorySaver()





@entrypoint(checkpointer=checkpointer)

def workflow(inputs: list[BaseMessage], *, previous: list[BaseMessage]):

    if previous:

        inputs = add_messages(previous, inputs)



    response = call_model(inputs).result()

    return entrypoint.final(value=response, save=add_messages(inputs, response))


API Reference: BaseMessage | add_messages | entrypoint | task | MemorySaver

If we try to use this workflow, the context of the conversation will be persisted across interactions:

Note

If you're using LangGraph Cloud or LangGraph Studio, you don't need to pass checkpointer to the entrypoint decorator, since it's done automatically.

We can now interact with the agent and see that it remembers previous messages!

config = {"configurable": {"thread_id": "1"}}

input_message = {"role": "user", "content": "hi! I'm bob"}

for chunk in workflow.stream([input_message], config, stream_mode="values"):

    chunk.pretty_print()

==================================[1m Ai Message [0m==================================



Hi Bob! I'm Claude. Nice to meet you! How are you today?

You can always resume previous threads:

input_message = {"role": "user", "content": "what's my name?"}

for chunk in workflow.stream([input_message], config, stream_mode="values"):

    chunk.pretty_print()

==================================[1m Ai Message [0m==================================



Your name is Bob.

If we want to start a new conversation, we can pass in a different thread_id. Poof! All the memories are gone!

input_message = {"role": "user", "content": "what's my name?"}

for chunk in workflow.stream(

    [input_message],

    {"configurable": {"thread_id": "2"}},

    stream_mode="values",

):

    chunk.pretty_print()

==================================[1m Ai Message [0m==================================



I don't know your name unless you tell me. Each conversation I have starts fresh, so I don't have access to any previous interactions or personal information unless you share it with me.


Streaming tokens

If you would like to stream LLM tokens from your chatbot, you can use stream_mode="messages". Check out this how-to guide to learn more.

Comments
