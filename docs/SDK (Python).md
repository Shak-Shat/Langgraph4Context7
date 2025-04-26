# SDK (Python)



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/
- **html**: API reference
LangGraph Platform
Python SDK Reference¶

The LangGraph client implementations connect to the LangGraph API.

This module provides both asynchronous (LangGraphClient) and synchronous (SyncLanggraphClient) clients to interacting with the LangGraph API's core resources such as Assistants, Threads, Runs, and Cron jobs, as well as its persistent document Store.

 LangGraphClient ¶

Top-level client for LangGraph API.

Attributes:

assistants – 

Manages versioned configuration for your graphs.

threads – 

Handles (potentially) multi-turn interactions, such as conversational threads.

runs – 

Controls individual invocations of the graph.

crons – 

Manages scheduled operations.

store – 

Interfaces with persistent, shared data storage.

 HttpClient ¶

Handle async requests to the LangGraph API.

Adds additional error messaging & content handling above the provided httpx client.

Attributes:

client (AsyncClient) – 

Underlying HTTPX async client.

 get(path: str, *, params: Optional[QueryParamTypes] = None) -> Any async ¶

Send a GET request.

 post(path: str, *, json: Optional[dict]) -> Any async ¶

Send a POST request.

 put(path: str, *, json: dict) -> Any async ¶

Send a PUT request.

 patch(path: str, *, json: dict) -> Any async ¶

Send a PATCH request.

 delete(path: str, *, json: Optional[Any] = None) -> None async ¶

Send a DELETE request.

 stream(path: str, method: str, *, json: Optional[dict] = None, params: Optional[QueryParamTypes] = None) -> AsyncIterator[StreamPart] async ¶

Stream results using SSE.

 AssistantsClient ¶

Client for managing assistants in LangGraph.

This class provides methods to interact with assistants, which are versioned configurations of your graph.

Example:

client = get_client()
assistant = await client.assistants.get("assistant_id_123")

 get(assistant_id: str) -> Assistant async ¶

Get an assistant by ID.

Parameters:

assistant_id (str) – 

The ID of the assistant to get.

Returns:

Assistant ( Assistant ) – 

Assistant Object.

Example Usage:

assistant = await client.assistants.get(
    assistant_id="my_assistant_id"
)
print(assistant)

----------------------------------------------------

{
    'assistant_id': 'my_assistant_id',
    'graph_id': 'agent',
    'created_at': '2024-06-25T17:10:33.109781+00:00',
    'updated_at': '2024-06-25T17:10:33.109781+00:00',
    'config': {},
    'metadata': {'created_by': 'system'},
    'version': 1,
    'name': 'my_assistant'
}

 get_graph(assistant_id: str, *, xray: Union[int, bool] = False) -> dict[str, list[dict[str, Any]]] async ¶

Get the graph of an assistant by ID.

Parameters:

assistant_id (str) – 

The ID of the assistant to get the graph of.

xray (Union[int, bool], default: False ) – 

Include graph representation of subgraphs. If an integer value is provided, only subgraphs with a depth less than or equal to the value will be included.

Returns:

Graph ( dict[str, list[dict[str, Any]]] ) – 

The graph information for the assistant in JSON format.

Example Usage:

graph_info = await client.assistants.get_graph(
    assistant_id="my_assistant_id"
)
print(graph_info)

--------------------------------------------------------------------------------------------------------------------------

{
    'nodes':
        [
            {'id': '__start__', 'type': 'schema', 'data': '__start__'},
            {'id': '__end__', 'type': 'schema', 'data': '__end__'},
            {'id': 'agent','type': 'runnable','data': {'id': ['langgraph', 'utils', 'RunnableCallable'],'name': 'agent'}},
        ],
    'edges':
        [
            {'source': '__start__', 'target': 'agent'},
            {'source': 'agent','target': '__end__'}
        ]
}

 get_schemas(assistant_id: str) -> GraphSchema async ¶

Get the schemas of an assistant by ID.

Parameters:

assistant_id (str) – 

The ID of the assistant to get the schema of.

Returns:

GraphSchema ( GraphSchema ) – 

The graph schema for the assistant.

Example Usage:

schema = await client.assistants.get_schemas(
    assistant_id="my_assistant_id"
)
print(schema)

----------------------------------------------------------------------------------------------------------------------------

{
    'graph_id': 'agent',
    'state_schema':
        {
            'title': 'LangGraphInput',
            '$ref': '#/definitions/AgentState',
            'definitions':
                {
                    'BaseMessage':
                        {
                            'title': 'BaseMessage',
                            'description': 'Base abstract Message class. Messages are the inputs and outputs of ChatModels.',
                            'type': 'object',
                            'properties':
                                {
                                 'content':
                                    {
                                        'title': 'Content',
                                        'anyOf': [
                                            {'type': 'string'},
                                            {'type': 'array','items': {'anyOf': [{'type': 'string'}, {'type': 'object'}]}}
                                        ]
                                    },
                                'additional_kwargs':
                                    {
                                        'title': 'Additional Kwargs',
                                        'type': 'object'
                                    },
                                'response_metadata':
                                    {
                                        'title': 'Response Metadata',
                                        'type': 'object'
                                    },
                                'type':
                                    {
                                        'title': 'Type',
                                        'type': 'string'
                                    },
                                'name':
                                    {
                                        'title': 'Name',
                                        'type': 'string'
                                    },
                                'id':
                                    {
                                        'title': 'Id',
                                        'type': 'string'
                                    }
                                },
                            'required': ['content', 'type']
                        },
                    'AgentState':
                        {
                            'title': 'AgentState',
                            'type': 'object',
                            'properties':
                                {
                                    'messages':
                                        {
                                            'title': 'Messages',
                                            'type': 'array',
                                            'items': {'$ref': '#/definitions/BaseMessage'}
                                        }
                                },
                            'required': ['messages']
                        }
                }
        },
    'config_schema':
        {
            'title': 'Configurable',
            'type': 'object',
            'properties':
                {
                    'model_name':
                        {
                            'title': 'Model Name',
                            'enum': ['anthropic', 'openai'],
                            'type': 'string'
                        }
                }
        }
}

 get_subgraphs(assistant_id: str, namespace: Optional[str] = None, recurse: bool = False) -> Subgraphs async ¶

Get the schemas of an assistant by ID.

Parameters:

assistant_id (str) – 

The ID of the assistant to get the schema of.

Returns:

Subgraphs ( Subgraphs ) – 

The graph schema for the assistant.

 create(graph_id: Optional[str], config: Optional[Config] = None, *, metadata: Json = None, assistant_id: Optional[str] = None, if_exists: Optional[OnConflictBehavior] = None, name: Optional[str] = None) -> Assistant async ¶

Create a new assistant.

Useful when graph is configurable and you want to create different assistants based on different configurations.

Parameters:

graph_id (Optional[str]) – 

The ID of the graph the assistant should use. The graph ID is normally set in your langgraph.json configuration.

config (Optional[Config], default: None ) – 

Configuration to use for the graph.

metadata (Json, default: None ) – 

Metadata to add to assistant.

assistant_id (Optional[str], default: None ) – 

Assistant ID to use, will default to a random UUID if not provided.

if_exists (Optional[OnConflictBehavior], default: None ) – 

How to handle duplicate creation. Defaults to 'raise' under the hood. Must be either 'raise' (raise error if duplicate), or 'do_nothing' (return existing assistant).

name (Optional[str], default: None ) – 

The name of the assistant. Defaults to 'Untitled' under the hood.

Returns:

Assistant ( Assistant ) – 

The created assistant.

Example Usage:

assistant = await client.assistants.create(
    graph_id="agent",
    config={"configurable": {"model_name": "openai"}},
    metadata={"number":1},
    assistant_id="my-assistant-id",
    if_exists="do_nothing",
    name="my_name"
)

 update(assistant_id: str, *, graph_id: Optional[str] = None, config: Optional[Config] = None, metadata: Json = None, name: Optional[str] = None) -> Assistant async ¶

Update an assistant.

Use this to point to a different graph, update the configuration, or change the metadata of an assistant.

Parameters:

assistant_id (str) – 

Assistant to update.

graph_id (Optional[str], default: None ) – 

The ID of the graph the assistant should use. The graph ID is normally set in your langgraph.json configuration. If None, assistant will keep pointing to same graph.

config (Optional[Config], default: None ) – 

Configuration to use for the graph.

metadata (Json, default: None ) – 

Metadata to merge with existing assistant metadata.

name (Optional[str], default: None ) – 

The new name for the assistant.

Returns:

Assistant ( Assistant ) – 

The updated assistant.

Example Usage:

assistant = await client.assistants.update(
    assistant_id='e280dad7-8618-443f-87f1-8e41841c180f',
    graph_id="other-graph",
    config={"configurable": {"model_name": "anthropic"}},
    metadata={"number":2}
)

 delete(assistant_id: str) -> None async ¶

Delete an assistant.

Parameters:

assistant_id (str) – 

The assistant ID to delete.

Returns:

None – 

None

Example Usage:

await client.assistants.delete(
    assistant_id="my_assistant_id"
)

 search(*, metadata: Json = None, graph_id: Optional[str] = None, limit: int = 10, offset: int = 0) -> list[Assistant] async ¶

Search for assistants.

Parameters:

metadata (Json, default: None ) – 

Metadata to filter by. Exact match filter for each KV pair.

graph_id (Optional[str], default: None ) – 

The ID of the graph to filter by. The graph ID is normally set in your langgraph.json configuration.

limit (int, default: 10 ) – 

The maximum number of results to return.

offset (int, default: 0 ) – 

The number of results to skip.

Returns:

list[Assistant] – 

list[Assistant]: A list of assistants.

Example Usage:

assistants = await client.assistants.search(
    metadata = {"name":"my_name"},
    graph_id="my_graph_id",
    limit=5,
    offset=5
)

 get_versions(assistant_id: str, metadata: Json = None, limit: int = 10, offset: int = 0) -> list[AssistantVersion] async ¶

List all versions of an assistant.

Parameters:

assistant_id (str) – 

The assistant ID to get versions for.

metadata (Json, default: None ) – 

Metadata to filter versions by. Exact match filter for each KV pair.

limit (int, default: 10 ) – 

The maximum number of versions to return.

offset (int, default: 0 ) – 

The number of versions to skip.

Returns:

list[AssistantVersion] – 

list[AssistantVersion]: A list of assistant versions.

Example Usage:

assistant_versions = await client.assistants.get_versions(
    assistant_id="my_assistant_id"
)

 set_latest(assistant_id: str, version: int) -> Assistant async ¶

Change the version of an assistant.

Parameters:

assistant_id (str) – 

The assistant ID to delete.

version (int) – 

The version to change to.

Returns:

Assistant ( Assistant ) – 

Assistant Object.

Example Usage:

new_version_assistant = await client.assistants.set_latest(
    assistant_id="my_assistant_id",
    version=3
)

 ThreadsClient ¶

Client for managing threads in LangGraph.

A thread maintains the state of a graph across multiple interactions/invocations (aka runs). It accumulates and persists the graph's state, allowing for continuity between separate invocations of the graph.

Example:

client = get_client()
new_thread = await client.threads.create(metadata={"user_id": "123"})

 get(thread_id: str) -> Thread async ¶

Get a thread by ID.

Parameters:

thread_id (str) – 

The ID of the thread to get.

Returns:

Thread ( Thread ) – 

Thread object.

Example Usage:

thread = await client.threads.get(
    thread_id="my_thread_id"
)
print(thread)

-----------------------------------------------------

{
    'thread_id': 'my_thread_id',
    'created_at': '2024-07-18T18:35:15.540834+00:00',
    'updated_at': '2024-07-18T18:35:15.540834+00:00',
    'metadata': {'graph_id': 'agent'}
}

 create(*, metadata: Json = None, thread_id: Optional[str] = None, if_exists: Optional[OnConflictBehavior] = None) -> Thread async ¶

Create a new thread.

Parameters:

metadata (Json, default: None ) – 

Metadata to add to thread.

thread_id (Optional[str], default: None ) – 

ID of thread. If None, ID will be a randomly generated UUID.

if_exists (Optional[OnConflictBehavior], default: None ) – 

How to handle duplicate creation. Defaults to 'raise' under the hood. Must be either 'raise' (raise error if duplicate), or 'do_nothing' (return existing thread).

Returns:

Thread ( Thread ) – 

The created thread.

Example Usage:

thread = await client.threads.create(
    metadata={"number":1},
    thread_id="my-thread-id",
    if_exists="raise"
)

 update(thread_id: str, *, metadata: dict[str, Any]) -> Thread async ¶

Update a thread.

Parameters:

thread_id (str) – 

ID of thread to update.

metadata (dict[str, Any]) – 

Metadata to merge with existing thread metadata.

Returns:

Thread ( Thread ) – 

The created thread.

Example Usage:

thread = await client.threads.update(
    thread_id="my-thread-id",
    metadata={"number":1},
)

 delete(thread_id: str) -> None async ¶

Delete a thread.

Parameters:

thread_id (str) – 

The ID of the thread to delete.

Returns:

None – 

None

Example Usage:

await client.threads.delete(
    thread_id="my_thread_id"
)

 search(*, metadata: Json = None, values: Json = None, status: Optional[ThreadStatus] = None, limit: int = 10, offset: int = 0) -> list[Thread] async ¶

Search for threads.

Parameters:

metadata (Json, default: None ) – 

Thread metadata to filter on.

values (Json, default: None ) – 

State values to filter on.

status (Optional[ThreadStatus], default: None ) – 

Thread status to filter on. Must be one of 'idle', 'busy', 'interrupted' or 'error'.

limit (int, default: 10 ) – 

Limit on number of threads to return.

offset (int, default: 0 ) – 

Offset in threads table to start search from.

Returns:

list[Thread] – 

list[Thread]: List of the threads matching the search parameters.

Example Usage:

threads = await client.threads.search(
    metadata={"number":1},
    status="interrupted",
    limit=15,
    offset=5
)

 copy(thread_id: str) -> None async ¶

Copy a thread.

Parameters:

thread_id (str) – 

The ID of the thread to copy.

Returns:

None – 

None

Example Usage:

await client.threads.copy(
    thread_id="my_thread_id"
)

 get_state(thread_id: str, checkpoint: Optional[Checkpoint] = None, checkpoint_id: Optional[str] = None, *, subgraphs: bool = False) -> ThreadState async ¶

Get the state of a thread.

Parameters:

thread_id (str) – 

The ID of the thread to get the state of.

checkpoint (Optional[Checkpoint], default: None ) – 

The checkpoint to get the state of.

subgraphs (bool, default: False ) – 

Include subgraphs states.

Returns:

ThreadState ( ThreadState ) – 

the thread of the state.

Example Usage:

thread_state = await client.threads.get_state(
    thread_id="my_thread_id",
    checkpoint_id="my_checkpoint_id"
)
print(thread_state)

----------------------------------------------------------------------------------------------------------------------------------------------------------------------

{
    'values': {
        'messages': [
            {
                'content': 'how are you?',
                'additional_kwargs': {},
                'response_metadata': {},
                'type': 'human',
                'name': None,
                'id': 'fe0a5778-cfe9-42ee-b807-0adaa1873c10',
                'example': False
            },
            {
                'content': "I'm doing well, thanks for asking! I'm an AI assistant created by Anthropic to be helpful, honest, and harmless.",
                'additional_kwargs': {},
                'response_metadata': {},
                'type': 'ai',
                'name': None,
                'id': 'run-159b782c-b679-4830-83c6-cef87798fe8b',
                'example': False,
                'tool_calls': [],
                'invalid_tool_calls': [],
                'usage_metadata': None
            }
        ]
    },
    'next': [],
    'checkpoint':
        {
            'thread_id': 'e2496803-ecd5-4e0c-a779-3226296181c2',
            'checkpoint_ns': '',
            'checkpoint_id': '1ef4a9b8-e6fb-67b1-8001-abd5184439d1'
        }
    'metadata':
        {
            'step': 1,
            'run_id': '1ef4a9b8-d7da-679a-a45a-872054341df2',
            'source': 'loop',
            'writes':
                {
                    'agent':
                        {
                            'messages': [
                                {
                                    'id': 'run-159b782c-b679-4830-83c6-cef87798fe8b',
                                    'name': None,
                                    'type': 'ai',
                                    'content': "I'm doing well, thanks for asking! I'm an AI assistant created by Anthropic to be helpful, honest, and harmless.",
                                    'example': False,
                                    'tool_calls': [],
                                    'usage_metadata': None,
                                    'additional_kwargs': {},
                                    'response_metadata': {},
                                    'invalid_tool_calls': []
                                }
                            ]
                        }
                },
    'user_id': None,
    'graph_id': 'agent',
    'thread_id': 'e2496803-ecd5-4e0c-a779-3226296181c2',
    'created_by': 'system',
    'assistant_id': 'fe096781-5601-53d2-b2f6-0d3403f7e9ca'},
    'created_at': '2024-07-25T15:35:44.184703+00:00',
    'parent_config':
        {
            'thread_id': 'e2496803-ecd5-4e0c-a779-3226296181c2',
            'checkpoint_ns': '',
            'checkpoint_id': '1ef4a9b8-d80d-6fa7-8000-9300467fad0f'
        }
}

 update_state(thread_id: str, values: Optional[Union[dict, Sequence[dict]]], *, as_node: Optional[str] = None, checkpoint: Optional[Checkpoint] = None, checkpoint_id: Optional[str] = None) -> ThreadUpdateStateResponse async ¶

Update the state of a thread.

Parameters:

thread_id (str) – 

The ID of the thread to update.

values (Optional[Union[dict, Sequence[dict]]]) – 

The values to update the state with.

as_node (Optional[str], default: None ) – 

Update the state as if this node had just executed.

checkpoint (Optional[Checkpoint], default: None ) – 

The checkpoint to update the state of.

Returns:

ThreadUpdateStateResponse ( ThreadUpdateStateResponse ) – 

Response after updating a thread's state.

Example Usage:

response = await client.threads.update_state(
    thread_id="my_thread_id",
    values={"messages":[{"role": "user", "content": "hello!"}]},
    as_node="my_node",
)
print(response)

----------------------------------------------------------------------------------------------------------------------------------------------------------------------

{
    'checkpoint': {
        'thread_id': 'e2496803-ecd5-4e0c-a779-3226296181c2',
        'checkpoint_ns': '',
        'checkpoint_id': '1ef4a9b8-e6fb-67b1-8001-abd5184439d1',
        'checkpoint_map': {}
    }
}

 get_history(thread_id: str, *, limit: int = 10, before: Optional[str | Checkpoint] = None, metadata: Optional[dict] = None, checkpoint: Optional[Checkpoint] = None) -> list[ThreadState] async ¶

Get the state history of a thread.

Parameters:

thread_id (str) – 

The ID of the thread to get the state history for.

checkpoint (Optional[Checkpoint], default: None ) – 

Return states for this subgraph. If empty defaults to root.

limit (int, default: 10 ) – 

The maximum number of states to return.

before (Optional[str | Checkpoint], default: None ) – 

Return states before this checkpoint.

metadata (Optional[dict], default: None ) – 

Filter states by metadata key-value pairs.

Returns:

list[ThreadState] – 

list[ThreadState]: the state history of the thread.

Example Usage:

thread_state = await client.threads.get_history(
    thread_id="my_thread_id",
    limit=5,
)

 RunsClient ¶

Client for managing runs in LangGraph.

A run is a single assistant invocation with optional input, config, and metadata. This client manages runs, which can be stateful (on threads) or stateless.

Example:

client = get_client()
run = await client.runs.create(assistant_id="asst_123", thread_id="thread_456", input={"query": "Hello"})

 stream(thread_id: Optional[str], assistant_id: str, *, input: Optional[dict] = None, command: Optional[Command] = None, stream_mode: Union[StreamMode, Sequence[StreamMode]] = 'values', stream_subgraphs: bool = False, metadata: Optional[dict] = None, config: Optional[Config] = None, checkpoint: Optional[Checkpoint] = None, checkpoint_id: Optional[str] = None, interrupt_before: Optional[Union[All, Sequence[str]]] = None, interrupt_after: Optional[Union[All, Sequence[str]]] = None, feedback_keys: Optional[Sequence[str]] = None, on_disconnect: Optional[DisconnectMode] = None, on_completion: Optional[OnCompletionBehavior] = None, webhook: Optional[str] = None, multitask_strategy: Optional[MultitaskStrategy] = None, if_not_exists: Optional[IfNotExists] = None, after_seconds: Optional[int] = None) -> AsyncIterator[StreamPart] ¶

Create a run and stream the results.

Parameters:

thread_id (Optional[str]) – 

the thread ID to assign to the thread. If None will create a stateless run.

assistant_id (str) – 

The assistant ID or graph name to stream from. If using graph name, will default to first assistant created from that graph.

input (Optional[dict], default: None ) – 

The input to the graph.

command (Optional[Command], default: None ) – 

A command to execute. Cannot be combined with input.

stream_mode (Union[StreamMode, Sequence[StreamMode]], default: 'values' ) – 

The stream mode(s) to use.

stream_subgraphs (bool, default: False ) – 

Whether to stream output from subgraphs.

metadata (Optional[dict], default: None ) – 

Metadata to assign to the run.

config (Optional[Config], default: None ) – 

The configuration for the assistant.

checkpoint (Optional[Checkpoint], default: None ) – 

The checkpoint to resume from.

interrupt_before (Optional[Union[All, Sequence[str]]], default: None ) – 

Nodes to interrupt immediately before they get executed.

interrupt_after (Optional[Union[All, Sequence[str]]], default: None ) – 

Nodes to Nodes to interrupt immediately after they get executed.

feedback_keys (Optional[Sequence[str]], default: None ) – 

Feedback keys to assign to run.

on_disconnect (Optional[DisconnectMode], default: None ) – 

The disconnect mode to use. Must be one of 'cancel' or 'continue'.

on_completion (Optional[OnCompletionBehavior], default: None ) – 

Whether to delete or keep the thread created for a stateless run. Must be one of 'delete' or 'keep'.

webhook (Optional[str], default: None ) – 

Webhook to call after LangGraph API call is done.

multitask_strategy (Optional[MultitaskStrategy], default: None ) – 

Multitask strategy to use. Must be one of 'reject', 'interrupt', 'rollback', or 'enqueue'.

if_not_exists (Optional[IfNotExists], default: None ) – 

How to handle missing thread. Defaults to 'reject'. Must be either 'reject' (raise error if missing), or 'create' (create new thread).

after_seconds (Optional[int], default: None ) – 

The number of seconds to wait before starting the run. Use to schedule future runs.

Returns:

AsyncIterator[StreamPart] – 

AsyncIterator[StreamPart]: Asynchronous iterator of stream results.

Example Usage:

async for chunk in client.runs.stream(
    thread_id=None,
    assistant_id="agent",
    input={"messages": [{"role": "user", "content": "how are you?"}]},
    stream_mode=["values","debug"],
    metadata={"name":"my_run"},
    config={"configurable": {"model_name": "anthropic"}},
    interrupt_before=["node_to_stop_before_1","node_to_stop_before_2"],
    interrupt_after=["node_to_stop_after_1","node_to_stop_after_2"],
    feedback_keys=["my_feedback_key_1","my_feedback_key_2"],
    webhook="https://my.fake.webhook.com",
    multitask_strategy="interrupt"
):
    print(chunk)

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

StreamPart(event='metadata', data={'run_id': '1ef4a9b8-d7da-679a-a45a-872054341df2'})
StreamPart(event='values', data={'messages': [{'content': 'how are you?', 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'human', 'name': None, 'id': 'fe0a5778-cfe9-42ee-b807-0adaa1873c10', 'example': False}]})
StreamPart(event='values', data={'messages': [{'content': 'how are you?', 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'human', 'name': None, 'id': 'fe0a5778-cfe9-42ee-b807-0adaa1873c10', 'example': False}, {'content': "I'm doing well, thanks for asking! I'm an AI assistant created by Anthropic to be helpful, honest, and harmless.", 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'ai', 'name': None, 'id': 'run-159b782c-b679-4830-83c6-cef87798fe8b', 'example': False, 'tool_calls': [], 'invalid_tool_calls': [], 'usage_metadata': None}]})
StreamPart(event='end', data=None)

 create(thread_id: Optional[str], assistant_id: str, *, input: Optional[dict] = None, command: Optional[Command] = None, stream_mode: Union[StreamMode, Sequence[StreamMode]] = 'values', stream_subgraphs: bool = False, metadata: Optional[dict] = None, config: Optional[Config] = None, checkpoint: Optional[Checkpoint] = None, checkpoint_id: Optional[str] = None, interrupt_before: Optional[Union[All, Sequence[str]]] = None, interrupt_after: Optional[Union[All, Sequence[str]]] = None, webhook: Optional[str] = None, multitask_strategy: Optional[MultitaskStrategy] = None, if_not_exists: Optional[IfNotExists] = None, on_completion: Optional[OnCompletionBehavior] = None, after_seconds: Optional[int] = None) -> Run async ¶

Create a background run.

Parameters:

thread_id (Optional[str]) – 

the thread ID to assign to the thread. If None will create a stateless run.

assistant_id (str) – 

The assistant ID or graph name to stream from. If using graph name, will default to first assistant created from that graph.

input (Optional[dict], default: None ) – 

The input to the graph.

command (Optional[Command], default: None ) – 

A command to execute. Cannot be combined with input.

stream_mode (Union[StreamMode, Sequence[StreamMode]], default: 'values' ) – 

The stream mode(s) to use.

stream_subgraphs (bool, default: False ) – 

Whether to stream output from subgraphs.

metadata (Optional[dict], default: None ) – 

Metadata to assign to the run.

config (Optional[Config], default: None ) – 

The configuration for the assistant.

checkpoint (Optional[Checkpoint], default: None ) – 

The checkpoint to resume from.

interrupt_before (Optional[Union[All, Sequence[str]]], default: None ) – 

Nodes to interrupt immediately before they get executed.

interrupt_after (Optional[Union[All, Sequence[str]]], default: None ) – 

Nodes to Nodes to interrupt immediately after they get executed.

webhook (Optional[str], default: None ) – 

Webhook to call after LangGraph API call is done.

multitask_strategy (Optional[MultitaskStrategy], default: None ) – 

Multitask strategy to use. Must be one of 'reject', 'interrupt', 'rollback', or 'enqueue'.

on_completion (Optional[OnCompletionBehavior], default: None ) – 

Whether to delete or keep the thread created for a stateless run. Must be one of 'delete' or 'keep'.

if_not_exists (Optional[IfNotExists], default: None ) – 

How to handle missing thread. Defaults to 'reject'. Must be either 'reject' (raise error if missing), or 'create' (create new thread).

after_seconds (Optional[int], default: None ) – 

The number of seconds to wait before starting the run. Use to schedule future runs.

Returns:

Run ( Run ) – 

The created background run.

Example Usage:

background_run = await client.runs.create(
    thread_id="my_thread_id",
    assistant_id="my_assistant_id",
    input={"messages": [{"role": "user", "content": "hello!"}]},
    metadata={"name":"my_run"},
    config={"configurable": {"model_name": "openai"}},
    interrupt_before=["node_to_stop_before_1","node_to_stop_before_2"],
    interrupt_after=["node_to_stop_after_1","node_to_stop_after_2"],
    webhook="https://my.fake.webhook.com",
    multitask_strategy="interrupt"
)
print(background_run)

--------------------------------------------------------------------------------

{
    'run_id': 'my_run_id',
    'thread_id': 'my_thread_id',
    'assistant_id': 'my_assistant_id',
    'created_at': '2024-07-25T15:35:42.598503+00:00',
    'updated_at': '2024-07-25T15:35:42.598503+00:00',
    'metadata': {},
    'status': 'pending',
    'kwargs':
        {
            'input':
                {
                    'messages': [
                        {
                            'role': 'user',
                            'content': 'how are you?'
                        }
                    ]
                },
            'config':
                {
                    'metadata':
                        {
                            'created_by': 'system'
                        },
                    'configurable':
                        {
                            'run_id': 'my_run_id',
                            'user_id': None,
                            'graph_id': 'agent',
                            'thread_id': 'my_thread_id',
                            'checkpoint_id': None,
                            'model_name': "openai",
                            'assistant_id': 'my_assistant_id'
                        }
                },
            'webhook': "https://my.fake.webhook.com",
            'temporary': False,
            'stream_mode': ['values'],
            'feedback_keys': None,
            'interrupt_after': ["node_to_stop_after_1","node_to_stop_after_2"],
            'interrupt_before': ["node_to_stop_before_1","node_to_stop_before_2"]
        },
    'multitask_strategy': 'interrupt'
}

 create_batch(payloads: list[RunCreate]) -> list[Run] async ¶

Create a batch of stateless background runs.

 wait(thread_id: Optional[str], assistant_id: str, *, input: Optional[dict] = None, command: Optional[Command] = None, metadata: Optional[dict] = None, config: Optional[Config] = None, checkpoint: Optional[Checkpoint] = None, checkpoint_id: Optional[str] = None, interrupt_before: Optional[Union[All, Sequence[str]]] = None, interrupt_after: Optional[Union[All, Sequence[str]]] = None, webhook: Optional[str] = None, on_disconnect: Optional[DisconnectMode] = None, on_completion: Optional[OnCompletionBehavior] = None, multitask_strategy: Optional[MultitaskStrategy] = None, if_not_exists: Optional[IfNotExists] = None, after_seconds: Optional[int] = None, raise_error: bool = True) -> Union[list[dict], dict[str, Any]] async ¶

Create a run, wait until it finishes and return the final state.

Parameters:

thread_id (Optional[str]) – 

the thread ID to create the run on. If None will create a stateless run.

assistant_id (str) – 

The assistant ID or graph name to run. If using graph name, will default to first assistant created from that graph.

input (Optional[dict], default: None ) – 

The input to the graph.

command (Optional[Command], default: None ) – 

A command to execute. Cannot be combined with input.

metadata (Optional[dict], default: None ) – 

Metadata to assign to the run.

config (Optional[Config], default: None ) – 

The configuration for the assistant.

checkpoint (Optional[Checkpoint], default: None ) – 

The checkpoint to resume from.

interrupt_before (Optional[Union[All, Sequence[str]]], default: None ) – 

Nodes to interrupt immediately before they get executed.

interrupt_after (Optional[Union[All, Sequence[str]]], default: None ) – 

Nodes to Nodes to interrupt immediately after they get executed.

webhook (Optional[str], default: None ) – 

Webhook to call after LangGraph API call is done.

on_disconnect (Optional[DisconnectMode], default: None ) – 

The disconnect mode to use. Must be one of 'cancel' or 'continue'.

on_completion (Optional[OnCompletionBehavior], default: None ) – 

Whether to delete or keep the thread created for a stateless run. Must be one of 'delete' or 'keep'.

multitask_strategy (Optional[MultitaskStrategy], default: None ) – 

Multitask strategy to use. Must be one of 'reject', 'interrupt', 'rollback', or 'enqueue'.

if_not_exists (Optional[IfNotExists], default: None ) – 

How to handle missing thread. Defaults to 'reject'. Must be either 'reject' (raise error if missing), or 'create' (create new thread).

after_seconds (Optional[int], default: None ) – 

The number of seconds to wait before starting the run. Use to schedule future runs.

Returns:

Union[list[dict], dict[str, Any]] – 

Union[list[dict], dict[str, Any]]: The output of the run.

Example Usage:

final_state_of_run = await client.runs.wait(
    thread_id=None,
    assistant_id="agent",
    input={"messages": [{"role": "user", "content": "how are you?"}]},
    metadata={"name":"my_run"},
    config={"configurable": {"model_name": "anthropic"}},
    interrupt_before=["node_to_stop_before_1","node_to_stop_before_2"],
    interrupt_after=["node_to_stop_after_1","node_to_stop_after_2"],
    webhook="https://my.fake.webhook.com",
    multitask_strategy="interrupt"
)
print(final_state_of_run)

-------------------------------------------------------------------------------------------------------------------------------------------

{
    'messages': [
        {
            'content': 'how are you?',
            'additional_kwargs': {},
            'response_metadata': {},
            'type': 'human',
            'name': None,
            'id': 'f51a862c-62fe-4866-863b-b0863e8ad78a',
            'example': False
        },
        {
            'content': "I'm doing well, thanks for asking! I'm an AI assistant created by Anthropic to be helpful, honest, and harmless.",
            'additional_kwargs': {},
            'response_metadata': {},
            'type': 'ai',
            'name': None,
            'id': 'run-bf1cd3c6-768f-4c16-b62d-ba6f17ad8b36',
            'example': False,
            'tool_calls': [],
            'invalid_tool_calls': [],
            'usage_metadata': None
        }
    ]
}

 list(thread_id: str, *, limit: int = 10, offset: int = 0, status: Optional[RunStatus] = None) -> List[Run] async ¶

List runs.

Parameters:

thread_id (str) – 

The thread ID to list runs for.

limit (int, default: 10 ) – 

The maximum number of results to return.

offset (int, default: 0 ) – 

The number of results to skip.

status (Optional[RunStatus], default: None ) – 

The status of the run to filter by.

Returns:

List[Run] – 

List[Run]: The runs for the thread.

Example Usage:

await client.runs.list(
    thread_id="thread_id",
    limit=5,
    offset=5,
)

 get(thread_id: str, run_id: str) -> Run async ¶

Get a run.

Parameters:

thread_id (str) – 

The thread ID to get.

run_id (str) – 

The run ID to get.

Returns:

Run ( Run ) – 

Run object.

Example Usage:

run = await client.runs.get(
    thread_id="thread_id_to_delete",
    run_id="run_id_to_delete",
)

 cancel(thread_id: str, run_id: str, *, wait: bool = False, action: CancelAction = 'interrupt') -> None async ¶

Get a run.

Parameters:

thread_id (str) – 

The thread ID to cancel.

run_id (str) – 

The run ID to cancel.

wait (bool, default: False ) – 

Whether to wait until run has completed.

action (CancelAction, default: 'interrupt' ) – 

Action to take when cancelling the run. Possible values are interrupt or rollback. Default is interrupt.

Returns:

None – 

None

Example Usage:

await client.runs.cancel(
    thread_id="thread_id_to_cancel",
    run_id="run_id_to_cancel",
    wait=True,
    action="interrupt"
)

 join(thread_id: str, run_id: str) -> dict async ¶

Block until a run is done. Returns the final state of the thread.

Parameters:

thread_id (str) – 

The thread ID to join.

run_id (str) – 

The run ID to join.

Returns:

dict – 

None

Example Usage:

result =await client.runs.join(
    thread_id="thread_id_to_join",
    run_id="run_id_to_join"
)

 join_stream(thread_id: str, run_id: str, *, cancel_on_disconnect: bool = False) -> AsyncIterator[StreamPart] ¶

Stream output from a run in real-time, until the run is done. Output is not buffered, so any output produced before this call will not be received here.

Parameters:

thread_id (str) – 

The thread ID to join.

run_id (str) – 

The run ID to join.

cancel_on_disconnect (bool, default: False ) – 

Whether to cancel the run when the stream is disconnected.

Returns:

AsyncIterator[StreamPart] – 

None

Example Usage:

await client.runs.join_stream(
    thread_id="thread_id_to_join",
    run_id="run_id_to_join"
)

 delete(thread_id: str, run_id: str) -> None async ¶

Delete a run.

Parameters:

thread_id (str) – 

The thread ID to delete.

run_id (str) – 

The run ID to delete.

Returns:

None – 

None

Example Usage:

await client.runs.delete(
    thread_id="thread_id_to_delete",
    run_id="run_id_to_delete"
)

 CronClient ¶

Client for managing recurrent runs (cron jobs) in LangGraph.

A run is a single invocation of an assistant with optional input and config. This client allows scheduling recurring runs to occur automatically.

Example:

client = get_client()
cron_job = await client.crons.create_for_thread(
    thread_id="thread_123",
    assistant_id="asst_456",
    schedule="0 9 * * *",
    input={"message": "Daily update"}
)

 create_for_thread(thread_id: str, assistant_id: str, *, schedule: str, input: Optional[dict] = None, metadata: Optional[dict] = None, config: Optional[Config] = None, interrupt_before: Optional[Union[All, list[str]]] = None, interrupt_after: Optional[Union[All, list[str]]] = None, webhook: Optional[str] = None, multitask_strategy: Optional[str] = None) -> Run async ¶

Create a cron job for a thread.

Parameters:

thread_id (str) – 

the thread ID to run the cron job on.

assistant_id (str) – 

The assistant ID or graph name to use for the cron job. If using graph name, will default to first assistant created from that graph.

schedule (str) – 

The cron schedule to execute this job on.

input (Optional[dict], default: None ) – 

The input to the graph.

metadata (Optional[dict], default: None ) – 

Metadata to assign to the cron job runs.

config (Optional[Config], default: None ) – 

The configuration for the assistant.

interrupt_before (Optional[Union[All, list[str]]], default: None ) – 

Nodes to interrupt immediately before they get executed.

interrupt_after (Optional[Union[All, list[str]]], default: None ) – 

Nodes to Nodes to interrupt immediately after they get executed.

webhook (Optional[str], default: None ) – 

Webhook to call after LangGraph API call is done.

multitask_strategy (Optional[str], default: None ) – 

Multitask strategy to use. Must be one of 'reject', 'interrupt', 'rollback', or 'enqueue'.

Returns:

Run ( Run ) – 

The cron run.

Example Usage:

cron_run = await client.crons.create_for_thread(
    thread_id="my-thread-id",
    assistant_id="agent",
    schedule="27 15 * * *",
    input={"messages": [{"role": "user", "content": "hello!"}]},
    metadata={"name":"my_run"},
    config={"configurable": {"model_name": "openai"}},
    interrupt_before=["node_to_stop_before_1","node_to_stop_before_2"],
    interrupt_after=["node_to_stop_after_1","node_to_stop_after_2"],
    webhook="https://my.fake.webhook.com",
    multitask_strategy="interrupt"
)

 create(assistant_id: str, *, schedule: str, input: Optional[dict] = None, metadata: Optional[dict] = None, config: Optional[Config] = None, interrupt_before: Optional[Union[All, list[str]]] = None, interrupt_after: Optional[Union[All, list[str]]] = None, webhook: Optional[str] = None, multitask_strategy: Optional[str] = None) -> Run async ¶

Create a cron run.

Parameters:

assistant_id (str) – 

The assistant ID or graph name to use for the cron job. If using graph name, will default to first assistant created from that graph.

schedule (str) – 

The cron schedule to execute this job on.

input (Optional[dict], default: None ) – 

The input to the graph.

metadata (Optional[dict], default: None ) – 

Metadata to assign to the cron job runs.

config (Optional[Config], default: None ) – 

The configuration for the assistant.

interrupt_before (Optional[Union[All, list[str]]], default: None ) – 

Nodes to interrupt immediately before they get executed.

interrupt_after (Optional[Union[All, list[str]]], default: None ) – 

Nodes to Nodes to interrupt immediately after they get executed.

webhook (Optional[str], default: None ) – 

Webhook to call after LangGraph API call is done.

multitask_strategy (Optional[str], default: None ) – 

Multitask strategy to use. Must be one of 'reject', 'interrupt', 'rollback', or 'enqueue'.

Returns:

Run ( Run ) – 

The cron run.

Example Usage:

cron_run = client.crons.create(
    assistant_id="agent",
    schedule="27 15 * * *",
    input={"messages": [{"role": "user", "content": "hello!"}]},
    metadata={"name":"my_run"},
    config={"configurable": {"model_name": "openai"}},
    interrupt_before=["node_to_stop_before_1","node_to_stop_before_2"],
    interrupt_after=["node_to_stop_after_1","node_to_stop_after_2"],
    webhook="https://my.fake.webhook.com",
    multitask_strategy="interrupt"
)

 delete(cron_id: str) -> None async ¶

Delete a cron.

Parameters:

cron_id (str) – 

The cron ID to delete.

Returns:

None – 

None

Example Usage:

await client.crons.delete(
    cron_id="cron_to_delete"
)

 search(*, assistant_id: Optional[str] = None, thread_id: Optional[str] = None, limit: int = 10, offset: int = 0) -> list[Cron] async ¶

Get a list of cron jobs.

Parameters:

assistant_id (Optional[str], default: None ) – 

The assistant ID or graph name to search for.

thread_id (Optional[str], default: None ) – 

the thread ID to search for.

limit (int, default: 10 ) – 

The maximum number of results to return.

offset (int, default: 0 ) – 

The number of results to skip.

Returns:

list[Cron] – 

list[Cron]: The list of cron jobs returned by the search,

Example Usage:

cron_jobs = await client.crons.search(
    assistant_id="my_assistant_id",
    thread_id="my_thread_id",
    limit=5,
    offset=5,
)
print(cron_jobs)

----------------------------------------------------------

[
    {
        'cron_id': '1ef3cefa-4c09-6926-96d0-3dc97fd5e39b',
        'assistant_id': 'my_assistant_id',
        'thread_id': 'my_thread_id',
        'user_id': None,
        'payload':
            {
                'input': {'start_time': ''},
                'schedule': '4 * * * *',
                'assistant_id': 'my_assistant_id'
            },
        'schedule': '4 * * * *',
        'next_run_date': '2024-07-25T17:04:00+00:00',
        'end_time': None,
        'created_at': '2024-07-08T06:02:23.073257+00:00',
        'updated_at': '2024-07-08T06:02:23.073257+00:00'
    }
]

 StoreClient ¶

Client for interacting with the graph's shared storage.

The Store provides a key-value storage system for persisting data across graph executions, allowing for stateful operations and data sharing across threads.

Example:

client = get_client()
await client.store.put_item(["users", "user123"], "mem-123451342", {"name": "Alice", "score": 100})

 put_item(namespace: Sequence[str], /, key: str, value: dict[str, Any], index: Optional[Union[Literal[False], list[str]]] = None, ttl: Optional[int] = None) -> None async ¶

Store or update an item.

Parameters:

namespace (Sequence[str]) – 

A list of strings representing the namespace path.

key (str) – 

The unique identifier for the item within the namespace.

value (dict[str, Any]) – 

A dictionary containing the item's data.

index (Optional[Union[Literal[False], list[str]]], default: None ) – 

Controls search indexing - None (use defaults), False (disable), or list of field paths to index.

ttl (Optional[int], default: None ) – 

Optional time-to-live in minutes for the item, or None for no expiration.

Returns:

None – 

None

Example Usage:

await client.store.put_item(
    ["documents", "user123"],
    key="item456",
    value={"title": "My Document", "content": "Hello World"}
)

 get_item(namespace: Sequence[str], /, key: str, *, refresh_ttl: Optional[bool] = None) -> Item async ¶

Retrieve a single item.

Parameters:

key (str) – 

The unique identifier for the item.

namespace (Sequence[str]) – 

Optional list of strings representing the namespace path.

refresh_ttl (Optional[bool], default: None ) – 

Whether to refresh the TTL on this read operation. If None, uses the store's default behavior.

Returns:

Item ( Item ) – 

The retrieved item.

Example Usage:

item = await client.store.get_item(
    ["documents", "user123"],
    key="item456",
)
print(item)

----------------------------------------------------------------

{
    'namespace': ['documents', 'user123'],
    'key': 'item456',
    'value': {'title': 'My Document', 'content': 'Hello World'},
    'created_at': '2024-07-30T12:00:00Z',
    'updated_at': '2024-07-30T12:00:00Z'
}

 delete_item(namespace: Sequence[str], /, key: str) -> None async ¶

Delete an item.

Parameters:

key (str) – 

The unique identifier for the item.

namespace (Sequence[str]) – 

Optional list of strings representing the namespace path.

Returns:

None – 

None

Example Usage:

await client.store.delete_item(
    ["documents", "user123"],
    key="item456",
)

 search_items(namespace_prefix: Sequence[str], /, filter: Optional[dict[str, Any]] = None, limit: int = 10, offset: int = 0, query: Optional[str] = None, refresh_ttl: Optional[bool] = None) -> SearchItemsResponse async ¶

Search for items within a namespace prefix.

Parameters:

namespace_prefix (Sequence[str]) – 

List of strings representing the namespace prefix.

filter (Optional[dict[str, Any]], default: None ) – 

Optional dictionary of key-value pairs to filter results.

limit (int, default: 10 ) – 

Maximum number of items to return (default is 10).

offset (int, default: 0 ) – 

Number of items to skip before returning results (default is 0).

query (Optional[str], default: None ) – 

Optional query for natural language search.

refresh_ttl (Optional[bool], default: None ) – 

Whether to refresh the TTL on items returned by this search. If None, uses the store's default behavior.

Returns:

SearchItemsResponse – 

List[Item]: A list of items matching the search criteria.

Example Usage:

items = await client.store.search_items(
    ["documents"],
    filter={"author": "John Doe"},
    limit=5,
    offset=0
)
print(items)

----------------------------------------------------------------

{
    "items": [
        {
            "namespace": ["documents", "user123"],
            "key": "item789",
            "value": {
                "title": "Another Document",
                "author": "John Doe"
            },
            "created_at": "2024-07-30T12:00:00Z",
            "updated_at": "2024-07-30T12:00:00Z"
        },
        # ... additional items ...
    ]
}

 list_namespaces(prefix: Optional[List[str]] = None, suffix: Optional[List[str]] = None, max_depth: Optional[int] = None, limit: int = 100, offset: int = 0) -> ListNamespaceResponse async ¶

List namespaces with optional match conditions.

Parameters:

prefix (Optional[List[str]], default: None ) – 

Optional list of strings representing the prefix to filter namespaces.

suffix (Optional[List[str]], default: None ) – 

Optional list of strings representing the suffix to filter namespaces.

max_depth (Optional[int], default: None ) – 

Optional integer specifying the maximum depth of namespaces to return.

limit (int, default: 100 ) – 

Maximum number of namespaces to return (default is 100).

offset (int, default: 0 ) – 

Number of namespaces to skip before returning results (default is 0).

Returns:

ListNamespaceResponse – 

List[List[str]]: A list of namespaces matching the criteria.

Example Usage:

namespaces = await client.store.list_namespaces(
    prefix=["documents"],
    max_depth=3,
    limit=10,
    offset=0
)
print(namespaces)

----------------------------------------------------------------

[
    ["documents", "user123", "reports"],
    ["documents", "user456", "invoices"],
    ...
]

 SyncLangGraphClient ¶

Synchronous client for interacting with the LangGraph API.

This class provides synchronous access to LangGraph API endpoints for managing assistants, threads, runs, cron jobs, and data storage.

Example:

client = get_sync_client()
assistant = client.assistants.get("asst_123")

 SyncHttpClient ¶
 get(path: str, *, params: Optional[QueryParamTypes] = None) -> Any ¶

Send a GET request.

 post(path: str, *, json: Optional[dict]) -> Any ¶

Send a POST request.

 put(path: str, *, json: dict) -> Any ¶

Send a PUT request.

 patch(path: str, *, json: dict) -> Any ¶

Send a PATCH request.

 delete(path: str, *, json: Optional[Any] = None) -> None ¶

Send a DELETE request.

 stream(path: str, method: str, *, json: Optional[dict] = None, params: Optional[QueryParamTypes] = None) -> Iterator[StreamPart] ¶

Stream the results of a request using SSE.

 SyncAssistantsClient ¶

Client for managing assistants in LangGraph synchronously.

This class provides methods to interact with assistants, which are versioned configurations of your graph.

Example:

client = get_client()
assistant = client.assistants.get("assistant_id_123")

 get(assistant_id: str) -> Assistant ¶

Get an assistant by ID.

Parameters:

assistant_id (str) – 

The ID of the assistant to get.

Returns:

Assistant ( Assistant ) – 

Assistant Object.

Example Usage:

assistant = client.assistants.get(
    assistant_id="my_assistant_id"
)
print(assistant)

----------------------------------------------------

{
    'assistant_id': 'my_assistant_id',
    'graph_id': 'agent',
    'created_at': '2024-06-25T17:10:33.109781+00:00',
    'updated_at': '2024-06-25T17:10:33.109781+00:00',
    'config': {},
    'metadata': {'created_by': 'system'}
}

 get_graph(assistant_id: str, *, xray: Union[int, bool] = False) -> dict[str, list[dict[str, Any]]] ¶

Get the graph of an assistant by ID.

Parameters:

assistant_id (str) – 

The ID of the assistant to get the graph of.

xray (Union[int, bool], default: False ) – 

Include graph representation of subgraphs. If an integer value is provided, only subgraphs with a depth less than or equal to the value will be included.

Returns:

Graph ( dict[str, list[dict[str, Any]]] ) – 

The graph information for the assistant in JSON format.

Example Usage:

graph_info = client.assistants.get_graph(
    assistant_id="my_assistant_id"
)
print(graph_info)

--------------------------------------------------------------------------------------------------------------------------

{
    'nodes':
        [
            {'id': '__start__', 'type': 'schema', 'data': '__start__'},
            {'id': '__end__', 'type': 'schema', 'data': '__end__'},
            {'id': 'agent','type': 'runnable','data': {'id': ['langgraph', 'utils', 'RunnableCallable'],'name': 'agent'}},
        ],
    'edges':
        [
            {'source': '__start__', 'target': 'agent'},
            {'source': 'agent','target': '__end__'}
        ]
}

 get_schemas(assistant_id: str) -> GraphSchema ¶

Get the schemas of an assistant by ID.

Parameters:

assistant_id (str) – 

The ID of the assistant to get the schema of.

Returns:

GraphSchema ( GraphSchema ) – 

The graph schema for the assistant.

Example Usage:

schema = client.assistants.get_schemas(
    assistant_id="my_assistant_id"
)
print(schema)

----------------------------------------------------------------------------------------------------------------------------

{
    'graph_id': 'agent',
    'state_schema':
        {
            'title': 'LangGraphInput',
            '$ref': '#/definitions/AgentState',
            'definitions':
                {
                    'BaseMessage':
                        {
                            'title': 'BaseMessage',
                            'description': 'Base abstract Message class. Messages are the inputs and outputs of ChatModels.',
                            'type': 'object',
                            'properties':
                                {
                                 'content':
                                    {
                                        'title': 'Content',
                                        'anyOf': [
                                            {'type': 'string'},
                                            {'type': 'array','items': {'anyOf': [{'type': 'string'}, {'type': 'object'}]}}
                                        ]
                                    },
                                'additional_kwargs':
                                    {
                                        'title': 'Additional Kwargs',
                                        'type': 'object'
                                    },
                                'response_metadata':
                                    {
                                        'title': 'Response Metadata',
                                        'type': 'object'
                                    },
                                'type':
                                    {
                                        'title': 'Type',
                                        'type': 'string'
                                    },
                                'name':
                                    {
                                        'title': 'Name',
                                        'type': 'string'
                                    },
                                'id':
                                    {
                                        'title': 'Id',
                                        'type': 'string'
                                    }
                                },
                            'required': ['content', 'type']
                        },
                    'AgentState':
                        {
                            'title': 'AgentState',
                            'type': 'object',
                            'properties':
                                {
                                    'messages':
                                        {
                                            'title': 'Messages',
                                            'type': 'array',
                                            'items': {'$ref': '#/definitions/BaseMessage'}
                                        }
                                },
                            'required': ['messages']
                        }
                }
        },
    'config_schema':
        {
            'title': 'Configurable',
            'type': 'object',
            'properties':
                {
                    'model_name':
                        {
                            'title': 'Model Name',
                            'enum': ['anthropic', 'openai'],
                            'type': 'string'
                        }
                }
        }
}

 get_subgraphs(assistant_id: str, namespace: Optional[str] = None, recurse: bool = False) -> Subgraphs ¶

Get the schemas of an assistant by ID.

Parameters:

assistant_id (str) – 

The ID of the assistant to get the schema of.

Returns:

Subgraphs ( Subgraphs ) – 

The graph schema for the assistant.

 create(graph_id: Optional[str], config: Optional[Config] = None, *, metadata: Json = None, assistant_id: Optional[str] = None, if_exists: Optional[OnConflictBehavior] = None, name: Optional[str] = None) -> Assistant ¶

Create a new assistant.

Useful when graph is configurable and you want to create different assistants based on different configurations.

Parameters:

graph_id (Optional[str]) – 

The ID of the graph the assistant should use. The graph ID is normally set in your langgraph.json configuration.

config (Optional[Config], default: None ) – 

Configuration to use for the graph.

metadata (Json, default: None ) – 

Metadata to add to assistant.

assistant_id (Optional[str], default: None ) – 

Assistant ID to use, will default to a random UUID if not provided.

if_exists (Optional[OnConflictBehavior], default: None ) – 

How to handle duplicate creation. Defaults to 'raise' under the hood. Must be either 'raise' (raise error if duplicate), or 'do_nothing' (return existing assistant).

name (Optional[str], default: None ) – 

The name of the assistant. Defaults to 'Untitled' under the hood.

Returns:

Assistant ( Assistant ) – 

The created assistant.

Example Usage:

assistant = client.assistants.create(
    graph_id="agent",
    config={"configurable": {"model_name": "openai"}},
    metadata={"number":1},
    assistant_id="my-assistant-id",
    if_exists="do_nothing",
    name="my_name"
)

 update(assistant_id: str, *, graph_id: Optional[str] = None, config: Optional[Config] = None, metadata: Json = None, name: Optional[str] = None) -> Assistant ¶

Update an assistant.

Use this to point to a different graph, update the configuration, or change the metadata of an assistant.

Parameters:

assistant_id (str) – 

Assistant to update.

graph_id (Optional[str], default: None ) – 

The ID of the graph the assistant should use. The graph ID is normally set in your langgraph.json configuration. If None, assistant will keep pointing to same graph.

config (Optional[Config], default: None ) – 

Configuration to use for the graph.

metadata (Json, default: None ) – 

Metadata to merge with existing assistant metadata.

name (Optional[str], default: None ) – 

The new name for the assistant.

Returns:

Assistant ( Assistant ) – 

The updated assistant.

Example Usage:

assistant = client.assistants.update(
    assistant_id='e280dad7-8618-443f-87f1-8e41841c180f',
    graph_id="other-graph",
    config={"configurable": {"model_name": "anthropic"}},
    metadata={"number":2}
)

 delete(assistant_id: str) -> None ¶

Delete an assistant.

Parameters:

assistant_id (str) – 

The assistant ID to delete.

Returns:

None – 

None

Example Usage:

client.assistants.delete(
    assistant_id="my_assistant_id"
)

 search(*, metadata: Json = None, graph_id: Optional[str] = None, limit: int = 10, offset: int = 0) -> list[Assistant] ¶

Search for assistants.

Parameters:

metadata (Json, default: None ) – 

Metadata to filter by. Exact match filter for each KV pair.

graph_id (Optional[str], default: None ) – 

The ID of the graph to filter by. The graph ID is normally set in your langgraph.json configuration.

limit (int, default: 10 ) – 

The maximum number of results to return.

offset (int, default: 0 ) – 

The number of results to skip.

Returns:

list[Assistant] – 

list[Assistant]: A list of assistants.

Example Usage:

assistants = client.assistants.search(
    metadata = {"name":"my_name"},
    graph_id="my_graph_id",
    limit=5,
    offset=5
)

 get_versions(assistant_id: str, metadata: Json = None, limit: int = 10, offset: int = 0) -> list[AssistantVersion] ¶

List all versions of an assistant.

Parameters:

assistant_id (str) – 

The assistant ID to get versions for.

metadata (Json, default: None ) – 

Metadata to filter versions by. Exact match filter for each KV pair.

limit (int, default: 10 ) – 

The maximum number of versions to return.

offset (int, default: 0 ) – 

The number of versions to skip.

Returns:

list[AssistantVersion] – 

list[Assistant]: A list of assistants.

Example Usage:

assistant_versions = await client.assistants.get_versions(
    assistant_id="my_assistant_id"
)

 set_latest(assistant_id: str, version: int) -> Assistant ¶

Change the version of an assistant.

Parameters:

assistant_id (str) – 

The assistant ID to delete.

version (int) – 

The version to change to.

Returns:

Assistant ( Assistant ) – 

Assistant Object.

Example Usage:

new_version_assistant = await client.assistants.set_latest(
    assistant_id="my_assistant_id",
    version=3
)

 SyncThreadsClient ¶

Synchronous client for managing threads in LangGraph.

This class provides methods to create, retrieve, and manage threads, which represent conversations or stateful interactions.

Example:

client = get_sync_client()
thread = client.threads.create(metadata={"user_id": "123"})

 get(thread_id: str) -> Thread ¶

Get a thread by ID.

Parameters:

thread_id (str) – 

The ID of the thread to get.

Returns:

Thread ( Thread ) – 

Thread object.

Example Usage:

thread = client.threads.get(
    thread_id="my_thread_id"
)
print(thread)

-----------------------------------------------------

{
    'thread_id': 'my_thread_id',
    'created_at': '2024-07-18T18:35:15.540834+00:00',
    'updated_at': '2024-07-18T18:35:15.540834+00:00',
    'metadata': {'graph_id': 'agent'}
}

 create(*, metadata: Json = None, thread_id: Optional[str] = None, if_exists: Optional[OnConflictBehavior] = None) -> Thread ¶

Create a new thread.

Parameters:

metadata (Json, default: None ) – 

Metadata to add to thread.

thread_id (Optional[str], default: None ) – 

ID of thread. If None, ID will be a randomly generated UUID.

if_exists (Optional[OnConflictBehavior], default: None ) – 

How to handle duplicate creation. Defaults to 'raise' under the hood. Must be either 'raise' (raise error if duplicate), or 'do_nothing' (return existing thread).

Returns:

Thread ( Thread ) – 

The created thread.

Example Usage:

thread = client.threads.create(
    metadata={"number":1},
    thread_id="my-thread-id",
    if_exists="raise"
)

 update(thread_id: str, *, metadata: dict[str, Any]) -> Thread ¶

Update a thread.

Parameters:

thread_id (str) – 

ID of thread to update.

metadata (dict[str, Any]) – 

Metadata to merge with existing thread metadata.

Returns:

Thread ( Thread ) – 

The created thread.

Example Usage:

thread = client.threads.update(
    thread_id="my-thread-id",
    metadata={"number":1},
)

 delete(thread_id: str) -> None ¶

Delete a thread.

Parameters:

thread_id (str) – 

The ID of the thread to delete.

Returns:

None – 

None

Example Usage:

client.threads.delete(
    thread_id="my_thread_id"
)

 search(*, metadata: Json = None, values: Json = None, status: Optional[ThreadStatus] = None, limit: int = 10, offset: int = 0) -> list[Thread] ¶

Search for threads.

Parameters:

metadata (Json, default: None ) – 

Thread metadata to filter on.

values (Json, default: None ) – 

State values to filter on.

status (Optional[ThreadStatus], default: None ) – 

Thread status to filter on. Must be one of 'idle', 'busy', 'interrupted' or 'error'.

limit (int, default: 10 ) – 

Limit on number of threads to return.

offset (int, default: 0 ) – 

Offset in threads table to start search from.

Returns:

list[Thread] – 

list[Thread]: List of the threads matching the search parameters.

Example Usage:

threads = client.threads.search(
    metadata={"number":1},
    status="interrupted",
    limit=15,
    offset=5
)

 copy(thread_id: str) -> None ¶

Copy a thread.

Parameters:

thread_id (str) – 

The ID of the thread to copy.

Returns:

None – 

None

Example Usage:

client.threads.copy(
    thread_id="my_thread_id"
)

 get_state(thread_id: str, checkpoint: Optional[Checkpoint] = None, checkpoint_id: Optional[str] = None, *, subgraphs: bool = False) -> ThreadState ¶

Get the state of a thread.

Parameters:

thread_id (str) – 

The ID of the thread to get the state of.

checkpoint (Optional[Checkpoint], default: None ) – 

The checkpoint to get the state of.

subgraphs (bool, default: False ) – 

Include subgraphs states.

Returns:

ThreadState ( ThreadState ) – 

the thread of the state.

Example Usage:

thread_state = client.threads.get_state(
    thread_id="my_thread_id",
    checkpoint_id="my_checkpoint_id"
)
print(thread_state)

----------------------------------------------------------------------------------------------------------------------------------------------------------------------

{
    'values': {
        'messages': [
            {
                'content': 'how are you?',
                'additional_kwargs': {},
                'response_metadata': {},
                'type': 'human',
                'name': None,
                'id': 'fe0a5778-cfe9-42ee-b807-0adaa1873c10',
                'example': False
            },
            {
                'content': "I'm doing well, thanks for asking! I'm an AI assistant created by Anthropic to be helpful, honest, and harmless.",
                'additional_kwargs': {},
                'response_metadata': {},
                'type': 'ai',
                'name': None,
                'id': 'run-159b782c-b679-4830-83c6-cef87798fe8b',
                'example': False,
                'tool_calls': [],
                'invalid_tool_calls': [],
                'usage_metadata': None
            }
        ]
    },
    'next': [],
    'checkpoint':
        {
            'thread_id': 'e2496803-ecd5-4e0c-a779-3226296181c2',
            'checkpoint_ns': '',
            'checkpoint_id': '1ef4a9b8-e6fb-67b1-8001-abd5184439d1'
        }
    'metadata':
        {
            'step': 1,
            'run_id': '1ef4a9b8-d7da-679a-a45a-872054341df2',
            'source': 'loop',
            'writes':
                {
                    'agent':
                        {
                            'messages': [
                                {
                                    'id': 'run-159b782c-b679-4830-83c6-cef87798fe8b',
                                    'name': None,
                                    'type': 'ai',
                                    'content': "I'm doing well, thanks for asking! I'm an AI assistant created by Anthropic to be helpful, honest, and harmless.",
                                    'example': False,
                                    'tool_calls': [],
                                    'usage_metadata': None,
                                    'additional_kwargs': {},
                                    'response_metadata': {},
                                    'invalid_tool_calls': []
                                }
                            ]
                        }
                },
    'user_id': None,
    'graph_id': 'agent',
    'thread_id': 'e2496803-ecd5-4e0c-a779-3226296181c2',
    'created_by': 'system',
    'assistant_id': 'fe096781-5601-53d2-b2f6-0d3403f7e9ca'},
    'created_at': '2024-07-25T15:35:44.184703+00:00',
    'parent_config':
        {
            'thread_id': 'e2496803-ecd5-4e0c-a779-3226296181c2',
            'checkpoint_ns': '',
            'checkpoint_id': '1ef4a9b8-d80d-6fa7-8000-9300467fad0f'
        }
}

 update_state(thread_id: str, values: Optional[Union[dict, Sequence[dict]]], *, as_node: Optional[str] = None, checkpoint: Optional[Checkpoint] = None, checkpoint_id: Optional[str] = None) -> ThreadUpdateStateResponse ¶

Update the state of a thread.

Parameters:

thread_id (str) – 

The ID of the thread to update.

values (Optional[Union[dict, Sequence[dict]]]) – 

The values to update the state with.

as_node (Optional[str], default: None ) – 

Update the state as if this node had just executed.

checkpoint (Optional[Checkpoint], default: None ) – 

The checkpoint to update the state of.

Returns:

ThreadUpdateStateResponse ( ThreadUpdateStateResponse ) – 

Response after updating a thread's state.

Example Usage:

response = client.threads.update_state(
    thread_id="my_thread_id",
    values={"messages":[{"role": "user", "content": "hello!"}]},
    as_node="my_node",
)
print(response)

----------------------------------------------------------------------------------------------------------------------------------------------------------------------

{
    'checkpoint': {
        'thread_id': 'e2496803-ecd5-4e0c-a779-3226296181c2',
        'checkpoint_ns': '',
        'checkpoint_id': '1ef4a9b8-e6fb-67b1-8001-abd5184439d1',
        'checkpoint_map': {}
    }
}

 get_history(thread_id: str, *, limit: int = 10, before: Optional[str | Checkpoint] = None, metadata: Optional[dict] = None, checkpoint: Optional[Checkpoint] = None) -> list[ThreadState] ¶

Get the state history of a thread.

Parameters:

thread_id (str) – 

The ID of the thread to get the state history for.

checkpoint (Optional[Checkpoint], default: None ) – 

Return states for this subgraph. If empty defaults to root.

limit (int, default: 10 ) – 

The maximum number of states to return.

before (Optional[str | Checkpoint], default: None ) – 

Return states before this checkpoint.

metadata (Optional[dict], default: None ) – 

Filter states by metadata key-value pairs.

Returns:

list[ThreadState] – 

list[ThreadState]: the state history of the thread.

Example Usage:

thread_state = client.threads.get_history(
    thread_id="my_thread_id",
    limit=5,
    before="my_timestamp",
    metadata={"name":"my_name"}
)

 SyncRunsClient ¶

Synchronous client for managing runs in LangGraph.

This class provides methods to create, retrieve, and manage runs, which represent individual executions of graphs.

Example:

client = get_sync_client()
run = client.runs.create(thread_id="thread_123", assistant_id="asst_456")

 stream(thread_id: Optional[str], assistant_id: str, *, input: Optional[dict] = None, command: Optional[Command] = None, stream_mode: Union[StreamMode, Sequence[StreamMode]] = 'values', stream_subgraphs: bool = False, metadata: Optional[dict] = None, config: Optional[Config] = None, checkpoint: Optional[Checkpoint] = None, checkpoint_id: Optional[str] = None, interrupt_before: Optional[Union[All, Sequence[str]]] = None, interrupt_after: Optional[Union[All, Sequence[str]]] = None, feedback_keys: Optional[Sequence[str]] = None, on_disconnect: Optional[DisconnectMode] = None, on_completion: Optional[OnCompletionBehavior] = None, webhook: Optional[str] = None, multitask_strategy: Optional[MultitaskStrategy] = None, if_not_exists: Optional[IfNotExists] = None, after_seconds: Optional[int] = None) -> Iterator[StreamPart] ¶

Create a run and stream the results.

Parameters:

thread_id (Optional[str]) – 

the thread ID to assign to the thread. If None will create a stateless run.

assistant_id (str) – 

The assistant ID or graph name to stream from. If using graph name, will default to first assistant created from that graph.

input (Optional[dict], default: None ) – 

The input to the graph.

command (Optional[Command], default: None ) – 

The command to execute.

stream_mode (Union[StreamMode, Sequence[StreamMode]], default: 'values' ) – 

The stream mode(s) to use.

stream_subgraphs (bool, default: False ) – 

Whether to stream output from subgraphs.

metadata (Optional[dict], default: None ) – 

Metadata to assign to the run.

config (Optional[Config], default: None ) – 

The configuration for the assistant.

checkpoint (Optional[Checkpoint], default: None ) – 

The checkpoint to resume from.

interrupt_before (Optional[Union[All, Sequence[str]]], default: None ) – 

Nodes to interrupt immediately before they get executed.

interrupt_after (Optional[Union[All, Sequence[str]]], default: None ) – 

Nodes to Nodes to interrupt immediately after they get executed.

feedback_keys (Optional[Sequence[str]], default: None ) – 

Feedback keys to assign to run.

on_disconnect (Optional[DisconnectMode], default: None ) – 

The disconnect mode to use. Must be one of 'cancel' or 'continue'.

on_completion (Optional[OnCompletionBehavior], default: None ) – 

Whether to delete or keep the thread created for a stateless run. Must be one of 'delete' or 'keep'.

webhook (Optional[str], default: None ) – 

Webhook to call after LangGraph API call is done.

multitask_strategy (Optional[MultitaskStrategy], default: None ) – 

Multitask strategy to use. Must be one of 'reject', 'interrupt', 'rollback', or 'enqueue'.

if_not_exists (Optional[IfNotExists], default: None ) – 

How to handle missing thread. Defaults to 'reject'. Must be either 'reject' (raise error if missing), or 'create' (create new thread).

after_seconds (Optional[int], default: None ) – 

The number of seconds to wait before starting the run. Use to schedule future runs.

Returns:

Iterator[StreamPart] – 

Iterator[StreamPart]: Iterator of stream results.

Example Usage:

async for chunk in client.runs.stream(
    thread_id=None,
    assistant_id="agent",
    input={"messages": [{"role": "user", "content": "how are you?"}]},
    stream_mode=["values","debug"],
    metadata={"name":"my_run"},
    config={"configurable": {"model_name": "anthropic"}},
    interrupt_before=["node_to_stop_before_1","node_to_stop_before_2"],
    interrupt_after=["node_to_stop_after_1","node_to_stop_after_2"],
    feedback_keys=["my_feedback_key_1","my_feedback_key_2"],
    webhook="https://my.fake.webhook.com",
    multitask_strategy="interrupt"
):
    print(chunk)

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

StreamPart(event='metadata', data={'run_id': '1ef4a9b8-d7da-679a-a45a-872054341df2'})
StreamPart(event='values', data={'messages': [{'content': 'how are you?', 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'human', 'name': None, 'id': 'fe0a5778-cfe9-42ee-b807-0adaa1873c10', 'example': False}]})
StreamPart(event='values', data={'messages': [{'content': 'how are you?', 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'human', 'name': None, 'id': 'fe0a5778-cfe9-42ee-b807-0adaa1873c10', 'example': False}, {'content': "I'm doing well, thanks for asking! I'm an AI assistant created by Anthropic to be helpful, honest, and harmless.", 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'ai', 'name': None, 'id': 'run-159b782c-b679-4830-83c6-cef87798fe8b', 'example': False, 'tool_calls': [], 'invalid_tool_calls': [], 'usage_metadata': None}]})
StreamPart(event='end', data=None)

 create(thread_id: Optional[str], assistant_id: str, *, input: Optional[dict] = None, command: Optional[Command] = None, stream_mode: Union[StreamMode, Sequence[StreamMode]] = 'values', stream_subgraphs: bool = False, metadata: Optional[dict] = None, config: Optional[Config] = None, checkpoint: Optional[Checkpoint] = None, checkpoint_id: Optional[str] = None, interrupt_before: Optional[Union[All, Sequence[str]]] = None, interrupt_after: Optional[Union[All, Sequence[str]]] = None, webhook: Optional[str] = None, multitask_strategy: Optional[MultitaskStrategy] = None, on_completion: Optional[OnCompletionBehavior] = None, if_not_exists: Optional[IfNotExists] = None, after_seconds: Optional[int] = None) -> Run ¶

Create a background run.

Parameters:

thread_id (Optional[str]) – 

the thread ID to assign to the thread. If None will create a stateless run.

assistant_id (str) – 

The assistant ID or graph name to stream from. If using graph name, will default to first assistant created from that graph.

input (Optional[dict], default: None ) – 

The input to the graph.

command (Optional[Command], default: None ) – 

The command to execute.

stream_mode (Union[StreamMode, Sequence[StreamMode]], default: 'values' ) – 

The stream mode(s) to use.

stream_subgraphs (bool, default: False ) – 

Whether to stream output from subgraphs.

metadata (Optional[dict], default: None ) – 

Metadata to assign to the run.

config (Optional[Config], default: None ) – 

The configuration for the assistant.

checkpoint (Optional[Checkpoint], default: None ) – 

The checkpoint to resume from.

interrupt_before (Optional[Union[All, Sequence[str]]], default: None ) – 

Nodes to interrupt immediately before they get executed.

interrupt_after (Optional[Union[All, Sequence[str]]], default: None ) – 

Nodes to Nodes to interrupt immediately after they get executed.

webhook (Optional[str], default: None ) – 

Webhook to call after LangGraph API call is done.

multitask_strategy (Optional[MultitaskStrategy], default: None ) – 

Multitask strategy to use. Must be one of 'reject', 'interrupt', 'rollback', or 'enqueue'.

on_completion (Optional[OnCompletionBehavior], default: None ) – 

Whether to delete or keep the thread created for a stateless run. Must be one of 'delete' or 'keep'.

if_not_exists (Optional[IfNotExists], default: None ) – 

How to handle missing thread. Defaults to 'reject'. Must be either 'reject' (raise error if missing), or 'create' (create new thread).

after_seconds (Optional[int], default: None ) – 

The number of seconds to wait before starting the run. Use to schedule future runs.

Returns:

Run ( Run ) – 

The created background run.

Example Usage:

background_run = client.runs.create(
    thread_id="my_thread_id",
    assistant_id="my_assistant_id",
    input={"messages": [{"role": "user", "content": "hello!"}]},
    metadata={"name":"my_run"},
    config={"configurable": {"model_name": "openai"}},
    interrupt_before=["node_to_stop_before_1","node_to_stop_before_2"],
    interrupt_after=["node_to_stop_after_1","node_to_stop_after_2"],
    webhook="https://my.fake.webhook.com",
    multitask_strategy="interrupt"
)
print(background_run)

--------------------------------------------------------------------------------

{
    'run_id': 'my_run_id',
    'thread_id': 'my_thread_id',
    'assistant_id': 'my_assistant_id',
    'created_at': '2024-07-25T15:35:42.598503+00:00',
    'updated_at': '2024-07-25T15:35:42.598503+00:00',
    'metadata': {},
    'status': 'pending',
    'kwargs':
        {
            'input':
                {
                    'messages': [
                        {
                            'role': 'user',
                            'content': 'how are you?'
                        }
                    ]
                },
            'config':
                {
                    'metadata':
                        {
                            'created_by': 'system'
                        },
                    'configurable':
                        {
                            'run_id': 'my_run_id',
                            'user_id': None,
                            'graph_id': 'agent',
                            'thread_id': 'my_thread_id',
                            'checkpoint_id': None,
                            'model_name': "openai",
                            'assistant_id': 'my_assistant_id'
                        }
                },
            'webhook': "https://my.fake.webhook.com",
            'temporary': False,
            'stream_mode': ['values'],
            'feedback_keys': None,
            'interrupt_after': ["node_to_stop_after_1","node_to_stop_after_2"],
            'interrupt_before': ["node_to_stop_before_1","node_to_stop_before_2"]
        },
    'multitask_strategy': 'interrupt'
}

 create_batch(payloads: list[RunCreate]) -> list[Run] ¶

Create a batch of stateless background runs.

 wait(thread_id: Optional[str], assistant_id: str, *, input: Optional[dict] = None, command: Optional[Command] = None, metadata: Optional[dict] = None, config: Optional[Config] = None, checkpoint: Optional[Checkpoint] = None, checkpoint_id: Optional[str] = None, interrupt_before: Optional[Union[All, Sequence[str]]] = None, interrupt_after: Optional[Union[All, Sequence[str]]] = None, webhook: Optional[str] = None, on_disconnect: Optional[DisconnectMode] = None, on_completion: Optional[OnCompletionBehavior] = None, multitask_strategy: Optional[MultitaskStrategy] = None, if_not_exists: Optional[IfNotExists] = None, after_seconds: Optional[int] = None) -> Union[list[dict], dict[str, Any]] ¶

Create a run, wait until it finishes and return the final state.

Parameters:

thread_id (Optional[str]) – 

the thread ID to create the run on. If None will create a stateless run.

assistant_id (str) – 

The assistant ID or graph name to run. If using graph name, will default to first assistant created from that graph.

input (Optional[dict], default: None ) – 

The input to the graph.

command (Optional[Command], default: None ) – 

The command to execute.

metadata (Optional[dict], default: None ) – 

Metadata to assign to the run.

config (Optional[Config], default: None ) – 

The configuration for the assistant.

checkpoint (Optional[Checkpoint], default: None ) – 

The checkpoint to resume from.

interrupt_before (Optional[Union[All, Sequence[str]]], default: None ) – 

Nodes to interrupt immediately before they get executed.

interrupt_after (Optional[Union[All, Sequence[str]]], default: None ) – 

Nodes to Nodes to interrupt immediately after they get executed.

webhook (Optional[str], default: None ) – 

Webhook to call after LangGraph API call is done.

on_disconnect (Optional[DisconnectMode], default: None ) – 

The disconnect mode to use. Must be one of 'cancel' or 'continue'.

on_completion (Optional[OnCompletionBehavior], default: None ) – 

Whether to delete or keep the thread created for a stateless run. Must be one of 'delete' or 'keep'.

multitask_strategy (Optional[MultitaskStrategy], default: None ) – 

Multitask strategy to use. Must be one of 'reject', 'interrupt', 'rollback', or 'enqueue'.

if_not_exists (Optional[IfNotExists], default: None ) – 

How to handle missing thread. Defaults to 'reject'. Must be either 'reject' (raise error if missing), or 'create' (create new thread).

after_seconds (Optional[int], default: None ) – 

The number of seconds to wait before starting the run. Use to schedule future runs.

Returns:

Union[list[dict], dict[str, Any]] – 

Union[list[dict], dict[str, Any]]: The output of the run.

Example Usage:

final_state_of_run = client.runs.wait(
    thread_id=None,
    assistant_id="agent",
    input={"messages": [{"role": "user", "content": "how are you?"}]},
    metadata={"name":"my_run"},
    config={"configurable": {"model_name": "anthropic"}},
    interrupt_before=["node_to_stop_before_1","node_to_stop_before_2"],
    interrupt_after=["node_to_stop_after_1","node_to_stop_after_2"],
    webhook="https://my.fake.webhook.com",
    multitask_strategy="interrupt"
)
print(final_state_of_run)

-------------------------------------------------------------------------------------------------------------------------------------------

{
    'messages': [
        {
            'content': 'how are you?',
            'additional_kwargs': {},
            'response_metadata': {},
            'type': 'human',
            'name': None,
            'id': 'f51a862c-62fe-4866-863b-b0863e8ad78a',
            'example': False
        },
        {
            'content': "I'm doing well, thanks for asking! I'm an AI assistant created by Anthropic to be helpful, honest, and harmless.",
            'additional_kwargs': {},
            'response_metadata': {},
            'type': 'ai',
            'name': None,
            'id': 'run-bf1cd3c6-768f-4c16-b62d-ba6f17ad8b36',
            'example': False,
            'tool_calls': [],
            'invalid_tool_calls': [],
            'usage_metadata': None
        }
    ]
}

 list(thread_id: str, *, limit: int = 10, offset: int = 0) -> List[Run] ¶

List runs.

Parameters:

thread_id (str) – 

The thread ID to list runs for.

limit (int, default: 10 ) – 

The maximum number of results to return.

offset (int, default: 0 ) – 

The number of results to skip.

Returns:

List[Run] – 

List[Run]: The runs for the thread.

Example Usage:

client.runs.list(
    thread_id="thread_id",
    limit=5,
    offset=5,
)

 get(thread_id: str, run_id: str) -> Run ¶

Get a run.

Parameters:

thread_id (str) – 

The thread ID to get.

run_id (str) – 

The run ID to get.

Returns:

Run ( Run ) – 

Run object.

Example Usage:

run = client.runs.get(
    thread_id="thread_id_to_delete",
    run_id="run_id_to_delete",
)

 cancel(thread_id: str, run_id: str, *, wait: bool = False, action: CancelAction = 'interrupt') -> None ¶

Get a run.

Parameters:

thread_id (str) – 

The thread ID to cancel.

run_id (str) – 

The run ID to cancel.

wait (bool, default: False ) – 

Whether to wait until run has completed.

action (CancelAction, default: 'interrupt' ) – 

Action to take when cancelling the run. Possible values are interrupt or rollback. Default is interrupt.

Returns:

None – 

None

Example Usage:

client.runs.cancel(
    thread_id="thread_id_to_cancel",
    run_id="run_id_to_cancel",
    wait=True,
    action="interrupt"
)

 join(thread_id: str, run_id: str) -> dict ¶

Block until a run is done. Returns the final state of the thread.

Parameters:

thread_id (str) – 

The thread ID to join.

run_id (str) – 

The run ID to join.

Returns:

dict – 

None

Example Usage:

client.runs.join(
    thread_id="thread_id_to_join",
    run_id="run_id_to_join"
)

 join_stream(thread_id: str, run_id: str) -> Iterator[StreamPart] ¶

Stream output from a run in real-time, until the run is done. Output is not buffered, so any output produced before this call will not be received here.

Parameters:

thread_id (str) – 

The thread ID to join.

run_id (str) – 

The run ID to join.

Returns:

Iterator[StreamPart] – 

None

Example Usage:

client.runs.join_stream(
    thread_id="thread_id_to_join",
    run_id="run_id_to_join"
)

 delete(thread_id: str, run_id: str) -> None ¶

Delete a run.

Parameters:

thread_id (str) – 

The thread ID to delete.

run_id (str) – 

The run ID to delete.

Returns:

None – 

None

Example Usage:

client.runs.delete(
    thread_id="thread_id_to_delete",
    run_id="run_id_to_delete"
)

 SyncCronClient ¶

Synchronous client for managing cron jobs in LangGraph.

This class provides methods to create and manage scheduled tasks (cron jobs) for automated graph executions.

Example:

client = get_sync_client()
cron_job = client.crons.create_for_thread(thread_id="thread_123", assistant_id="asst_456", schedule="0 * * * *")

 create_for_thread(thread_id: str, assistant_id: str, *, schedule: str, input: Optional[dict] = None, metadata: Optional[dict] = None, config: Optional[Config] = None, interrupt_before: Optional[Union[All, list[str]]] = None, interrupt_after: Optional[Union[All, list[str]]] = None, webhook: Optional[str] = None, multitask_strategy: Optional[str] = None) -> Run ¶

Create a cron job for a thread.

Parameters:

thread_id (str) – 

the thread ID to run the cron job on.

assistant_id (str) – 

The assistant ID or graph name to use for the cron job. If using graph name, will default to first assistant created from that graph.

schedule (str) – 

The cron schedule to execute this job on.

input (Optional[dict], default: None ) – 

The input to the graph.

metadata (Optional[dict], default: None ) – 

Metadata to assign to the cron job runs.

config (Optional[Config], default: None ) – 

The configuration for the assistant.

interrupt_before (Optional[Union[All, list[str]]], default: None ) – 

Nodes to interrupt immediately before they get executed.

interrupt_after (Optional[Union[All, list[str]]], default: None ) – 

Nodes to Nodes to interrupt immediately after they get executed.

webhook (Optional[str], default: None ) – 

Webhook to call after LangGraph API call is done.

multitask_strategy (Optional[str], default: None ) – 

Multitask strategy to use. Must be one of 'reject', 'interrupt', 'rollback', or 'enqueue'.

Returns:

Run ( Run ) – 

The cron run.

Example Usage:

cron_run = client.crons.create_for_thread(
    thread_id="my-thread-id",
    assistant_id="agent",
    schedule="27 15 * * *",
    input={"messages": [{"role": "user", "content": "hello!"}]},
    metadata={"name":"my_run"},
    config={"configurable": {"model_name": "openai"}},
    interrupt_before=["node_to_stop_before_1","node_to_stop_before_2"],
    interrupt_after=["node_to_stop_after_1","node_to_stop_after_2"],
    webhook="https://my.fake.webhook.com",
    multitask_strategy="interrupt"
)

 create(assistant_id: str, *, schedule: str, input: Optional[dict] = None, metadata: Optional[dict] = None, config: Optional[Config] = None, interrupt_before: Optional[Union[All, list[str]]] = None, interrupt_after: Optional[Union[All, list[str]]] = None, webhook: Optional[str] = None, multitask_strategy: Optional[str] = None) -> Run ¶

Create a cron run.

Parameters:

assistant_id (str) – 

The assistant ID or graph name to use for the cron job. If using graph name, will default to first assistant created from that graph.

schedule (str) – 

The cron schedule to execute this job on.

input (Optional[dict], default: None ) – 

The input to the graph.

metadata (Optional[dict], default: None ) – 

Metadata to assign to the cron job runs.

config (Optional[Config], default: None ) – 

The configuration for the assistant.

interrupt_before (Optional[Union[All, list[str]]], default: None ) – 

Nodes to interrupt immediately before they get executed.

interrupt_after (Optional[Union[All, list[str]]], default: None ) – 

Nodes to Nodes to interrupt immediately after they get executed.

webhook (Optional[str], default: None ) – 

Webhook to call after LangGraph API call is done.

multitask_strategy (Optional[str], default: None ) – 

Multitask strategy to use. Must be one of 'reject', 'interrupt', 'rollback', or 'enqueue'.

Returns:

Run ( Run ) – 

The cron run.

Example Usage:

cron_run = client.crons.create(
    assistant_id="agent",
    schedule="27 15 * * *",
    input={"messages": [{"role": "user", "content": "hello!"}]},
    metadata={"name":"my_run"},
    config={"configurable": {"model_name": "openai"}},
    interrupt_before=["node_to_stop_before_1","node_to_stop_before_2"],
    interrupt_after=["node_to_stop_after_1","node_to_stop_after_2"],
    webhook="https://my.fake.webhook.com",
    multitask_strategy="interrupt"
)

 delete(cron_id: str) -> None ¶

Delete a cron.

Parameters:

cron_id (str) – 

The cron ID to delete.

Returns:

None – 

None

Example Usage:

client.crons.delete(
    cron_id="cron_to_delete"
)

 search(*, assistant_id: Optional[str] = None, thread_id: Optional[str] = None, limit: int = 10, offset: int = 0) -> list[Cron] ¶

Get a list of cron jobs.

Parameters:

assistant_id (Optional[str], default: None ) – 

The assistant ID or graph name to search for.

thread_id (Optional[str], default: None ) – 

the thread ID to search for.

limit (int, default: 10 ) – 

The maximum number of results to return.

offset (int, default: 0 ) – 

The number of results to skip.

Returns:

list[Cron] – 

list[Cron]: The list of cron jobs returned by the search,

Example Usage:

cron_jobs = client.crons.search(
    assistant_id="my_assistant_id",
    thread_id="my_thread_id",
    limit=5,
    offset=5,
)
print(cron_jobs)

----------------------------------------------------------

[
    {
        'cron_id': '1ef3cefa-4c09-6926-96d0-3dc97fd5e39b',
        'assistant_id': 'my_assistant_id',
        'thread_id': 'my_thread_id',
        'user_id': None,
        'payload':
            {
                'input': {'start_time': ''},
                'schedule': '4 * * * *',
                'assistant_id': 'my_assistant_id'
            },
        'schedule': '4 * * * *',
        'next_run_date': '2024-07-25T17:04:00+00:00',
        'end_time': None,
        'created_at': '2024-07-08T06:02:23.073257+00:00',
        'updated_at': '2024-07-08T06:02:23.073257+00:00'
    }
]

 SyncStoreClient ¶

A client for synchronous operations on a key-value store.

Provides methods to interact with a remote key-value store, allowing storage and retrieval of items within namespaced hierarchies.

Example:

client = get_sync_client()
client.store.put_item(["users", "profiles"], "user123", {"name": "Alice", "age": 30})

 put_item(namespace: Sequence[str], /, key: str, value: dict[str, Any], index: Optional[Union[Literal[False], list[str]]] = None, ttl: Optional[int] = None) -> None ¶

Store or update an item.

Parameters:

namespace (Sequence[str]) – 

A list of strings representing the namespace path.

key (str) – 

The unique identifier for the item within the namespace.

value (dict[str, Any]) – 

A dictionary containing the item's data.

index (Optional[Union[Literal[False], list[str]]], default: None ) – 

Controls search indexing - None (use defaults), False (disable), or list of field paths to index.

ttl (Optional[int], default: None ) – 

Optional time-to-live in minutes for the item, or None for no expiration.

Returns: None

Example Usage:

client.store.put_item(
    ["documents", "user123"],
    key="item456",
    value={"title": "My Document", "content": "Hello World"}
)

 get_item(namespace: Sequence[str], /, key: str, *, refresh_ttl: Optional[bool] = None) -> Item ¶

Retrieve a single item.

Parameters:

key (str) – 

The unique identifier for the item.

namespace (Sequence[str]) – 

Optional list of strings representing the namespace path.

refresh_ttl (Optional[bool], default: None ) – 

Whether to refresh the TTL on this read operation. If None, uses the store's default behavior.

Returns:

Item ( Item ) – 

The retrieved item.

Example Usage:

item = client.store.get_item(
    ["documents", "user123"],
    key="item456",
)
print(item)

----------------------------------------------------------------

{
    'namespace': ['documents', 'user123'],
    'key': 'item456',
    'value': {'title': 'My Document', 'content': 'Hello World'},
    'created_at': '2024-07-30T12:00:00Z',
    'updated_at': '2024-07-30T12:00:00Z'
}

 delete_item(namespace: Sequence[str], /, key: str) -> None ¶

Delete an item.

Parameters:

key (str) – 

The unique identifier for the item.

namespace (Sequence[str]) – 

Optional list of strings representing the namespace path.

Returns:

None – 

None

Example Usage:

client.store.delete_item(
    ["documents", "user123"],
    key="item456",
)

 search_items(namespace_prefix: Sequence[str], /, filter: Optional[dict[str, Any]] = None, limit: int = 10, offset: int = 0, query: Optional[str] = None, refresh_ttl: Optional[bool] = None) -> SearchItemsResponse ¶

Search for items within a namespace prefix.

Parameters:

namespace_prefix (Sequence[str]) – 

List of strings representing the namespace prefix.

filter (Optional[dict[str, Any]], default: None ) – 

Optional dictionary of key-value pairs to filter results.

limit (int, default: 10 ) – 

Maximum number of items to return (default is 10).

offset (int, default: 0 ) – 

Number of items to skip before returning results (default is 0).

query (Optional[str], default: None ) – 

Optional query for natural language search.

refresh_ttl (Optional[bool], default: None ) – 

Whether to refresh the TTL on items returned by this search. If None, uses the store's default behavior.

Returns:

SearchItemsResponse – 

List[Item]: A list of items matching the search criteria.

Example Usage:

items = client.store.search_items(
    ["documents"],
    filter={"author": "John Doe"},
    limit=5,
    offset=0
)
print(items)

----------------------------------------------------------------

{
    "items": [
        {
            "namespace": ["documents", "user123"],
            "key": "item789",
            "value": {
                "title": "Another Document",
                "author": "John Doe"
            },
            "created_at": "2024-07-30T12:00:00Z",
            "updated_at": "2024-07-30T12:00:00Z"
        },
        # ... additional items ...
    ]
}

 list_namespaces(prefix: Optional[List[str]] = None, suffix: Optional[List[str]] = None, max_depth: Optional[int] = None, limit: int = 100, offset: int = 0) -> ListNamespaceResponse ¶

List namespaces with optional match conditions.

Parameters:

prefix (Optional[List[str]], default: None ) – 

Optional list of strings representing the prefix to filter namespaces.

suffix (Optional[List[str]], default: None ) – 

Optional list of strings representing the suffix to filter namespaces.

max_depth (Optional[int], default: None ) – 

Optional integer specifying the maximum depth of namespaces to return.

limit (int, default: 100 ) – 

Maximum number of namespaces to return (default is 100).

offset (int, default: 0 ) – 

Number of namespaces to skip before returning results (default is 0).

Returns:

ListNamespaceResponse – 

List[List[str]]: A list of namespaces matching the criteria.

Example Usage:

namespaces = client.store.list_namespaces(
    prefix=["documents"],
    max_depth=3,
    limit=10,
    offset=0
)
print(namespaces)

----------------------------------------------------------------

[
    ["documents", "user123", "reports"],
    ["documents", "user456", "invoices"],
    ...
]

 get_headers(api_key: Optional[str], custom_headers: Optional[dict[str, str]]) -> dict[str, str] ¶

Combine api_key and custom user-provided headers.

 get_client(*, url: Optional[str] = None, api_key: Optional[str] = None, headers: Optional[dict[str, str]] = None) -> LangGraphClient ¶

Get a LangGraphClient instance.

Parameters:

url (Optional[str], default: None ) – 

The URL of the LangGraph API.

api_key (Optional[str], default: None ) – 

The API key. If not provided, it will be read from the environment. Precedence: 1. explicit argument 2. LANGGRAPH_API_KEY 3. LANGSMITH_API_KEY 4. LANGCHAIN_API_KEY

headers (Optional[dict[str, str]], default: None ) – 

Optional custom headers

Returns:

LangGraphClient ( LangGraphClient ) – 

The top-level client for accessing AssistantsClient,

LangGraphClient – 

ThreadsClient, RunsClient, and CronClient.

Example:

from langgraph_sdk import get_client

# get top-level LangGraphClient
client = get_client(url="http://localhost:8123")

# example usage: client.<model>.<method_name>()
assistants = await client.assistants.get(assistant_id="some_uuid")

 get_sync_client(*, url: Optional[str] = None, api_key: Optional[str] = None, headers: Optional[dict[str, str]] = None) -> SyncLangGraphClient ¶

Get a synchronous LangGraphClient instance.

Parameters:

url (Optional[str], default: None ) – 

The URL of the LangGraph API.

api_key (Optional[str], default: None ) – 

The API key. If not provided, it will be read from the environment. Precedence: 1. explicit argument 2. LANGGRAPH_API_KEY 3. LANGSMITH_API_KEY 4. LANGCHAIN_API_KEY

headers (Optional[dict[str, str]], default: None ) – 

Optional custom headers

Returns: SyncLangGraphClient: The top-level synchronous client for accessing AssistantsClient, ThreadsClient, RunsClient, and CronClient.

Example:

from langgraph_sdk import get_sync_client

# get top-level synchronous LangGraphClient
client = get_sync_client(url="http://localhost:8123")

# example usage: client.<model>.<method_name>()
assistant = client.assistants.get(assistant_id="some_uuid")


Data models for interacting with the LangGraph API.

 Json = Optional[dict[str, Any]] module-attribute ¶

Represents a JSON-like structure, which can be None or a dictionary with string keys and any values.

 RunStatus = Literal['pending', 'error', 'success', 'timeout', 'interrupted'] module-attribute ¶

Represents the status of a run: - "pending": The run is waiting to start. - "error": The run encountered an error and stopped. - "success": The run completed successfully. - "timeout": The run exceeded its time limit. - "interrupted": The run was manually stopped or interrupted.

 ThreadStatus = Literal['idle', 'busy', 'interrupted', 'error'] module-attribute ¶

Represents the status of a thread: - "idle": The thread is not currently processing any task. - "busy": The thread is actively processing a task. - "interrupted": The thread's execution was interrupted. - "error": An exception occurred during task processing.

 StreamMode = Literal['values', 'messages', 'updates', 'events', 'debug', 'custom', 'messages-tuple'] module-attribute ¶

Defines the mode of streaming: - "values": Stream only the values. - "messages": Stream complete messages. - "updates": Stream updates to the state. - "events": Stream events occurring during execution. - "debug": Stream detailed debug information. - "custom": Stream custom events.

 DisconnectMode = Literal['cancel', 'continue'] module-attribute ¶

Specifies behavior on disconnection: - "cancel": Cancel the operation on disconnection. - "continue": Continue the operation even if disconnected.

 MultitaskStrategy = Literal['reject', 'interrupt', 'rollback', 'enqueue'] module-attribute ¶

Defines how to handle multiple tasks: - "reject": Reject new tasks when busy. - "interrupt": Interrupt current task for new ones. - "rollback": Roll back current task and start new one. - "enqueue": Queue new tasks for later execution.

 OnConflictBehavior = Literal['raise', 'do_nothing'] module-attribute ¶

Specifies behavior on conflict: - "raise": Raise an exception when a conflict occurs. - "do_nothing": Ignore conflicts and proceed.

 OnCompletionBehavior = Literal['delete', 'keep'] module-attribute ¶

Defines action after completion: - "delete": Delete resources after completion. - "keep": Retain resources after completion.

 All = Literal['*'] module-attribute ¶

Represents a wildcard or 'all' selector.

 IfNotExists = Literal['create', 'reject'] module-attribute ¶

Specifies behavior if the thread doesn't exist: - "create": Create a new thread if it doesn't exist. - "reject": Reject the operation if the thread doesn't exist.

 CancelAction = Literal['interrupt', 'rollback'] module-attribute ¶

Action to take when cancelling the run. - "interrupt": Simply cancel the run. - "rollback": Cancel the run. Then delete the run and associated checkpoints.

 Config ¶

Bases: TypedDict

Configuration options for a call.

 tags: list[str] instance-attribute ¶

Tags for this call and any sub-calls (eg. a Chain calling an LLM). You can use these to filter calls.

 recursion_limit: int instance-attribute ¶

Maximum number of times a call can recurse. If not provided, defaults to 25.

 configurable: dict[str, Any] instance-attribute ¶

Runtime values for attributes previously made configurable on this Runnable, or sub-Runnables, through .configurable_fields() or .configurable_alternatives(). Check .output_schema() for a description of the attributes that have been made configurable.

 Checkpoint ¶

Bases: TypedDict

Represents a checkpoint in the execution process.

 thread_id: str instance-attribute ¶

Unique identifier for the thread associated with this checkpoint.

 checkpoint_ns: str instance-attribute ¶

Namespace for the checkpoint, used for organization and retrieval.

 checkpoint_id: Optional[str] instance-attribute ¶

Optional unique identifier for the checkpoint itself.

 checkpoint_map: Optional[dict[str, Any]] instance-attribute ¶

Optional dictionary containing checkpoint-specific data.

 GraphSchema ¶

Bases: TypedDict

Defines the structure and properties of a graph.

 graph_id: str instance-attribute ¶

The ID of the graph.

 input_schema: Optional[dict] instance-attribute ¶

The schema for the graph input. Missing if unable to generate JSON schema from graph.

 output_schema: Optional[dict] instance-attribute ¶

The schema for the graph output. Missing if unable to generate JSON schema from graph.

 state_schema: Optional[dict] instance-attribute ¶

The schema for the graph state. Missing if unable to generate JSON schema from graph.

 config_schema: Optional[dict] instance-attribute ¶

The schema for the graph config. Missing if unable to generate JSON schema from graph.

 AssistantBase ¶

Bases: TypedDict

Base model for an assistant.

 assistant_id: str instance-attribute ¶

The ID of the assistant.

 graph_id: str instance-attribute ¶

The ID of the graph.

 config: Config instance-attribute ¶

The assistant config.

 created_at: datetime instance-attribute ¶

The time the assistant was created.

 metadata: Json instance-attribute ¶

The assistant metadata.

 version: int instance-attribute ¶

The version of the assistant

 name: str instance-attribute ¶

The name of the assistant

 AssistantVersion ¶

Bases: AssistantBase

Represents a specific version of an assistant.

 assistant_id: str instance-attribute ¶

The ID of the assistant.

 graph_id: str instance-attribute ¶

The ID of the graph.

 config: Config instance-attribute ¶

The assistant config.

 created_at: datetime instance-attribute ¶

The time the assistant was created.

 metadata: Json instance-attribute ¶

The assistant metadata.

 version: int instance-attribute ¶

The version of the assistant

 name: str instance-attribute ¶

The name of the assistant

 Assistant ¶

Bases: AssistantBase

Represents an assistant with additional properties.

 assistant_id: str instance-attribute ¶

The ID of the assistant.

 graph_id: str instance-attribute ¶

The ID of the graph.

 config: Config instance-attribute ¶

The assistant config.

 created_at: datetime instance-attribute ¶

The time the assistant was created.

 metadata: Json instance-attribute ¶

The assistant metadata.

 version: int instance-attribute ¶

The version of the assistant

 name: str instance-attribute ¶

The name of the assistant

 updated_at: datetime instance-attribute ¶

The last time the assistant was updated.

 Interrupt ¶

Bases: TypedDict

Represents an interruption in the execution flow.

 value: Any instance-attribute ¶

The value associated with the interrupt.

 when: Literal['during'] instance-attribute ¶

When the interrupt occurred.

 resumable: bool instance-attribute ¶

Whether the interrupt can be resumed.

 ns: Optional[list[str]] instance-attribute ¶

Optional namespace for the interrupt.

 Thread ¶

Bases: TypedDict

Represents a conversation thread.

 thread_id: str instance-attribute ¶

The ID of the thread.

 created_at: datetime instance-attribute ¶

The time the thread was created.

 updated_at: datetime instance-attribute ¶

The last time the thread was updated.

 metadata: Json instance-attribute ¶

The thread metadata.

 status: ThreadStatus instance-attribute ¶

The status of the thread, one of 'idle', 'busy', 'interrupted'.

 values: Json instance-attribute ¶

The current state of the thread.

 interrupts: Dict[str, list[Interrupt]] instance-attribute ¶

Interrupts which were thrown in this thread

 ThreadTask ¶

Bases: TypedDict

Represents a task within a thread.

 ThreadState ¶

Bases: TypedDict

Represents the state of a thread.

 values: Union[list[dict], dict[str, Any]] instance-attribute ¶

The state values.

 next: Sequence[str] instance-attribute ¶

The next nodes to execute. If empty, the thread is done until new input is received.

 checkpoint: Checkpoint instance-attribute ¶

The ID of the checkpoint.

 metadata: Json instance-attribute ¶

Metadata for this state

 created_at: Optional[str] instance-attribute ¶

Timestamp of state creation

 parent_checkpoint: Optional[Checkpoint] instance-attribute ¶

The ID of the parent checkpoint. If missing, this is the root checkpoint.

 tasks: Sequence[ThreadTask] instance-attribute ¶

Tasks to execute in this step. If already attempted, may contain an error.

 ThreadUpdateStateResponse ¶

Bases: TypedDict

Represents the response from updating a thread's state.

 checkpoint: Checkpoint instance-attribute ¶

Checkpoint of the latest state.

 Run ¶

Bases: TypedDict

Represents a single execution run.

 run_id: str instance-attribute ¶

The ID of the run.

 thread_id: str instance-attribute ¶

The ID of the thread.

 assistant_id: str instance-attribute ¶

The assistant that was used for this run.

 created_at: datetime instance-attribute ¶

The time the run was created.

 updated_at: datetime instance-attribute ¶

The last time the run was updated.

 status: RunStatus instance-attribute ¶

The status of the run. One of 'pending', 'running', "error", 'success', "timeout", "interrupted".

 metadata: Json instance-attribute ¶

The run metadata.

 multitask_strategy: MultitaskStrategy instance-attribute ¶

Strategy to handle concurrent runs on the same thread.

 Cron ¶

Bases: TypedDict

Represents a scheduled task.

 cron_id: str instance-attribute ¶

The ID of the cron.

 thread_id: Optional[str] instance-attribute ¶

The ID of the thread.

 end_time: Optional[datetime] instance-attribute ¶

The end date to stop running the cron.

 schedule: str instance-attribute ¶

The schedule to run, cron format.

 created_at: datetime instance-attribute ¶

The time the cron was created.

 updated_at: datetime instance-attribute ¶

The last time the cron was updated.

 payload: dict instance-attribute ¶

The run payload to use for creating new run.

 RunCreate ¶

Bases: TypedDict

Defines the parameters for initiating a background run.

 thread_id: Optional[str] instance-attribute ¶

The identifier of the thread to run. If not provided, the run is stateless.

 assistant_id: str instance-attribute ¶

The identifier of the assistant to use for this run.

 input: Optional[dict] instance-attribute ¶

Initial input data for the run.

 metadata: Optional[dict] instance-attribute ¶

Additional metadata to associate with the run.

 config: Optional[Config] instance-attribute ¶

Configuration options for the run.

 checkpoint_id: Optional[str] instance-attribute ¶

The identifier of a checkpoint to resume from.

 interrupt_before: Optional[list[str]] instance-attribute ¶

List of node names to interrupt execution before.

 interrupt_after: Optional[list[str]] instance-attribute ¶

List of node names to interrupt execution after.

 webhook: Optional[str] instance-attribute ¶

URL to send webhook notifications about the run's progress.

 multitask_strategy: Optional[MultitaskStrategy] instance-attribute ¶

Strategy for handling concurrent runs on the same thread.

 Item ¶

Bases: TypedDict

Represents a single document or data entry in the graph's Store.

Items are used to store cross-thread memories.

 namespace: list[str] instance-attribute ¶

The namespace of the item. A namespace is analogous to a document's directory.

 key: str instance-attribute ¶

The unique identifier of the item within its namespace.

In general, keys needn't be globally unique.

 value: dict[str, Any] instance-attribute ¶

The value stored in the item. This is the document itself.

 created_at: datetime instance-attribute ¶

The timestamp when the item was created.

 updated_at: datetime instance-attribute ¶

The timestamp when the item was last updated.

 ListNamespaceResponse ¶

Bases: TypedDict

Response structure for listing namespaces.

 namespaces: list[list[str]] instance-attribute ¶

A list of namespace paths, where each path is a list of strings.

 SearchItem ¶

Bases: Item

Item with an optional relevance score from search operations.

Attributes:

score (Optional[float]) – 

Relevance/similarity score. Included when searching a compatible store with a natural language query.

 namespace: list[str] instance-attribute ¶

The namespace of the item. A namespace is analogous to a document's directory.

 key: str instance-attribute ¶

The unique identifier of the item within its namespace.

In general, keys needn't be globally unique.

 value: dict[str, Any] instance-attribute ¶

The value stored in the item. This is the document itself.

 created_at: datetime instance-attribute ¶

The timestamp when the item was created.

 updated_at: datetime instance-attribute ¶

The timestamp when the item was last updated.

 SearchItemsResponse ¶

Bases: TypedDict

Response structure for searching items.

 items: list[SearchItem] instance-attribute ¶

A list of items matching the search criteria.

 StreamPart ¶

Bases: NamedTuple

Represents a part of a stream response.

 event: str instance-attribute ¶

The type of event for this stream part.

 data: dict instance-attribute ¶

The data payload associated with the event.

 Send ¶

Bases: TypedDict

Represents a message to be sent to a specific node in the graph.

This type is used to explicitly send messages to nodes in the graph, typically used within Command objects to control graph execution flow.

 node: str instance-attribute ¶

The name of the target node to send the message to.

 input: Optional[dict[str, Any]] instance-attribute ¶

Optional dictionary containing the input data to be passed to the node.

If None, the node will be called with no input.

 Command ¶

Bases: TypedDict

Represents one or more commands to control graph execution flow and state.

This type defines the control commands that can be returned by nodes to influence graph execution. It lets you navigate to other nodes, update graph state, and resume from interruptions.

 goto: Union[Send, str, Sequence[Union[Send, str]]] instance-attribute ¶

Specifies where execution should continue. Can be:

A string node name to navigate to
A Send object to execute a node with specific input
A sequence of node names or Send objects to execute in order
 update: Union[dict[str, Any], Sequence[Tuple[str, Any]]] instance-attribute ¶

Updates to apply to the graph's state. Can be:

A dictionary of state updates to merge
A sequence of (key, value) tuples for ordered updates
 resume: Any instance-attribute ¶

Value to resume execution with after an interruption. Used in conjunction with interrupt() to implement control flow.

 Auth ¶

Add custom authentication and authorization management to your LangGraph application.

The Auth class provides a unified system for handling authentication and authorization in LangGraph applications. It supports custom user authentication protocols and fine-grained authorization rules for different resources and actions.

To use, create a separate python file and add the path to the file to your LangGraph API configuration file (langgraph.json). Within that file, create an instance of the Auth class and register authentication and authorization handlers as needed.

Example langgraph.json file:

{

  "dependencies": ["."],

  "graphs": {

    "agent": "./my_agent/agent.py:graph"

  },

  "env": ".env",

  "auth": {

    "path": "./auth.py:my_auth"

  }


Then the LangGraph server will load your auth file and run it server-side whenever a request comes in.

Basic Usage
from langgraph_sdk import Auth



my_auth = Auth()



async def verify_token(token: str) -> str:

    # Verify token and return user_id

    # This would typically be a call to your auth server

    return "user_id"



@auth.authenticate

async def authenticate(authorization: str) -> str:

    # Verify token and return user_id

    result = await verify_token(authorization)

    if result != "user_id":

        raise Auth.exceptions.HTTPException(

            status_code=401, detail="Unauthorized"

        )

    return result



# Global fallback handler

@auth.on

async def authorize_default(params: Auth.on.value):

    return False # Reject all requests (default behavior)



@auth.on.threads.create

async def authorize_thread_create(params: Auth.on.threads.create.value):

    # Allow the allowed user to create a thread

    assert params.get("metadata", {}).get("owner") == "allowed_user"



@auth.on.store

async def authorize_store(ctx: Auth.types.AuthContext, value: Auth.types.on):

    assert ctx.user.identity in value["namespace"], "Not authorized"

Request Processing Flow
Authentication (your @auth.authenticate handler) is performed first on every request
For authorization, the most specific matching handler is called:
If a handler exists for the exact resource and action, it is used (e.g., @auth.on.threads.create)
Otherwise, if a handler exists for the resource with any action, it is used (e.g., @auth.on.threads)
Finally, if no specific handlers match, the global handler is used (e.g., @auth.on)
If no global handler is set, the request is accepted

This allows you to set default behavior with a global handler while overriding specific routes as needed.

 types = types class-attribute instance-attribute ¶

Reference to auth type definitions.

Provides access to all type definitions used in the auth system, like ThreadsCreate, AssistantsRead, etc.

 exceptions = exceptions class-attribute instance-attribute ¶

Reference to auth exception definitions.

Provides access to all exception definitions used in the auth system, like HTTPException, etc.

 on = _On(self) instance-attribute ¶

Entry point for authorization handlers that control access to specific resources.

The on class provides a flexible way to define authorization rules for different resources and actions in your application. It supports three main usage patterns:

Global handlers that run for all resources and actions
Resource-specific handlers that run for all actions on a resource
Resource and action specific handlers for fine-grained control
Each handler must be an async function that accepts two parameters
ctx (AuthContext): Contains request context and authenticated user info
value: The data being authorized (type varies by endpoint)

The handler should return one of:

- None or True: Accept the request
- False: Reject with 403 error
- FilterType: Apply filtering rules to the response

Examples

Global handler for all requests:

@auth.on

async def reject_unhandled_requests(ctx: AuthContext, value: Any) -> None:

    print(f"Request to {ctx.path} by {ctx.user.identity}")

    return False


Resource-specific handler. This would take precedence over the global handler for all actions on the threads resource:

@auth.on.threads

async def check_thread_access(ctx: AuthContext, value: Any) -> bool:

    # Allow access only to threads created by the user

    return value.get("created_by") == ctx.user.identity


Resource and action specific handler:

@auth.on.threads.delete

async def prevent_thread_deletion(ctx: AuthContext, value: Any) -> bool:

    # Only admins can delete threads

    return "admin" in ctx.user.permissions


Multiple resources or actions:

@auth.on(resources=["threads", "runs"], actions=["create", "update"])

async def rate_limit_writes(ctx: AuthContext, value: Any) -> bool:

    # Implement rate limiting for write operations

    return await check_rate_limit(ctx.user.identity)


Auth for the store resource is a bit different since its structure is developer defined. You typically want to enforce user creds in the namespace. Y

@auth.on.store

async def check_store_access(ctx: AuthContext, value: Auth.types.on) -> bool:

    # Assuming you structure your store like (store.aput((user_id, application_context), key, value))

    assert value["namespace"][0] == ctx.user.identity


 authenticate(fn: AH) -> AH ¶

Register an authentication handler function.

The authentication handler is responsible for verifying credentials and returning user scopes. It can accept any of the following parameters by name:

- request (Request): The raw ASGI request object
- body (dict): The parsed request body
- path (str): The request path, e.g., "/threads/abcd-1234-abcd-1234/runs/abcd-1234-abcd-1234/stream"
- method (str): The HTTP method, e.g., "GET"
- path_params (dict[str, str]): URL path parameters, e.g., {"thread_id": "abcd-1234-abcd-1234", "run_id": "abcd-1234-abcd-1234"}
- query_params (dict[str, str]): URL query parameters, e.g., {"stream": "true"}
- headers (dict[bytes, bytes]): Request headers
- authorization (str | None): The Authorization header value (e.g., "Bearer <token>")


Parameters:

fn (Callable) – 

The authentication handler function to register. Must return a representation of the user. This could be a: - string (the user id) - dict containing {"identity": str, "permissions": list[str]} - or an object with identity and permissions properties Permissions can be optionally used by your handlers downstream.

Returns:

AH – 

The registered handler function.

Raises:

ValueError – 

If an authentication handler is already registered.

Examples

Basic token authentication:

@auth.authenticate

async def authenticate(authorization: str) -> str:

    user_id = verify_token(authorization)

    return user_id


Accept the full request context:

@auth.authenticate

async def authenticate(

    method: str,

    path: str,

    headers: dict[str, bytes]

) -> str:

    user = await verify_request(method, path, headers)

    return user


Return user name and permissions:

@auth.authenticate

async def authenticate(

    method: str,

    path: str,

    headers: dict[str, bytes]

) -> Auth.types.MinimalUserDict:

    permissions, user = await verify_request(method, path, headers)

    # Permissions could be things like ["runs:read", "runs:write", "threads:read", "threads:write"]

    return {

        "identity": user["id"],

        "permissions": permissions,

        "display_name": user["name"],

    }


Authentication and authorization types for LangGraph.

This module defines the core types used for authentication, authorization, and request handling in LangGraph. It includes user protocols, authentication contexts, and typed dictionaries for various API operations.

Note

All typing.TypedDict classes use total=False to make all fields typing.Optional by default.

 RunStatus = typing.Literal['pending', 'error', 'success', 'timeout', 'interrupted'] module-attribute ¶

Status of a run execution.

Values
pending: Run is queued or in progress
error: Run failed with an error
success: Run completed successfully
timeout: Run exceeded time limit
interrupted: Run was manually interrupted
 MultitaskStrategy = typing.Literal['reject', 'rollback', 'interrupt', 'enqueue'] module-attribute ¶

Strategy for handling multiple concurrent tasks.

Values
reject: Reject new tasks while one is in progress
rollback: Cancel current task and start new one
interrupt: Interrupt current task and start new one
enqueue: Queue new tasks to run after current one
 OnConflictBehavior = typing.Literal['raise', 'do_nothing'] module-attribute ¶

Behavior when encountering conflicts.

Values
raise: Raise an exception on conflict
do_nothing: Silently ignore conflicts
 IfNotExists = typing.Literal['create', 'reject'] module-attribute ¶

Behavior when an entity doesn't exist.

Values
create: Create the entity
reject: Reject the operation
 FilterType = typing.Union[typing.Dict[str, typing.Union[str, typing.Dict[typing.Literal['$eq', '$contains'], str]]], typing.Dict[str, str]] module-attribute ¶

Response type for authorization handlers.

Supports exact matches and operators
Exact match shorthand: {"field": "value"}
Exact match: {"field": {"$eq": "value"}}
Contains: {"field": {"$contains": "value"}}
Examples

Simple exact match filter for the resource owner:

filter = {"owner": "user-abcd123"}


Explicit version of the exact match filter:

filter = {"owner": {"$eq": "user-abcd123"}}


Containment:

filter = {"participants": {"$contains": "user-abcd123"}}


Combining filters (treated as a logical AND):

filter = {"owner": "user-abcd123", "participants": {"$contains": "user-efgh456"}}


 ThreadStatus = typing.Literal['idle', 'busy', 'interrupted', 'error'] module-attribute ¶

Status of a thread.

Values
idle: Thread is available for work
busy: Thread is currently processing
interrupted: Thread was interrupted
error: Thread encountered an error
 MetadataInput = typing.Dict[str, typing.Any] module-attribute ¶

Type for arbitrary metadata attached to entities.

Allows storing custom key-value pairs with any entity. Keys must be strings, values can be any JSON-serializable type.

Examples
metadata = {

    "created_by": "user123",

    "priority": 1,

    "tags": ["important", "urgent"]

}

 HandlerResult = typing.Union[None, bool, FilterType] module-attribute ¶

The result of a handler can be: * None | True: accept the request. * False: reject the request with a 403 error * FilterType: filter to apply

 Authenticator = Callable[..., Awaitable[typing.Union[MinimalUser, str, BaseUser, MinimalUserDict, typing.Mapping[str, typing.Any]],]] module-attribute ¶

Type for authentication functions.

An authenticator can return either: 1. A string (user_id) 2. A dict containing {"identity": str, "permissions": list[str]} 3. An object with identity and permissions properties

Permissions can be used downstream by your authorization logic to determine access permissions to different resources.

The authenticate decorator will automatically inject any of the following parameters by name if they are included in your function signature:

Parameters:

request (Request) – 

The raw ASGI request object

body (dict) – 

The parsed request body

path (str) – 

The request path

method (str) – 

The HTTP method (GET, POST, etc.)

path_params (dict[str, str] | None) – 

URL path parameters

query_params (dict[str, str] | None) – 

URL query parameters

headers (dict[str, bytes] | None) – 

Request headers

authorization (str | None) – 

The Authorization header value (e.g. "Bearer ")

Examples

Basic authentication with token:

from langgraph_sdk import Auth



auth = Auth()



@auth.authenticate

async def authenticate1(authorization: str) -> Auth.types.MinimalUserDict:

    return await get_user(authorization)


Authentication with multiple parameters:

@auth.authenticate

async def authenticate2(

    method: str,

    path: str,

    headers: dict[str, bytes]

) -> Auth.types.MinimalUserDict:

    # Custom auth logic using method, path and headers

    user = verify_request(method, path, headers)

    return user


Accepting the raw ASGI request:

MY_SECRET = "my-secret-key"

@auth.authenticate

async def get_current_user(request: Request) -> Auth.types.MinimalUserDict:

    try:

        token = (request.headers.get("authorization") or "").split(" ", 1)[1]

        payload = jwt.decode(token, MY_SECRET, algorithms=["HS256"])

    except (IndexError, InvalidTokenError):

        raise HTTPException(

            status_code=401,

            detail="Invalid token",

            headers={"WWW-Authenticate": "Bearer"},

        )



    async with httpx.AsyncClient() as client:

        response = await client.get(

            f"https://api.myauth-provider.com/auth/v1/user",

            headers={"Authorization": f"Bearer {MY_SECRET}"}

        )

        if response.status_code != 200:

            raise HTTPException(status_code=401, detail="User not found")



        user_data = response.json()

        return {

            "identity": user_data["id"],

            "display_name": user_data.get("name"),

            "permissions": user_data.get("permissions", []),

            "is_authenticated": True,

        }


 MinimalUser ¶

Bases: Protocol

User objects must at least expose the identity property.

 identity: str property ¶

The unique identifier for the user.

This could be a username, email, or any other unique identifier used to distinguish between different users in the system.

 MinimalUserDict ¶

Bases: TypedDict

The dictionary representation of a user.

 identity: typing_extensions.Required[str] instance-attribute ¶

The required unique identifier for the user.

 display_name: str instance-attribute ¶

The typing.Optional display name for the user.

 is_authenticated: bool instance-attribute ¶

Whether the user is authenticated. Defaults to True.

 permissions: Sequence[str] instance-attribute ¶

A list of permissions associated with the user.

You can use these in your @auth.on authorization logic to determine access permissions to different resources.

 BaseUser ¶

Bases: Protocol

The base ASGI user protocol

 is_authenticated: bool property ¶

Whether the user is authenticated.

 display_name: str property ¶

The display name of the user.

 identity: str property ¶

The unique identifier for the user.

 permissions: Sequence[str] property ¶

The permissions associated with the user.

 StudioUser ¶

A user object that's populated from authenticated requests from the LangGraph studio.

Note: Studio auth can be disabled in your langgraph.json config.

{

  "auth": {

    "disable_studio_auth": true

  }

}


You can use isinstance checks in your authorization handlers (@auth.on) to control access specifically for developers accessing the instance from the LangGraph Studio UI.

Examples
@auth.on

async def allow_developers(ctx: Auth.types.AuthContext, value: Any) -> None:

    if isinstance(ctx.user, Auth.types.StudioUser):

        return None

    ...

    return False

 BaseAuthContext ¶

Base class for authentication context.

Provides the fundamental authentication information needed for authorization decisions.

 permissions: Sequence[str] instance-attribute ¶

The permissions granted to the authenticated user.

 user: BaseUser instance-attribute ¶

The authenticated user.

 AuthContext ¶

Bases: BaseAuthContext

Complete authentication context with resource and action information.

Extends BaseAuthContext with specific resource and action being accessed, allowing for fine-grained access control decisions.

 permissions: Sequence[str] instance-attribute ¶

The permissions granted to the authenticated user.

 user: BaseUser instance-attribute ¶

The authenticated user.

 resource: typing.Literal['runs', 'threads', 'crons', 'assistants', 'store'] instance-attribute ¶

The resource being accessed.

 action: typing.Literal['create', 'read', 'update', 'delete', 'search', 'create_run', 'put', 'get', 'list_namespaces'] instance-attribute ¶

The action being performed on the resource.

Most resources support the following actions: - create: Create a new resource - read: Read information about a resource - update: Update an existing resource - delete: Delete a resource - search: Search for resources

The store supports the following actions: - put: Add or update a document in the store - get: Get a document from the store - list_namespaces: List the namespaces in the store

 ThreadsCreate ¶

Bases: TypedDict

Parameters for creating a new thread.

Examples
create_params = {

    "thread_id": UUID("123e4567-e89b-12d3-a456-426614174000"),

    "metadata": {"owner": "user123"},

    "if_exists": "do_nothing"

}

 thread_id: UUID instance-attribute ¶

Unique identifier for the thread.

 metadata: MetadataInput instance-attribute ¶

typing.Optional metadata to attach to the thread.

 if_exists: OnConflictBehavior instance-attribute ¶

Behavior when a thread with the same ID already exists.

 ThreadsRead ¶

Bases: TypedDict

Parameters for reading thread state or run information.

This type is used in three contexts: 1. Reading thread, thread version, or thread state information: Only thread_id is provided 2. Reading run information: Both thread_id and run_id are provided

 thread_id: UUID instance-attribute ¶

Unique identifier for the thread.

 run_id: typing.Optional[UUID] instance-attribute ¶

Run ID to filter by. Only used when reading run information within a thread.

 ThreadsUpdate ¶

Bases: TypedDict

Parameters for updating a thread or run.

Called for updates to a thread, thread version, or run cancellation.

 thread_id: UUID instance-attribute ¶

Unique identifier for the thread.

 metadata: MetadataInput instance-attribute ¶

typing.Optional metadata to update.

 action: typing.Optional[typing.Literal['interrupt', 'rollback']] instance-attribute ¶

typing.Optional action to perform on the thread.

 ThreadsDelete ¶

Bases: TypedDict

Parameters for deleting a thread.

Called for deletes to a thread, thread version, or run

 thread_id: UUID instance-attribute ¶

Unique identifier for the thread.

 run_id: typing.Optional[UUID] instance-attribute ¶

typing.Optional run ID to filter by.

 ThreadsSearch ¶

Bases: TypedDict

Parameters for searching threads.

Called for searches to threads or runs.

 metadata: MetadataInput instance-attribute ¶

typing.Optional metadata to filter by.

 values: MetadataInput instance-attribute ¶

typing.Optional values to filter by.

 status: typing.Optional[ThreadStatus] instance-attribute ¶

typing.Optional status to filter by.

 limit: int instance-attribute ¶

Maximum number of results to return.

 offset: int instance-attribute ¶

Offset for pagination.

 thread_id: typing.Optional[UUID] instance-attribute ¶

typing.Optional thread ID to filter by.

 RunsCreate ¶

Bases: TypedDict

Payload for creating a run.

Examples
create_params = {

    "assistant_id": UUID("123e4567-e89b-12d3-a456-426614174000"),

    "thread_id": UUID("123e4567-e89b-12d3-a456-426614174001"),

    "run_id": UUID("123e4567-e89b-12d3-a456-426614174002"),

    "status": "pending",

    "metadata": {"owner": "user123"},

    "prevent_insert_if_inflight": True,

    "multitask_strategy": "reject",

    "if_not_exists": "create",

    "after_seconds": 10,

    "kwargs": {"key": "value"},

    "action": "interrupt"

}

 assistant_id: typing.Optional[UUID] instance-attribute ¶

typing.Optional assistant ID to use for this run.

 thread_id: typing.Optional[UUID] instance-attribute ¶

typing.Optional thread ID to use for this run.

 run_id: typing.Optional[UUID] instance-attribute ¶

typing.Optional run ID to use for this run.

 status: typing.Optional[RunStatus] instance-attribute ¶

typing.Optional status for this run.

 metadata: MetadataInput instance-attribute ¶

typing.Optional metadata for the run.

 prevent_insert_if_inflight: bool instance-attribute ¶

Prevent inserting a new run if one is already in flight.

 multitask_strategy: MultitaskStrategy instance-attribute ¶

Multitask strategy for this run.

 if_not_exists: IfNotExists instance-attribute ¶

IfNotExists for this run.

 after_seconds: int instance-attribute ¶

Number of seconds to wait before creating the run.

 kwargs: typing.Dict[str, typing.Any] instance-attribute ¶

Keyword arguments to pass to the run.

 action: typing.Optional[typing.Literal['interrupt', 'rollback']] instance-attribute ¶

Action to take if updating an existing run.

 AssistantsCreate ¶

Bases: TypedDict

Payload for creating an assistant.

Examples
create_params = {

    "assistant_id": UUID("123e4567-e89b-12d3-a456-426614174000"),

    "graph_id": "graph123",

    "config": {"key": "value"},

    "metadata": {"owner": "user123"},

    "if_exists": "do_nothing",

    "name": "Assistant 1"

}

 assistant_id: UUID instance-attribute ¶

Unique identifier for the assistant.

 graph_id: str instance-attribute ¶

Graph ID to use for this assistant.

 config: typing.Optional[typing.Union[typing.Dict[str, typing.Any], typing.Any]] instance-attribute ¶

typing.Optional configuration for the assistant.

 metadata: MetadataInput instance-attribute ¶

typing.Optional metadata to attach to the assistant.

 if_exists: OnConflictBehavior instance-attribute ¶

Behavior when an assistant with the same ID already exists.

 name: str instance-attribute ¶

Name of the assistant.

 AssistantsRead ¶

Bases: TypedDict

Payload for reading an assistant.

Examples
read_params = {

    "assistant_id": UUID("123e4567-e89b-12d3-a456-426614174000"),

    "metadata": {"owner": "user123"}

}

 assistant_id: UUID instance-attribute ¶

Unique identifier for the assistant.

 metadata: MetadataInput instance-attribute ¶

typing.Optional metadata to filter by.

 AssistantsUpdate ¶

Bases: TypedDict

Payload for updating an assistant.

Examples
update_params = {

    "assistant_id": UUID("123e4567-e89b-12d3-a456-426614174000"),

    "graph_id": "graph123",

    "config": {"key": "value"},

    "metadata": {"owner": "user123"},

    "name": "Assistant 1",

    "version": 1

}

 assistant_id: UUID instance-attribute ¶

Unique identifier for the assistant.

 graph_id: typing.Optional[str] instance-attribute ¶

typing.Optional graph ID to update.

 config: typing.Optional[typing.Union[typing.Dict[str, typing.Any], typing.Any]] instance-attribute ¶

typing.Optional configuration to update.

 metadata: MetadataInput instance-attribute ¶

typing.Optional metadata to update.

 name: typing.Optional[str] instance-attribute ¶

typing.Optional name to update.

 version: typing.Optional[int] instance-attribute ¶

typing.Optional version to update.

 AssistantsDelete ¶

Bases: TypedDict

Payload for deleting an assistant.

Examples
delete_params = {

    "assistant_id": UUID("123e4567-e89b-12d3-a456-426614174000")

}

 assistant_id: UUID instance-attribute ¶

Unique identifier for the assistant.

 AssistantsSearch ¶

Bases: TypedDict

Payload for searching assistants.

Examples
search_params = {

    "graph_id": "graph123",

    "metadata": {"owner": "user123"},

    "limit": 10,

    "offset": 0

}

 graph_id: typing.Optional[str] instance-attribute ¶

typing.Optional graph ID to filter by.

 metadata: MetadataInput instance-attribute ¶

typing.Optional metadata to filter by.

 limit: int instance-attribute ¶

Maximum number of results to return.

 offset: int instance-attribute ¶

Offset for pagination.

 CronsCreate ¶

Bases: TypedDict

Payload for creating a cron job.

Examples
create_params = {

    "payload": {"key": "value"},

    "schedule": "0 0 * * *",

    "cron_id": UUID("123e4567-e89b-12d3-a456-426614174000"),

    "thread_id": UUID("123e4567-e89b-12d3-a456-426614174001"),

    "user_id": "user123",

    "end_time": datetime(2024, 3, 16, 10, 0, 0)

}

 payload: typing.Dict[str, typing.Any] instance-attribute ¶

Payload for the cron job.

 schedule: str instance-attribute ¶

Schedule for the cron job.

 cron_id: typing.Optional[UUID] instance-attribute ¶

typing.Optional unique identifier for the cron job.

 thread_id: typing.Optional[UUID] instance-attribute ¶

typing.Optional thread ID to use for this cron job.

 user_id: typing.Optional[str] instance-attribute ¶

typing.Optional user ID to use for this cron job.

 end_time: typing.Optional[datetime] instance-attribute ¶

typing.Optional end time for the cron job.

 CronsDelete ¶

Bases: TypedDict

Payload for deleting a cron job.

Examples
delete_params = {

    "cron_id": UUID("123e4567-e89b-12d3-a456-426614174000")

}

 cron_id: UUID instance-attribute ¶

Unique identifier for the cron job.

 CronsRead ¶

Bases: TypedDict

Payload for reading a cron job.

Examples
read_params = {

    "cron_id": UUID("123e4567-e89b-12d3-a456-426614174000")

}

 cron_id: UUID instance-attribute ¶

Unique identifier for the cron job.

 CronsUpdate ¶

Bases: TypedDict

Payload for updating a cron job.

Examples
update_params = {

    "cron_id": UUID("123e4567-e89b-12d3-a456-426614174000"),

    "payload": {"key": "value"},

    "schedule": "0 0 * * *"

}

 cron_id: UUID instance-attribute ¶

Unique identifier for the cron job.

 payload: typing.Optional[typing.Dict[str, typing.Any]] instance-attribute ¶

typing.Optional payload to update.

 schedule: typing.Optional[str] instance-attribute ¶

typing.Optional schedule to update.

 CronsSearch ¶

Bases: TypedDict

Payload for searching cron jobs.

Examples
search_params = {

    "assistant_id": UUID("123e4567-e89b-12d3-a456-426614174000"),

    "thread_id": UUID("123e4567-e89b-12d3-a456-426614174001"),

    "limit": 10,

    "offset": 0

}

 assistant_id: typing.Optional[UUID] instance-attribute ¶

typing.Optional assistant ID to filter by.

 thread_id: typing.Optional[UUID] instance-attribute ¶

typing.Optional thread ID to filter by.

 limit: int instance-attribute ¶

Maximum number of results to return.

 offset: int instance-attribute ¶

Offset for pagination.

 StoreGet ¶

Bases: TypedDict

Operation to retrieve a specific item by its namespace and key.

 namespace: tuple[str, ...] instance-attribute ¶

Hierarchical path that uniquely identifies the item's location.

 key: str instance-attribute ¶

Unique identifier for the item within its specific namespace.

 StoreSearch ¶

Bases: TypedDict

Operation to search for items within a specified namespace hierarchy.

 namespace: tuple[str, ...] instance-attribute ¶

Prefix filter for defining the search scope.

 filter: typing.Optional[dict[str, typing.Any]] instance-attribute ¶

Key-value pairs for filtering results based on exact matches or comparison operators.

 limit: int instance-attribute ¶

Maximum number of items to return in the search results.

 offset: int instance-attribute ¶

Number of matching items to skip for pagination.

 query: typing.Optional[str] instance-attribute ¶

Naturalj language search query for semantic search capabilities.

 StoreListNamespaces ¶

Bases: TypedDict

Operation to list and filter namespaces in the store.

 namespace: typing.Optional[tuple[str, ...]] instance-attribute ¶

Prefix filter namespaces.

 suffix: typing.Optional[tuple[str, ...]] instance-attribute ¶

Optional conditions for filtering namespaces.

 max_depth: typing.Optional[int] instance-attribute ¶

Maximum depth of namespace hierarchy to return.

Note

Namespaces deeper than this level will be truncated.

 limit: int instance-attribute ¶

Maximum number of namespaces to return.

 offset: int instance-attribute ¶

Number of namespaces to skip for pagination.

 StorePut ¶

Bases: TypedDict

Operation to store, update, or delete an item in the store.

 namespace: tuple[str, ...] instance-attribute ¶

Hierarchical path that identifies the location of the item.

 key: str instance-attribute ¶

Unique identifier for the item within its namespace.

 value: typing.Optional[dict[str, typing.Any]] instance-attribute ¶

The data to store, or None to mark the item for deletion.

 index: typing.Optional[typing.Union[typing.Literal[False], list[str]]] instance-attribute ¶

Optional index configuration for full-text search.

 StoreDelete ¶

Bases: TypedDict

Operation to delete an item from the store.

 namespace: tuple[str, ...] instance-attribute ¶

Hierarchical path that uniquely identifies the item's location.

 key: str instance-attribute ¶

Unique identifier for the item within its specific namespace.

 on ¶

Namespace for type definitions of different API operations.

This class organizes type definitions for create, read, update, delete, and search operations across different resources (threads, assistants, crons).

Usage
from langgraph_sdk import Auth



auth = Auth()



@auth.on

def handle_all(params: Auth.on.value):

    raise Exception("Not authorized")



@auth.on.threads.create

def handle_thread_create(params: Auth.on.threads.create.value):

    # Handle thread creation

    pass



@auth.on.assistants.search

def handle_assistant_search(params: Auth.on.assistants.search.value):

    # Handle assistant search

    pass

 threads ¶

Types for thread-related operations.

 create ¶

Type for thread creation parameters.

 create_run ¶

Type for creating or streaming a run.

 read ¶

Type for thread read parameters.

 update ¶

Type for thread update parameters.

 delete ¶

Type for thread deletion parameters.

 search ¶

Type for thread search parameters.

 assistants ¶

Types for assistant-related operations.

 create ¶

Type for assistant creation parameters.

 read ¶

Type for assistant read parameters.

 update ¶

Type for assistant update parameters.

 delete ¶

Type for assistant deletion parameters.

 search ¶

Type for assistant search parameters.

 crons ¶

Types for cron-related operations.

 create ¶

Type for cron creation parameters.

 read ¶

Type for cron read parameters.

 update ¶

Type for cron update parameters.

 delete ¶

Type for cron deletion parameters.

 search ¶

Type for cron search parameters.

 store ¶

Types for store-related operations.

 put ¶

Type for store put parameters.

 get ¶

Type for store get parameters.

 search ¶

Type for store search parameters.

 delete ¶

Type for store delete parameters.

 list_namespaces ¶

Type for store list namespaces parameters.

Exceptions used in the auth system.

 HTTPException ¶

Bases: Exception

HTTP exception that you can raise to return a specific HTTP error response.

Since this is defined in the auth module, we default to a 401 status code.

Parameters:

status_code (int, default: 401 ) – 

HTTP status code for the error. Defaults to 401 "Unauthorized".

detail (str | None, default: None ) – 

Detailed error message. If None, uses a default message based on the status code.

headers (Mapping[str, str] | None, default: None ) – 

Additional HTTP headers to include in the error response.

Example

Default:

raise HTTPException()

# HTTPException(status_code=401, detail='Unauthorized')


Add headers:

raise HTTPException(headers={"X-Custom-Header": "Custom Value"})

# HTTPException(status_code=401, detail='Unauthorized', headers={"WWW-Authenticate": "Bearer"})


Custom error:

raise HTTPException(status_code=404, detail="Not found")


Comments
