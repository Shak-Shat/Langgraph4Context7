# How to interact with the deployment using RemoteGraph



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/how-tos/use-remote-graph/
- **html**: Home
Guides
Concepts
LangGraph Platform
Components
How to interact with the deployment using RemoteGraph¶

Prerequisites

LangGraph Platform
LangGraph Server

RemoteGraph is an interface that allows you to interact with your LangGraph Platform deployment as if it were a regular, locally-defined LangGraph graph (e.g. a CompiledGraph). This guide shows you how you can initialize a RemoteGraph and interact with it.

Initializing the graph¶

When initializing a RemoteGraph, you must always specify:

name: the name of the graph you want to interact with. This is the same graph name you use in langgraph.json configuration file for your deployment.
api_key: a valid LangSmith API key. Can be set as an environment variable (LANGSMITH_API_KEY) or passed directly via the api_key argument. The API key could also be provided via the client / sync_client arguments, if LangGraphClient / SyncLangGraphClient were initialized with api_key argument.

Additionally, you have to provide one of the following:

url: URL of the deployment you want to interact with. If you pass url argument, both sync and async clients will be created using the provided URL, headers (if provided) and default configuration values (e.g. timeout, etc).
client: a LangGraphClient instance for interacting with the deployment asynchronously (e.g. using .astream(), .ainvoke(), .aget_state(), .aupdate_state(), etc.)
sync_client: a SyncLangGraphClient instance for interacting with the deployment synchronously (e.g. using .stream(), .invoke(), .get_state(), .update_state(), etc.)

Note

If you pass both client or sync_client as well as url argument, they will take precedence over the url argument. If none of the client / sync_client / url arguments are provided, RemoteGraph will raise a ValueError at runtime.

Using URL¶
Python
JavaScript
from langgraph.pregel.remote import RemoteGraph



url = <DEPLOYMENT_URL>

graph_name = "agent"

remote_graph = RemoteGraph(graph_name, url=url)

Using clients¶
Python
JavaScript
from langgraph_sdk import get_client, get_sync_client

from langgraph.pregel.remote import RemoteGraph



url = <DEPLOYMENT_URL>

graph_name = "agent"

client = get_client(url=url)

sync_client = get_sync_client(url=url)

remote_graph = RemoteGraph(graph_name, client=client, sync_client=sync_client)

Invoking the graph¶

Since RemoteGraph is a Runnable that implements the same methods as CompiledGraph, you can interact with it the same way you normally would with a compiled graph, i.e. by calling .invoke(), .stream(), .get_state(), .update_state(), etc (as well as their async counterparts).

Asynchronously¶

Note

To use the graph asynchronously, you must provide either the url or client when initializing the RemoteGraph.

Python
JavaScript
# invoke the graph

result = await remote_graph.ainvoke({

    "messages": [{"role": "user", "content": "what's the weather in sf"}]

})



# stream outputs from the graph

async for chunk in remote_graph.astream({

    "messages": [{"role": "user", "content": "what's the weather in la"}]

}):

    print(chunk)

Synchronously¶

Note

To use the graph synchronously, you must provide either the url or sync_client when initializing the RemoteGraph.

Python
# invoke the graph

result = remote_graph.invoke({

    "messages": [{"role": "user", "content": "what's the weather in sf"}]

})



# stream outputs from the graph

for chunk in remote_graph.stream({

    "messages": [{"role": "user", "content": "what's the weather in la"}]

}):

    print(chunk)

Thread-level persistence¶

By default, the graph runs (i.e. .invoke() or .stream() invocations) are stateless - the checkpoints and the final state of the graph are not persisted. If you would like to persist the outputs of the graph run (for example, to enable human-in-the-loop features), you can create a thread and provide the thread ID via the config argument, same as you would with a regular compiled graph:

Python
JavaScript
from langgraph_sdk import get_sync_client

url = <DEPLOYMENT_URL>

graph_name = "agent"

sync_client = get_sync_client(url=url)

remote_graph = RemoteGraph(graph_name, url=url)



# create a thread (or use an existing thread instead)

thread = sync_client.threads.create()



# invoke the graph with the thread config

config = {"configurable": {"thread_id": thread["thread_id"]}}

result = remote_graph.invoke({

    "messages": [{"role": "user", "content": "what's the weather in sf"}]

}, config=config)



# verify that the state was persisted to the thread

thread_state = remote_graph.get_state(config)

print(thread_state)

Using as a subgraph¶

Note

If you need to use a checkpointer with a graph that has a RemoteGraph subgraph node, make sure to use UUIDs as thread IDs.

Since the RemoteGraph behaves the same way as a regular CompiledGraph, it can be also used as a subgraph in another graph. For example:

Python
JavaScript
from langgraph_sdk import get_sync_client

from langgraph.graph import StateGraph, MessagesState, START

from typing import TypedDict



url = <DEPLOYMENT_URL>

graph_name = "agent"

remote_graph = RemoteGraph(graph_name, url=url)



# define parent graph

builder = StateGraph(MessagesState)

# add remote graph directly as a node

builder.add_node("child", remote_graph)

builder.add_edge(START, "child")

graph = builder.compile()



# invoke the parent graph

result = graph.invoke({

    "messages": [{"role": "user", "content": "what's the weather in sf"}]

})

print(result)



# stream outputs from both the parent graph and subgraph

for chunk in graph.stream({

    "messages": [{"role": "user", "content": "what's the weather in sf"}]

}, subgraphs=True):

    print(chunk)

Comments
