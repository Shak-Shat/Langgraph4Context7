# How to stream



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/how-tos/streaming/
- **html**: Home
Guides
How-to Guides
LangGraph
Streaming
How to stream¶

Prerequisites

This guide assumes familiarity with the following:

Streaming
Chat Models

Streaming is crucial for enhancing the responsiveness of applications built on LLMs. By displaying output progressively, even before a complete response is ready, streaming significantly improves user experience (UX), particularly when dealing with the latency of LLMs.

LangGraph is built with first class support for streaming. There are several different ways to stream back outputs from a graph run:

"values": Emit all values in the state after each step.
"updates": Emit only the node names and updates returned by the nodes after each step. If multiple updates are made in the same step (e.g. multiple nodes are run) then those updates are emitted separately.
"custom": Emit custom data from inside nodes using StreamWriter.
"messages": Emit LLM messages token-by-token together with metadata for any LLM invocations inside nodes.
"debug": Emit debug events with as much information as possible for each step.

You can stream outputs from the graph by using graph.stream(..., stream_mode=<stream_mode>) method, e.g.:

Sync
Async
for chunk in graph.stream(inputs, stream_mode="updates"):

    print(chunk)


You can also combine multiple streaming mode by providing a list to stream_mode parameter:

Sync
Async
for chunk in graph.stream(inputs, stream_mode=["updates", "custom"]):

    print(chunk)

Setup¶
%%capture --no-stderr

%pip install --quiet -U langgraph langchain_openai


import getpass

import os





def _set_env(var: str):

    if not os.environ.get(var):

        os.environ[var] = getpass.getpass(f"{var}: ")





_set_env("OPENAI_API_KEY")

OPENAI_API_KEY:  ········


Set up LangSmith for LangGraph development

Sign up for LangSmith to quickly spot issues and improve the performance of your LangGraph projects. LangSmith lets you use trace data to debug, test, and monitor your LLM apps built with LangGraph — read more about how to get started here.

Let's define a simple graph with two nodes:

Define graph¶
from typing import TypedDict

from langgraph.graph import StateGraph, START





class State(TypedDict):

    topic: str

    joke: str





def refine_topic(state: State):

    return {"topic": state["topic"] + " and cats"}





def generate_joke(state: State):

    return {"joke": f"This is a joke about {state['topic']}"}





graph = (

    StateGraph(State)

    .add_node(refine_topic)

    .add_node(generate_joke)

    .add_edge(START, "refine_topic")

    .add_edge("refine_topic", "generate_joke")

    .compile()

)


API Reference: StateGraph | START

Stream all values in the state (stream_mode="values")¶

Use this to stream all values in the state after each step.

for chunk in graph.stream(

    {"topic": "ice cream"},

    stream_mode="values",

):

    print(chunk)

{'topic': 'ice cream'}

{'topic': 'ice cream and cats'}

{'topic': 'ice cream and cats', 'joke': 'This is a joke about ice cream and cats'}


Stream state updates from the nodes (stream_mode="updates")¶

Use this to stream only the state updates returned by the nodes after each step. The streamed outputs include the name of the node as well as the update.

for chunk in graph.stream(

    {"topic": "ice cream"},

    stream_mode="updates",

):

    print(chunk)

{'refine_topic': {'topic': 'ice cream and cats'}}

{'generate_joke': {'joke': 'This is a joke about ice cream and cats'}}


Stream debug events (stream_mode="debug")¶

Use this to stream debug events with as much information as possible for each step. Includes information about tasks that were scheduled to be executed as well as the results of the task executions.

for chunk in graph.stream(

    {"topic": "ice cream"},

    stream_mode="debug",

):

    print(chunk)

{'type': 'task', 'timestamp': '2025-01-28T22:06:34.789803+00:00', 'step': 1, 'payload': {'id': 'eb305d74-3460-9510-d516-beed71a63414', 'name': 'refine_topic', 'input': {'topic': 'ice cream'}, 'triggers': ['start:refine_topic']}}

{'type': 'task_result', 'timestamp': '2025-01-28T22:06:34.790013+00:00', 'step': 1, 'payload': {'id': 'eb305d74-3460-9510-d516-beed71a63414', 'name': 'refine_topic', 'error': None, 'result': [('topic', 'ice cream and cats')], 'interrupts': []}}

{'type': 'task', 'timestamp': '2025-01-28T22:06:34.790165+00:00', 'step': 2, 'payload': {'id': '74355cb8-6284-25e0-579f-430493c1bdab', 'name': 'generate_joke', 'input': {'topic': 'ice cream and cats'}, 'triggers': ['refine_topic']}}

{'type': 'task_result', 'timestamp': '2025-01-28T22:06:34.790337+00:00', 'step': 2, 'payload': {'id': '74355cb8-6284-25e0-579f-430493c1bdab', 'name': 'generate_joke', 'error': None, 'result': [('joke', 'This is a joke about ice cream and cats')], 'interrupts': []}}


Stream LLM tokens (stream_mode="messages")¶

Use this to stream LLM messages token-by-token together with metadata for any LLM invocations inside nodes or tasks. Let's modify the above example to include LLM calls:

from langchain_openai import ChatOpenAI



llm = ChatOpenAI(model="gpt-4o-mini")





def generate_joke(state: State):

    llm_response = llm.invoke(

        [

            {"role": "user", "content": f"Generate a joke about {state['topic']}"}

        ]

    )

    return {"joke": llm_response.content}





graph = (

    StateGraph(State)

    .add_node(refine_topic)

    .add_node(generate_joke)

    .add_edge(START, "refine_topic")

    .add_edge("refine_topic", "generate_joke")

    .compile()

)


API Reference: ChatOpenAI

for message_chunk, metadata in graph.stream(

    {"topic": "ice cream"},

    stream_mode="messages",

):

    if message_chunk.content:

        print(message_chunk.content, end="|", flush=True)

Why| did| the| cat| sit| on| the| ice| cream| cone|?



|Because| it| wanted| to| be| a| "|p|urr|-f|ect|"| scoop|!| 🍦|🐱|


metadata

{'langgraph_step': 2,

 'langgraph_node': 'generate_joke',

 'langgraph_triggers': ['refine_topic'],

 'langgraph_path': ('__pregel_pull', 'generate_joke'),

 'langgraph_checkpoint_ns': 'generate_joke:568879bc-8800-2b0d-a5b5-059526a4bebf',

 'checkpoint_ns': 'generate_joke:568879bc-8800-2b0d-a5b5-059526a4bebf',

 'ls_provider': 'openai',

 'ls_model_name': 'gpt-4o-mini',

 'ls_model_type': 'chat',

 'ls_temperature': 0.7}

Stream custom data (stream_mode="custom")¶

Use this to stream custom data from inside nodes using StreamWriter.

from langgraph.types import StreamWriter





def generate_joke(state: State, writer: StreamWriter):

    writer({"custom_key": "Writing custom data while generating a joke"})

    return {"joke": f"This is a joke about {state['topic']}"}





graph = (

    StateGraph(State)

    .add_node(refine_topic)

    .add_node(generate_joke)

    .add_edge(START, "refine_topic")

    .add_edge("refine_topic", "generate_joke")

    .compile()

)


API Reference: StreamWriter

for chunk in graph.stream(

    {"topic": "ice cream"},

    stream_mode="custom",

):

    print(chunk)

{'custom_key': 'Writing custom data while generating a joke'}


Configure multiple streaming modes¶

Use this to combine multiple streaming modes. The outputs are streamed as tuples (stream_mode, streamed_output).

for stream_mode, chunk in graph.stream(

    {"topic": "ice cream"},

    stream_mode=["updates", "custom"],

):

    print(f"Stream mode: {stream_mode}")

    print(chunk)

    print("\n")

Stream mode: updates

{'refine_topic': {'topic': 'ice cream and cats'}}





Stream mode: custom

{'custom_key': 'Writing custom data while generating a joke'}





Stream mode: updates

{'generate_joke': {'joke': 'This is a joke about ice cream and cats'}}


Comments
