# RemoteGraph



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/reference/remote_graph/
- **html**: API reference
LangGraph Platform
RemoteGraph¶
 RemoteGraph ¶

Bases: PregelProtocol

The RemoteGraph class is a client implementation for calling remote APIs that implement the LangGraph Server API specification.

For example, the RemoteGraph class can be used to call APIs from deployments on LangGraph Cloud.

RemoteGraph behaves the same way as a Graph and can be used directly as a node in another Graph.

 __init__(name: str, /, *, url: Optional[str] = None, api_key: Optional[str] = None, headers: Optional[dict[str, str]] = None, client: Optional[LangGraphClient] = None, sync_client: Optional[SyncLangGraphClient] = None, config: Optional[RunnableConfig] = None) ¶

Specify url, api_key, and/or headers to create default sync and async clients.

If client or sync_client are provided, they will be used instead of the default clients. See LangGraphClient and SyncLangGraphClient for details on the default clients. At least one of url, client, or sync_client must be provided.

Parameters:

name (str) – 

The name of the graph.

url (Optional[str], default: None ) – 

The URL of the remote API.

api_key (Optional[str], default: None ) – 

The API key to use for authentication. If not provided, it will be read from the environment (LANGGRAPH_API_KEY, LANGSMITH_API_KEY, or LANGCHAIN_API_KEY).

headers (Optional[dict[str, str]], default: None ) – 

Additional headers to include in the requests.

client (Optional[LangGraphClient], default: None ) – 

A LangGraphClient instance to use instead of creating a default client.

sync_client (Optional[SyncLangGraphClient], default: None ) – 

A SyncLangGraphClient instance to use instead of creating a default client.

config (Optional[RunnableConfig], default: None ) – 

An optional RunnableConfig instance with additional configuration.

 get_graph(config: Optional[RunnableConfig] = None, *, xray: Union[int, bool] = False) -> DrawableGraph ¶

Get graph by graph name.

This method calls GET /assistants/{assistant_id}/graph.

Parameters:

config (Optional[RunnableConfig], default: None ) – 

This parameter is not used.

xray (Union[int, bool], default: False ) – 

Include graph representation of subgraphs. If an integer value is provided, only subgraphs with a depth less than or equal to the value will be included.

Returns:

Graph – 

The graph information for the assistant in JSON format.

 aget_graph(config: Optional[RunnableConfig] = None, *, xray: Union[int, bool] = False) -> DrawableGraph async ¶

Get graph by graph name.

This method calls GET /assistants/{assistant_id}/graph.

Parameters:

config (Optional[RunnableConfig], default: None ) – 

This parameter is not used.

xray (Union[int, bool], default: False ) – 

Include graph representation of subgraphs. If an integer value is provided, only subgraphs with a depth less than or equal to the value will be included.

Returns:

Graph – 

The graph information for the assistant in JSON format.

 get_state(config: RunnableConfig, *, subgraphs: bool = False) -> StateSnapshot ¶

Get the state of a thread.

This method calls POST /threads/{thread_id}/state/checkpoint if a checkpoint is specified in the config or GET /threads/{thread_id}/state if no checkpoint is specified.

Parameters:

config (RunnableConfig) – 

A RunnableConfig that includes thread_id in the configurable field.

subgraphs (bool, default: False ) – 

Include subgraphs in the state.

Returns:

StateSnapshot – 

The latest state of the thread.

 aget_state(config: RunnableConfig, *, subgraphs: bool = False) -> StateSnapshot async ¶

Get the state of a thread.

This method calls POST /threads/{thread_id}/state/checkpoint if a checkpoint is specified in the config or GET /threads/{thread_id}/state if no checkpoint is specified.

Parameters:

config (RunnableConfig) – 

A RunnableConfig that includes thread_id in the configurable field.

subgraphs (bool, default: False ) – 

Include subgraphs in the state.

Returns:

StateSnapshot – 

The latest state of the thread.

 get_state_history(config: RunnableConfig, *, filter: Optional[dict[str, Any]] = None, before: Optional[RunnableConfig] = None, limit: Optional[int] = None) -> Iterator[StateSnapshot] ¶

Get the state history of a thread.

This method calls POST /threads/{thread_id}/history.

Parameters:

config (RunnableConfig) – 

A RunnableConfig that includes thread_id in the configurable field.

filter (Optional[dict[str, Any]], default: None ) – 

Metadata to filter on.

before (Optional[RunnableConfig], default: None ) – 

A RunnableConfig that includes checkpoint metadata.

limit (Optional[int], default: None ) – 

Max number of states to return.

Returns:

Iterator[StateSnapshot] – 

States of the thread.

 aget_state_history(config: RunnableConfig, *, filter: Optional[dict[str, Any]] = None, before: Optional[RunnableConfig] = None, limit: Optional[int] = None) -> AsyncIterator[StateSnapshot] async ¶

Get the state history of a thread.

This method calls POST /threads/{thread_id}/history.

Parameters:

config (RunnableConfig) – 

A RunnableConfig that includes thread_id in the configurable field.

filter (Optional[dict[str, Any]], default: None ) – 

Metadata to filter on.

before (Optional[RunnableConfig], default: None ) – 

A RunnableConfig that includes checkpoint metadata.

limit (Optional[int], default: None ) – 

Max number of states to return.

Returns:

AsyncIterator[StateSnapshot] – 

States of the thread.

 update_state(config: RunnableConfig, values: Optional[Union[dict[str, Any], Any]], as_node: Optional[str] = None) -> RunnableConfig ¶

Update the state of a thread.

This method calls POST /threads/{thread_id}/state.

Parameters:

config (RunnableConfig) – 

A RunnableConfig that includes thread_id in the configurable field.

values (Optional[Union[dict[str, Any], Any]]) – 

Values to update to the state.

as_node (Optional[str], default: None ) – 

Update the state as if this node had just executed.

Returns:

RunnableConfig – 

RunnableConfig for the updated thread.

 aupdate_state(config: RunnableConfig, values: Optional[Union[dict[str, Any], Any]], as_node: Optional[str] = None) -> RunnableConfig async ¶

Update the state of a thread.

This method calls POST /threads/{thread_id}/state.

Parameters:

config (RunnableConfig) – 

A RunnableConfig that includes thread_id in the configurable field.

values (Optional[Union[dict[str, Any], Any]]) – 

Values to update to the state.

as_node (Optional[str], default: None ) – 

Update the state as if this node had just executed.

Returns:

RunnableConfig – 

RunnableConfig for the updated thread.

 stream(input: Union[dict[str, Any], Any], config: Optional[RunnableConfig] = None, *, stream_mode: Optional[Union[StreamMode, list[StreamMode]]] = None, interrupt_before: Optional[Union[All, Sequence[str]]] = None, interrupt_after: Optional[Union[All, Sequence[str]]] = None, subgraphs: bool = False, **kwargs: Any) -> Iterator[Union[dict[str, Any], Any]] ¶

Create a run and stream the results.

This method calls POST /threads/{thread_id}/runs/stream if a thread_id is speciffed in the configurable field of the config or POST /runs/stream otherwise.

Parameters:

input (Union[dict[str, Any], Any]) – 

Input to the graph.

config (Optional[RunnableConfig], default: None ) – 

A RunnableConfig for graph invocation.

stream_mode (Optional[Union[StreamMode, list[StreamMode]]], default: None ) – 

Stream mode(s) to use.

interrupt_before (Optional[Union[All, Sequence[str]]], default: None ) – 

Interrupt the graph before these nodes.

interrupt_after (Optional[Union[All, Sequence[str]]], default: None ) – 

Interrupt the graph after these nodes.

subgraphs (bool, default: False ) – 

Stream from subgraphs.

**kwargs (Any, default: {} ) – 

Additional params to pass to client.runs.stream.

Yields:

Union[dict[str, Any], Any] – 

The output of the graph.

 astream(input: Union[dict[str, Any], Any], config: Optional[RunnableConfig] = None, *, stream_mode: Optional[Union[StreamMode, list[StreamMode]]] = None, interrupt_before: Optional[Union[All, Sequence[str]]] = None, interrupt_after: Optional[Union[All, Sequence[str]]] = None, subgraphs: bool = False, **kwargs: Any) -> AsyncIterator[Union[dict[str, Any], Any]] async ¶

Create a run and stream the results.

This method calls POST /threads/{thread_id}/runs/stream if a thread_id is speciffed in the configurable field of the config or POST /runs/stream otherwise.

Parameters:

input (Union[dict[str, Any], Any]) – 

Input to the graph.

config (Optional[RunnableConfig], default: None ) – 

A RunnableConfig for graph invocation.

stream_mode (Optional[Union[StreamMode, list[StreamMode]]], default: None ) – 

Stream mode(s) to use.

interrupt_before (Optional[Union[All, Sequence[str]]], default: None ) – 

Interrupt the graph before these nodes.

interrupt_after (Optional[Union[All, Sequence[str]]], default: None ) – 

Interrupt the graph after these nodes.

subgraphs (bool, default: False ) – 

Stream from subgraphs.

**kwargs (Any, default: {} ) – 

Additional params to pass to client.runs.stream.

Yields:

AsyncIterator[Union[dict[str, Any], Any]] – 

The output of the graph.

 invoke(input: Union[dict[str, Any], Any], config: Optional[RunnableConfig] = None, *, interrupt_before: Optional[Union[All, Sequence[str]]] = None, interrupt_after: Optional[Union[All, Sequence[str]]] = None, **kwargs: Any) -> Union[dict[str, Any], Any] ¶

Create a run, wait until it finishes and return the final state.

Parameters:

input (Union[dict[str, Any], Any]) – 

Input to the graph.

config (Optional[RunnableConfig], default: None ) – 

A RunnableConfig for graph invocation.

interrupt_before (Optional[Union[All, Sequence[str]]], default: None ) – 

Interrupt the graph before these nodes.

interrupt_after (Optional[Union[All, Sequence[str]]], default: None ) – 

Interrupt the graph after these nodes.

**kwargs (Any, default: {} ) – 

Additional params to pass to RemoteGraph.stream.

Returns:

Union[dict[str, Any], Any] – 

The output of the graph.

 ainvoke(input: Union[dict[str, Any], Any], config: Optional[RunnableConfig] = None, *, interrupt_before: Optional[Union[All, Sequence[str]]] = None, interrupt_after: Optional[Union[All, Sequence[str]]] = None, **kwargs: Any) -> Union[dict[str, Any], Any] async ¶

Create a run, wait until it finishes and return the final state.

Parameters:

input (Union[dict[str, Any], Any]) – 

Input to the graph.

config (Optional[RunnableConfig], default: None ) – 

A RunnableConfig for graph invocation.

interrupt_before (Optional[Union[All, Sequence[str]]], default: None ) – 

Interrupt the graph before these nodes.

interrupt_after (Optional[Union[All, Sequence[str]]], default: None ) – 

Interrupt the graph after these nodes.

**kwargs (Any, default: {} ) – 

Additional params to pass to RemoteGraph.astream.

Returns:

Union[dict[str, Any], Any] – 

The output of the graph.

Comments
