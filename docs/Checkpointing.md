# Checkpointing



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/reference/checkpoints/
- **html**: API reference
Library
Checkpointers¶
 CheckpointMetadata ¶

Bases: TypedDict

Metadata associated with a checkpoint.

 source: Literal['input', 'loop', 'update', 'fork'] instance-attribute ¶

The source of the checkpoint.

"input": The checkpoint was created from an input to invoke/stream/batch.
"loop": The checkpoint was created from inside the pregel loop.
"update": The checkpoint was created from a manual state update.
"fork": The checkpoint was created as a copy of another checkpoint.
 step: int instance-attribute ¶

The step number of the checkpoint.

-1 for the first "input" checkpoint. 0 for the first "loop" checkpoint. ... for the nth checkpoint afterwards.

 writes: dict[str, Any] instance-attribute ¶

The writes that were made between the previous checkpoint and this one.

Mapping from node name to writes emitted by that node.

 parents: dict[str, str] instance-attribute ¶

The IDs of the parent checkpoints.

Mapping from checkpoint namespace to checkpoint ID.

 Checkpoint ¶

Bases: TypedDict

State snapshot at a given point in time.

 v: int instance-attribute ¶

The version of the checkpoint format. Currently 1.

 id: str instance-attribute ¶

The ID of the checkpoint. This is both unique and monotonically increasing, so can be used for sorting checkpoints from first to last.

 ts: str instance-attribute ¶

The timestamp of the checkpoint in ISO 8601 format.

 channel_values: dict[str, Any] instance-attribute ¶

The values of the channels at the time of the checkpoint. Mapping from channel name to deserialized channel snapshot value.

 channel_versions: ChannelVersions instance-attribute ¶

The versions of the channels at the time of the checkpoint. The keys are channel names and the values are monotonically increasing version strings for each channel.

 versions_seen: dict[str, ChannelVersions] instance-attribute ¶

Map from node ID to map from channel name to version seen. This keeps track of the versions of the channels that each node has seen. Used to determine which nodes to execute next.

 pending_sends: List[SendProtocol] instance-attribute ¶

List of inputs pushed to nodes but not yet processed. Cleared by the next checkpoint.

 BaseCheckpointSaver ¶

Bases: Generic[V]

Base class for creating a graph checkpointer.

Checkpointers allow LangGraph agents to persist their state within and across multiple interactions.

Attributes:

serde (SerializerProtocol) – 

Serializer for encoding/decoding checkpoints.

Note

When creating a custom checkpoint saver, consider implementing async versions to avoid blocking the main thread.

 config_specs: list[ConfigurableFieldSpec] property ¶

Define the configuration options for the checkpoint saver.

Returns:

list[ConfigurableFieldSpec] – 

list[ConfigurableFieldSpec]: List of configuration field specs.

 get(config: RunnableConfig) -> Optional[Checkpoint] ¶

Fetch a checkpoint using the given configuration.

Parameters:

config (RunnableConfig) – 

Configuration specifying which checkpoint to retrieve.

Returns:

Optional[Checkpoint] – 

Optional[Checkpoint]: The requested checkpoint, or None if not found.

 get_tuple(config: RunnableConfig) -> Optional[CheckpointTuple] ¶

Fetch a checkpoint tuple using the given configuration.

Parameters:

config (RunnableConfig) – 

Configuration specifying which checkpoint to retrieve.

Returns:

Optional[CheckpointTuple] – 

Optional[CheckpointTuple]: The requested checkpoint tuple, or None if not found.

Raises:

NotImplementedError – 

Implement this method in your custom checkpoint saver.

 list(config: Optional[RunnableConfig], *, filter: Optional[Dict[str, Any]] = None, before: Optional[RunnableConfig] = None, limit: Optional[int] = None) -> Iterator[CheckpointTuple] ¶

List checkpoints that match the given criteria.

Parameters:

config (Optional[RunnableConfig]) – 

Base configuration for filtering checkpoints.

filter (Optional[Dict[str, Any]], default: None ) – 

Additional filtering criteria.

before (Optional[RunnableConfig], default: None ) – 

List checkpoints created before this configuration.

limit (Optional[int], default: None ) – 

Maximum number of checkpoints to return.

Returns:

Iterator[CheckpointTuple] – 

Iterator[CheckpointTuple]: Iterator of matching checkpoint tuples.

Raises:

NotImplementedError – 

Implement this method in your custom checkpoint saver.

 put(config: RunnableConfig, checkpoint: Checkpoint, metadata: CheckpointMetadata, new_versions: ChannelVersions) -> RunnableConfig ¶

Store a checkpoint with its configuration and metadata.

Parameters:

config (RunnableConfig) – 

Configuration for the checkpoint.

checkpoint (Checkpoint) – 

The checkpoint to store.

metadata (CheckpointMetadata) – 

Additional metadata for the checkpoint.

new_versions (ChannelVersions) – 

New channel versions as of this write.

Returns:

RunnableConfig ( RunnableConfig ) – 

Updated configuration after storing the checkpoint.

Raises:

NotImplementedError – 

Implement this method in your custom checkpoint saver.

 put_writes(config: RunnableConfig, writes: Sequence[Tuple[str, Any]], task_id: str, task_path: str = '') -> None ¶

Store intermediate writes linked to a checkpoint.

Parameters:

config (RunnableConfig) – 

Configuration of the related checkpoint.

writes (List[Tuple[str, Any]]) – 

List of writes to store.

task_id (str) – 

Identifier for the task creating the writes.

task_path (str, default: '' ) – 

Path of the task creating the writes.

Raises:

NotImplementedError – 

Implement this method in your custom checkpoint saver.

 aget(config: RunnableConfig) -> Optional[Checkpoint] async ¶

Asynchronously fetch a checkpoint using the given configuration.

Parameters:

config (RunnableConfig) – 

Configuration specifying which checkpoint to retrieve.

Returns:

Optional[Checkpoint] – 

Optional[Checkpoint]: The requested checkpoint, or None if not found.

 aget_tuple(config: RunnableConfig) -> Optional[CheckpointTuple] async ¶

Asynchronously fetch a checkpoint tuple using the given configuration.

Parameters:

config (RunnableConfig) – 

Configuration specifying which checkpoint to retrieve.

Returns:

Optional[CheckpointTuple] – 

Optional[CheckpointTuple]: The requested checkpoint tuple, or None if not found.

Raises:

NotImplementedError – 

Implement this method in your custom checkpoint saver.

 alist(config: Optional[RunnableConfig], *, filter: Optional[Dict[str, Any]] = None, before: Optional[RunnableConfig] = None, limit: Optional[int] = None) -> AsyncIterator[CheckpointTuple] async ¶

Asynchronously list checkpoints that match the given criteria.

Parameters:

config (Optional[RunnableConfig]) – 

Base configuration for filtering checkpoints.

filter (Optional[Dict[str, Any]], default: None ) – 

Additional filtering criteria for metadata.

before (Optional[RunnableConfig], default: None ) – 

List checkpoints created before this configuration.

limit (Optional[int], default: None ) – 

Maximum number of checkpoints to return.

Returns:

AsyncIterator[CheckpointTuple] – 

AsyncIterator[CheckpointTuple]: Async iterator of matching checkpoint tuples.

Raises:

NotImplementedError – 

Implement this method in your custom checkpoint saver.

 aput(config: RunnableConfig, checkpoint: Checkpoint, metadata: CheckpointMetadata, new_versions: ChannelVersions) -> RunnableConfig async ¶

Asynchronously store a checkpoint with its configuration and metadata.

Parameters:

config (RunnableConfig) – 

Configuration for the checkpoint.

checkpoint (Checkpoint) – 

The checkpoint to store.

metadata (CheckpointMetadata) – 

Additional metadata for the checkpoint.

new_versions (ChannelVersions) – 

New channel versions as of this write.

Returns:

RunnableConfig ( RunnableConfig ) – 

Updated configuration after storing the checkpoint.

Raises:

NotImplementedError – 

Implement this method in your custom checkpoint saver.

 aput_writes(config: RunnableConfig, writes: Sequence[Tuple[str, Any]], task_id: str, task_path: str = '') -> None async ¶

Asynchronously store intermediate writes linked to a checkpoint.

Parameters:

config (RunnableConfig) – 

Configuration of the related checkpoint.

writes (List[Tuple[str, Any]]) – 

List of writes to store.

task_id (str) – 

Identifier for the task creating the writes.

task_path (str, default: '' ) – 

Path of the task creating the writes.

Raises:

NotImplementedError – 

Implement this method in your custom checkpoint saver.

 get_next_version(current: Optional[V], channel: ChannelProtocol) -> V ¶

Generate the next version ID for a channel.

Default is to use integer versions, incrementing by 1. If you override, you can use str/int/float versions, as long as they are monotonically increasing.

Parameters:

current (Optional[V]) – 

The current version identifier (int, float, or str).

channel (BaseChannel) – 

The channel being versioned.

Returns:

V ( V ) – 

The next version identifier, which must be increasing.

 create_checkpoint(checkpoint: Checkpoint, channels: Optional[Mapping[str, ChannelProtocol]], step: int, *, id: Optional[str] = None) -> Checkpoint ¶

Create a checkpoint for the given channels.

 SerializerProtocol ¶

Bases: Protocol

Protocol for serialization and deserialization of objects.

dumps: Serialize an object to bytes.
dumps_typed: Serialize an object to a tuple (type, bytes).
loads: Deserialize an object from bytes.
loads_typed: Deserialize an object from a tuple (type, bytes).

Valid implementations include the pickle, json and orjson modules.

 JsonPlusSerializer ¶

Bases: SerializerProtocol

 InMemorySaver ¶

Bases: BaseCheckpointSaver[str], AbstractContextManager, AbstractAsyncContextManager

An in-memory checkpoint saver.

This checkpoint saver stores checkpoints in memory using a defaultdict.

Note

Only use InMemorySaver for debugging or testing purposes. For production use cases we recommend installing langgraph-checkpoint-postgres and using PostgresSaver / AsyncPostgresSaver.

Parameters:

serde (Optional[SerializerProtocol], default: None ) – 

The serializer to use for serializing and deserializing checkpoints. Defaults to None.

Examples:

    import asyncio

    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.graph import StateGraph

    builder = StateGraph(int)
    builder.add_node("add_one", lambda x: x + 1)
    builder.set_entry_point("add_one")
    builder.set_finish_point("add_one")

    memory = InMemorySaver()
    graph = builder.compile(checkpointer=memory)
    coro = graph.ainvoke(1, {"configurable": {"thread_id": "thread-1"}})
    asyncio.run(coro)  # Output: 2

 config_specs: list[ConfigurableFieldSpec] property ¶

Define the configuration options for the checkpoint saver.

Returns:

list[ConfigurableFieldSpec] – 

list[ConfigurableFieldSpec]: List of configuration field specs.

 get(config: RunnableConfig) -> Optional[Checkpoint] ¶

Fetch a checkpoint using the given configuration.

Parameters:

config (RunnableConfig) – 

Configuration specifying which checkpoint to retrieve.

Returns:

Optional[Checkpoint] – 

Optional[Checkpoint]: The requested checkpoint, or None if not found.

 aget(config: RunnableConfig) -> Optional[Checkpoint] async ¶

Asynchronously fetch a checkpoint using the given configuration.

Parameters:

config (RunnableConfig) – 

Configuration specifying which checkpoint to retrieve.

Returns:

Optional[Checkpoint] – 

Optional[Checkpoint]: The requested checkpoint, or None if not found.

 get_tuple(config: RunnableConfig) -> Optional[CheckpointTuple] ¶

Get a checkpoint tuple from the in-memory storage.

This method retrieves a checkpoint tuple from the in-memory storage based on the provided config. If the config contains a "checkpoint_id" key, the checkpoint with the matching thread ID and timestamp is retrieved. Otherwise, the latest checkpoint for the given thread ID is retrieved.

Parameters:

config (RunnableConfig) – 

The config to use for retrieving the checkpoint.

Returns:

Optional[CheckpointTuple] – 

Optional[CheckpointTuple]: The retrieved checkpoint tuple, or None if no matching checkpoint was found.

 list(config: Optional[RunnableConfig], *, filter: Optional[dict[str, Any]] = None, before: Optional[RunnableConfig] = None, limit: Optional[int] = None) -> Iterator[CheckpointTuple] ¶

List checkpoints from the in-memory storage.

This method retrieves a list of checkpoint tuples from the in-memory storage based on the provided criteria.

Parameters:

config (Optional[RunnableConfig]) – 

Base configuration for filtering checkpoints.

filter (Optional[Dict[str, Any]], default: None ) – 

Additional filtering criteria for metadata.

before (Optional[RunnableConfig], default: None ) – 

List checkpoints created before this configuration.

limit (Optional[int], default: None ) – 

Maximum number of checkpoints to return.

Yields:

CheckpointTuple – 

Iterator[CheckpointTuple]: An iterator of matching checkpoint tuples.

 put(config: RunnableConfig, checkpoint: Checkpoint, metadata: CheckpointMetadata, new_versions: ChannelVersions) -> RunnableConfig ¶

Save a checkpoint to the in-memory storage.

This method saves a checkpoint to the in-memory storage. The checkpoint is associated with the provided config.

Parameters:

config (RunnableConfig) – 

The config to associate with the checkpoint.

checkpoint (Checkpoint) – 

The checkpoint to save.

metadata (CheckpointMetadata) – 

Additional metadata to save with the checkpoint.

new_versions (dict) – 

New versions as of this write

Returns:

RunnableConfig ( RunnableConfig ) – 

The updated config containing the saved checkpoint's timestamp.

 put_writes(config: RunnableConfig, writes: Sequence[tuple[str, Any]], task_id: str, task_path: str = '') -> None ¶

Save a list of writes to the in-memory storage.

This method saves a list of writes to the in-memory storage. The writes are associated with the provided config.

Parameters:

config (RunnableConfig) – 

The config to associate with the writes.

writes (list[tuple[str, Any]]) – 

The writes to save.

task_id (str) – 

Identifier for the task creating the writes.

task_path (str, default: '' ) – 

Path of the task creating the writes.

Returns:

RunnableConfig ( None ) – 

The updated config containing the saved writes' timestamp.

 aget_tuple(config: RunnableConfig) -> Optional[CheckpointTuple] async ¶

Asynchronous version of get_tuple.

This method is an asynchronous wrapper around get_tuple that runs the synchronous method in a separate thread using asyncio.

Parameters:

config (RunnableConfig) – 

The config to use for retrieving the checkpoint.

Returns:

Optional[CheckpointTuple] – 

Optional[CheckpointTuple]: The retrieved checkpoint tuple, or None if no matching checkpoint was found.

 alist(config: Optional[RunnableConfig], *, filter: Optional[dict[str, Any]] = None, before: Optional[RunnableConfig] = None, limit: Optional[int] = None) -> AsyncIterator[CheckpointTuple] async ¶

Asynchronous version of list.

This method is an asynchronous wrapper around list that runs the synchronous method in a separate thread using asyncio.

Parameters:

config (RunnableConfig) – 

The config to use for listing the checkpoints.

Yields:

AsyncIterator[CheckpointTuple] – 

AsyncIterator[CheckpointTuple]: An asynchronous iterator of checkpoint tuples.

 aput(config: RunnableConfig, checkpoint: Checkpoint, metadata: CheckpointMetadata, new_versions: ChannelVersions) -> RunnableConfig async ¶

Asynchronous version of put.

Parameters:

config (RunnableConfig) – 

The config to associate with the checkpoint.

checkpoint (Checkpoint) – 

The checkpoint to save.

metadata (CheckpointMetadata) – 

Additional metadata to save with the checkpoint.

new_versions (dict) – 

New versions as of this write

Returns:

RunnableConfig ( RunnableConfig ) – 

The updated config containing the saved checkpoint's timestamp.

 aput_writes(config: RunnableConfig, writes: Sequence[tuple[str, Any]], task_id: str, task_path: str = '') -> None async ¶

Asynchronous version of put_writes.

This method is an asynchronous wrapper around put_writes that runs the synchronous method in a separate thread using asyncio.

Parameters:

config (RunnableConfig) – 

The config to associate with the writes.

writes (List[Tuple[str, Any]]) – 

The writes to save, each as a (channel, value) pair.

task_id (str) – 

Identifier for the task creating the writes.

task_path (str, default: '' ) – 

Path of the task creating the writes.

Returns:

None – 

None

 PersistentDict ¶

Bases: defaultdict

Persistent dictionary with an API compatible with shelve and anydbm.

The dict is kept in memory, so the dictionary operations run as fast as a regular dictionary.

Write to disk is delayed until close or sync (similar to gdbm's fast mode).

Input file format is automatically discovered. Output file format is selectable between pickle, json, and csv. All three serialization formats are backed by fast C implementations.

Adapted from https://code.activestate.com/recipes/576642-persistent-dict-with-multiple-standard-file-format/

 sync() -> None ¶

Write dict to disk

 SqliteSaver ¶

Bases: BaseCheckpointSaver[str]

A checkpoint saver that stores checkpoints in a SQLite database.

Note

This class is meant for lightweight, synchronous use cases (demos and small projects) and does not scale to multiple threads. For a similar sqlite saver with async support, consider using AsyncSqliteSaver.

Parameters:

conn (Connection) – 

The SQLite database connection.

serde (Optional[SerializerProtocol], default: None ) – 

The serializer to use for serializing and deserializing checkpoints. Defaults to JsonPlusSerializerCompat.

Examples:

>>> import sqlite3
>>> from langgraph.checkpoint.sqlite import SqliteSaver
>>> from langgraph.graph import StateGraph
>>>
>>> builder = StateGraph(int)
>>> builder.add_node("add_one", lambda x: x + 1)
>>> builder.set_entry_point("add_one")
>>> builder.set_finish_point("add_one")
>>> conn = sqlite3.connect("checkpoints.sqlite")
>>> memory = SqliteSaver(conn)
>>> graph = builder.compile(checkpointer=memory)
>>> config = {"configurable": {"thread_id": "1"}}
>>> graph.get_state(config)
>>> result = graph.invoke(3, config)
>>> graph.get_state(config)
StateSnapshot(values=4, next=(), config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '0c62ca34-ac19-445d-bbb0-5b4984975b2a'}}, parent_config=None)

 config_specs: list[ConfigurableFieldSpec] property ¶

Define the configuration options for the checkpoint saver.

Returns:

list[ConfigurableFieldSpec] – 

list[ConfigurableFieldSpec]: List of configuration field specs.

 get(config: RunnableConfig) -> Optional[Checkpoint] ¶

Fetch a checkpoint using the given configuration.

Parameters:

config (RunnableConfig) – 

Configuration specifying which checkpoint to retrieve.

Returns:

Optional[Checkpoint] – 

Optional[Checkpoint]: The requested checkpoint, or None if not found.

 aget(config: RunnableConfig) -> Optional[Checkpoint] async ¶

Asynchronously fetch a checkpoint using the given configuration.

Parameters:

config (RunnableConfig) – 

Configuration specifying which checkpoint to retrieve.

Returns:

Optional[Checkpoint] – 

Optional[Checkpoint]: The requested checkpoint, or None if not found.

 aput_writes(config: RunnableConfig, writes: Sequence[Tuple[str, Any]], task_id: str, task_path: str = '') -> None async ¶

Asynchronously store intermediate writes linked to a checkpoint.

Parameters:

config (RunnableConfig) – 

Configuration of the related checkpoint.

writes (List[Tuple[str, Any]]) – 

List of writes to store.

task_id (str) – 

Identifier for the task creating the writes.

task_path (str, default: '' ) – 

Path of the task creating the writes.

Raises:

NotImplementedError – 

Implement this method in your custom checkpoint saver.

 from_conn_string(conn_string: str) -> Iterator[SqliteSaver] classmethod ¶

Create a new SqliteSaver instance from a connection string.

Parameters:

conn_string (str) – 

The SQLite connection string.

Yields:

SqliteSaver ( SqliteSaver ) – 

A new SqliteSaver instance.

Examples:

In memory:

    with SqliteSaver.from_conn_string(":memory:") as memory:
        ...

To disk:

    with SqliteSaver.from_conn_string("checkpoints.sqlite") as memory:
        ...

 setup() -> None ¶

Set up the checkpoint database.

This method creates the necessary tables in the SQLite database if they don't already exist. It is called automatically when needed and should not be called directly by the user.

 cursor(transaction: bool = True) -> Iterator[sqlite3.Cursor] ¶

Get a cursor for the SQLite database.

This method returns a cursor for the SQLite database. It is used internally by the SqliteSaver and should not be called directly by the user.

Parameters:

transaction (bool, default: True ) – 

Whether to commit the transaction when the cursor is closed. Defaults to True.

Yields:

Cursor – 

sqlite3.Cursor: A cursor for the SQLite database.

 get_tuple(config: RunnableConfig) -> Optional[CheckpointTuple] ¶

Get a checkpoint tuple from the database.

This method retrieves a checkpoint tuple from the SQLite database based on the provided config. If the config contains a "checkpoint_id" key, the checkpoint with the matching thread ID and checkpoint ID is retrieved. Otherwise, the latest checkpoint for the given thread ID is retrieved.

Parameters:

config (RunnableConfig) – 

The config to use for retrieving the checkpoint.

Returns:

Optional[CheckpointTuple] – 

Optional[CheckpointTuple]: The retrieved checkpoint tuple, or None if no matching checkpoint was found.

Examples:

Basic:
>>> config = {"configurable": {"thread_id": "1"}}
>>> checkpoint_tuple = memory.get_tuple(config)
>>> print(checkpoint_tuple)
CheckpointTuple(...)

With checkpoint ID:

>>> config = {
...    "configurable": {
...        "thread_id": "1",
...        "checkpoint_ns": "",
...        "checkpoint_id": "1ef4f797-8335-6428-8001-8a1503f9b875",
...    }
... }
>>> checkpoint_tuple = memory.get_tuple(config)
>>> print(checkpoint_tuple)
CheckpointTuple(...)

 list(config: Optional[RunnableConfig], *, filter: Optional[Dict[str, Any]] = None, before: Optional[RunnableConfig] = None, limit: Optional[int] = None) -> Iterator[CheckpointTuple] ¶

List checkpoints from the database.

This method retrieves a list of checkpoint tuples from the SQLite database based on the provided config. The checkpoints are ordered by checkpoint ID in descending order (newest first).

Parameters:

config (RunnableConfig) – 

The config to use for listing the checkpoints.

filter (Optional[Dict[str, Any]], default: None ) – 

Additional filtering criteria for metadata. Defaults to None.

before (Optional[RunnableConfig], default: None ) – 

If provided, only checkpoints before the specified checkpoint ID are returned. Defaults to None.

limit (Optional[int], default: None ) – 

The maximum number of checkpoints to return. Defaults to None.

Yields:

CheckpointTuple – 

Iterator[CheckpointTuple]: An iterator of checkpoint tuples.

Examples:

>>> from langgraph.checkpoint.sqlite import SqliteSaver

>>> with SqliteSaver.from_conn_string(":memory:") as memory:

... # Run a graph, then list the checkpoints

>>>     config = {"configurable": {"thread_id": "1"}}

>>>     checkpoints = list(memory.list(config, limit=2))

>>> print(checkpoints)

[CheckpointTuple(...), CheckpointTuple(...)]

>>> config = {"configurable": {"thread_id": "1"}}

>>> before = {"configurable": {"checkpoint_id": "1ef4f797-8335-6428-8001-8a1503f9b875"}}

>>> with SqliteSaver.from_conn_string(":memory:") as memory:

... # Run a graph, then list the checkpoints

>>>     checkpoints = list(memory.list(config, before=before))

>>> print(checkpoints)

[CheckpointTuple(...), ...]

 put(config: RunnableConfig, checkpoint: Checkpoint, metadata: CheckpointMetadata, new_versions: ChannelVersions) -> RunnableConfig ¶

Save a checkpoint to the database.

This method saves a checkpoint to the SQLite database. The checkpoint is associated with the provided config and its parent config (if any).

Parameters:

config (RunnableConfig) – 

The config to associate with the checkpoint.

checkpoint (Checkpoint) – 

The checkpoint to save.

metadata (CheckpointMetadata) – 

Additional metadata to save with the checkpoint.

new_versions (ChannelVersions) – 

New channel versions as of this write.

Returns:

RunnableConfig ( RunnableConfig ) – 

Updated configuration after storing the checkpoint.

Examples:

>>> from langgraph.checkpoint.sqlite import SqliteSaver
>>> with SqliteSaver.from_conn_string(":memory:") as memory:
>>>     config = {"configurable": {"thread_id": "1", "checkpoint_ns": ""}}
>>>     checkpoint = {"ts": "2024-05-04T06:32:42.235444+00:00", "id": "1ef4f797-8335-6428-8001-8a1503f9b875", "channel_values": {"key": "value"}}
>>>     saved_config = memory.put(config, checkpoint, {"source": "input", "step": 1, "writes": {"key": "value"}}, {})
>>> print(saved_config)
{'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef4f797-8335-6428-8001-8a1503f9b875'}}

 put_writes(config: RunnableConfig, writes: Sequence[Tuple[str, Any]], task_id: str, task_path: str = '') -> None ¶

Store intermediate writes linked to a checkpoint.

This method saves intermediate writes associated with a checkpoint to the SQLite database.

Parameters:

config (RunnableConfig) – 

Configuration of the related checkpoint.

writes (Sequence[Tuple[str, Any]]) – 

List of writes to store, each as (channel, value) pair.

task_id (str) – 

Identifier for the task creating the writes.

task_path (str, default: '' ) – 

Path of the task creating the writes.

 aget_tuple(config: RunnableConfig) -> Optional[CheckpointTuple] async ¶

Get a checkpoint tuple from the database asynchronously.

Note

This async method is not supported by the SqliteSaver class. Use get_tuple() instead, or consider using AsyncSqliteSaver.

 alist(config: Optional[RunnableConfig], *, filter: Optional[Dict[str, Any]] = None, before: Optional[RunnableConfig] = None, limit: Optional[int] = None) -> AsyncIterator[CheckpointTuple] async ¶

List checkpoints from the database asynchronously.

Note

This async method is not supported by the SqliteSaver class. Use list() instead, or consider using AsyncSqliteSaver.

 aput(config: RunnableConfig, checkpoint: Checkpoint, metadata: CheckpointMetadata, new_versions: ChannelVersions) -> RunnableConfig async ¶

Save a checkpoint to the database asynchronously.

Note

This async method is not supported by the SqliteSaver class. Use put() instead, or consider using AsyncSqliteSaver.

 get_next_version(current: Optional[str], channel: ChannelProtocol) -> str ¶

Generate the next version ID for a channel.

This method creates a new version identifier for a channel based on its current version.

Parameters:

current (Optional[str]) – 

The current version identifier of the channel.

channel (BaseChannel) – 

The channel being versioned.

Returns:

str ( str ) – 

The next version identifier, which is guaranteed to be monotonically increasing.

 AsyncSqliteSaver ¶

Bases: BaseCheckpointSaver[str]

An asynchronous checkpoint saver that stores checkpoints in a SQLite database.

This class provides an asynchronous interface for saving and retrieving checkpoints using a SQLite database. It's designed for use in asynchronous environments and offers better performance for I/O-bound operations compared to synchronous alternatives.

Attributes:

conn (Connection) – 

The asynchronous SQLite database connection.

serde (SerializerProtocol) – 

The serializer used for encoding/decoding checkpoints.

Tip

Requires the aiosqlite package. Install it with pip install aiosqlite.

Warning

While this class supports asynchronous checkpointing, it is not recommended for production workloads due to limitations in SQLite's write performance. For production use, consider a more robust database like PostgreSQL.

Tip

Remember to close the database connection after executing your code, otherwise, you may see the graph "hang" after execution (since the program will not exit until the connection is closed).

The easiest way is to use the async with statement as shown in the examples.

async with AsyncSqliteSaver.from_conn_string("checkpoints.sqlite") as saver:

    # Your code here

    graph = builder.compile(checkpointer=saver)

    config = {"configurable": {"thread_id": "thread-1"}}

    async for event in graph.astream_events(..., config, version="v1"):

        print(event)


Examples:

Usage within StateGraph:

>>> import asyncio

>>>

>>> from langgraph.checkpoint.sqlite.aio import AsyncSqliteSaver

>>> from langgraph.graph import StateGraph

>>>

>>> builder = StateGraph(int)

>>> builder.add_node("add_one", lambda x: x + 1)

>>> builder.set_entry_point("add_one")

>>> builder.set_finish_point("add_one")

>>> async with AsyncSqliteSaver.from_conn_string("checkpoints.db") as memory:

>>>     graph = builder.compile(checkpointer=memory)

>>>     coro = graph.ainvoke(1, {"configurable": {"thread_id": "thread-1"}})

>>>     print(asyncio.run(coro))

Output: 2

Raw usage:

>>> import asyncio

>>> import aiosqlite

>>> from langgraph.checkpoint.sqlite.aio import AsyncSqliteSaver

>>>

>>> async def main():

>>>     async with aiosqlite.connect("checkpoints.db") as conn:

...         saver = AsyncSqliteSaver(conn)

...         config = {"configurable": {"thread_id": "1"}}

...         checkpoint = {"ts": "2023-05-03T10:00:00Z", "data": {"key": "value"}}

...         saved_config = await saver.aput(config, checkpoint, {}, {})

...         print(saved_config)

>>> asyncio.run(main())

{"configurable": {"thread_id": "1", "checkpoint_id": "0c62ca34-ac19-445d-bbb0-5b4984975b2a"}}

 config_specs: list[ConfigurableFieldSpec] property ¶

Define the configuration options for the checkpoint saver.

Returns:

list[ConfigurableFieldSpec] – 

list[ConfigurableFieldSpec]: List of configuration field specs.

 get(config: RunnableConfig) -> Optional[Checkpoint] ¶

Fetch a checkpoint using the given configuration.

Parameters:

config (RunnableConfig) – 

Configuration specifying which checkpoint to retrieve.

Returns:

Optional[Checkpoint] – 

Optional[Checkpoint]: The requested checkpoint, or None if not found.

 aget(config: RunnableConfig) -> Optional[Checkpoint] async ¶

Asynchronously fetch a checkpoint using the given configuration.

Parameters:

config (RunnableConfig) – 

Configuration specifying which checkpoint to retrieve.

Returns:

Optional[Checkpoint] – 

Optional[Checkpoint]: The requested checkpoint, or None if not found.

 from_conn_string(conn_string: str) -> AsyncIterator[AsyncSqliteSaver] async classmethod ¶

Create a new AsyncSqliteSaver instance from a connection string.

Parameters:

conn_string (str) – 

The SQLite connection string.

Yields:

AsyncSqliteSaver ( AsyncIterator[AsyncSqliteSaver] ) – 

A new AsyncSqliteSaver instance.

 get_tuple(config: RunnableConfig) -> Optional[CheckpointTuple] ¶

Get a checkpoint tuple from the database.

This method retrieves a checkpoint tuple from the SQLite database based on the provided config. If the config contains a "checkpoint_id" key, the checkpoint with the matching thread ID and checkpoint ID is retrieved. Otherwise, the latest checkpoint for the given thread ID is retrieved.

Parameters:

config (RunnableConfig) – 

The config to use for retrieving the checkpoint.

Returns:

Optional[CheckpointTuple] – 

Optional[CheckpointTuple]: The retrieved checkpoint tuple, or None if no matching checkpoint was found.

 list(config: Optional[RunnableConfig], *, filter: Optional[dict[str, Any]] = None, before: Optional[RunnableConfig] = None, limit: Optional[int] = None) -> Iterator[CheckpointTuple] ¶

List checkpoints from the database asynchronously.

This method retrieves a list of checkpoint tuples from the SQLite database based on the provided config. The checkpoints are ordered by checkpoint ID in descending order (newest first).

Parameters:

config (Optional[RunnableConfig]) – 

Base configuration for filtering checkpoints.

filter (Optional[Dict[str, Any]], default: None ) – 

Additional filtering criteria for metadata.

before (Optional[RunnableConfig], default: None ) – 

If provided, only checkpoints before the specified checkpoint ID are returned. Defaults to None.

limit (Optional[int], default: None ) – 

Maximum number of checkpoints to return.

Yields:

CheckpointTuple – 

Iterator[CheckpointTuple]: An iterator of matching checkpoint tuples.

 put(config: RunnableConfig, checkpoint: Checkpoint, metadata: CheckpointMetadata, new_versions: ChannelVersions) -> RunnableConfig ¶

Save a checkpoint to the database.

This method saves a checkpoint to the SQLite database. The checkpoint is associated with the provided config and its parent config (if any).

Parameters:

config (RunnableConfig) – 

The config to associate with the checkpoint.

checkpoint (Checkpoint) – 

The checkpoint to save.

metadata (CheckpointMetadata) – 

Additional metadata to save with the checkpoint.

new_versions (ChannelVersions) – 

New channel versions as of this write.

Returns:

RunnableConfig ( RunnableConfig ) – 

Updated configuration after storing the checkpoint.

 setup() -> None async ¶

Set up the checkpoint database asynchronously.

This method creates the necessary tables in the SQLite database if they don't already exist. It is called automatically when needed and should not be called directly by the user.

 aget_tuple(config: RunnableConfig) -> Optional[CheckpointTuple] async ¶

Get a checkpoint tuple from the database asynchronously.

This method retrieves a checkpoint tuple from the SQLite database based on the provided config. If the config contains a "checkpoint_id" key, the checkpoint with the matching thread ID and checkpoint ID is retrieved. Otherwise, the latest checkpoint for the given thread ID is retrieved.

Parameters:

config (RunnableConfig) – 

The config to use for retrieving the checkpoint.

Returns:

Optional[CheckpointTuple] – 

Optional[CheckpointTuple]: The retrieved checkpoint tuple, or None if no matching checkpoint was found.

 alist(config: Optional[RunnableConfig], *, filter: Optional[dict[str, Any]] = None, before: Optional[RunnableConfig] = None, limit: Optional[int] = None) -> AsyncIterator[CheckpointTuple] async ¶

List checkpoints from the database asynchronously.

This method retrieves a list of checkpoint tuples from the SQLite database based on the provided config. The checkpoints are ordered by checkpoint ID in descending order (newest first).

Parameters:

config (Optional[RunnableConfig]) – 

Base configuration for filtering checkpoints.

filter (Optional[Dict[str, Any]], default: None ) – 

Additional filtering criteria for metadata.

before (Optional[RunnableConfig], default: None ) – 

If provided, only checkpoints before the specified checkpoint ID are returned. Defaults to None.

limit (Optional[int], default: None ) – 

Maximum number of checkpoints to return.

Yields:

AsyncIterator[CheckpointTuple] – 

AsyncIterator[CheckpointTuple]: An asynchronous iterator of matching checkpoint tuples.

 aput(config: RunnableConfig, checkpoint: Checkpoint, metadata: CheckpointMetadata, new_versions: ChannelVersions) -> RunnableConfig async ¶

Save a checkpoint to the database asynchronously.

This method saves a checkpoint to the SQLite database. The checkpoint is associated with the provided config and its parent config (if any).

Parameters:

config (RunnableConfig) – 

The config to associate with the checkpoint.

checkpoint (Checkpoint) – 

The checkpoint to save.

metadata (CheckpointMetadata) – 

Additional metadata to save with the checkpoint.

new_versions (ChannelVersions) – 

New channel versions as of this write.

Returns:

RunnableConfig ( RunnableConfig ) – 

Updated configuration after storing the checkpoint.

 aput_writes(config: RunnableConfig, writes: Sequence[tuple[str, Any]], task_id: str, task_path: str = '') -> None async ¶

Store intermediate writes linked to a checkpoint asynchronously.

This method saves intermediate writes associated with a checkpoint to the database.

Parameters:

config (RunnableConfig) – 

Configuration of the related checkpoint.

writes (Sequence[Tuple[str, Any]]) – 

List of writes to store, each as (channel, value) pair.

task_id (str) – 

Identifier for the task creating the writes.

task_path (str, default: '' ) – 

Path of the task creating the writes.

 get_next_version(current: Optional[str], channel: ChannelProtocol) -> str ¶

Generate the next version ID for a channel.

This method creates a new version identifier for a channel based on its current version.

Parameters:

current (Optional[str]) – 

The current version identifier of the channel.

channel (BaseChannel) – 

The channel being versioned.

Returns:

str ( str ) – 

The next version identifier, which is guaranteed to be monotonically increasing.

 BasePostgresSaver ¶

Bases: BaseCheckpointSaver[str]

 config_specs: list[ConfigurableFieldSpec] property ¶

Define the configuration options for the checkpoint saver.

Returns:

list[ConfigurableFieldSpec] – 

list[ConfigurableFieldSpec]: List of configuration field specs.

 get(config: RunnableConfig) -> Optional[Checkpoint] ¶

Fetch a checkpoint using the given configuration.

Parameters:

config (RunnableConfig) – 

Configuration specifying which checkpoint to retrieve.

Returns:

Optional[Checkpoint] – 

Optional[Checkpoint]: The requested checkpoint, or None if not found.

 get_tuple(config: RunnableConfig) -> Optional[CheckpointTuple] ¶

Fetch a checkpoint tuple using the given configuration.

Parameters:

config (RunnableConfig) – 

Configuration specifying which checkpoint to retrieve.

Returns:

Optional[CheckpointTuple] – 

Optional[CheckpointTuple]: The requested checkpoint tuple, or None if not found.

Raises:

NotImplementedError – 

Implement this method in your custom checkpoint saver.

 list(config: Optional[RunnableConfig], *, filter: Optional[Dict[str, Any]] = None, before: Optional[RunnableConfig] = None, limit: Optional[int] = None) -> Iterator[CheckpointTuple] ¶

List checkpoints that match the given criteria.

Parameters:

config (Optional[RunnableConfig]) – 

Base configuration for filtering checkpoints.

filter (Optional[Dict[str, Any]], default: None ) – 

Additional filtering criteria.

before (Optional[RunnableConfig], default: None ) – 

List checkpoints created before this configuration.

limit (Optional[int], default: None ) – 

Maximum number of checkpoints to return.

Returns:

Iterator[CheckpointTuple] – 

Iterator[CheckpointTuple]: Iterator of matching checkpoint tuples.

Raises:

NotImplementedError – 

Implement this method in your custom checkpoint saver.

 put(config: RunnableConfig, checkpoint: Checkpoint, metadata: CheckpointMetadata, new_versions: ChannelVersions) -> RunnableConfig ¶

Store a checkpoint with its configuration and metadata.

Parameters:

config (RunnableConfig) – 

Configuration for the checkpoint.

checkpoint (Checkpoint) – 

The checkpoint to store.

metadata (CheckpointMetadata) – 

Additional metadata for the checkpoint.

new_versions (ChannelVersions) – 

New channel versions as of this write.

Returns:

RunnableConfig ( RunnableConfig ) – 

Updated configuration after storing the checkpoint.

Raises:

NotImplementedError – 

Implement this method in your custom checkpoint saver.

 put_writes(config: RunnableConfig, writes: Sequence[Tuple[str, Any]], task_id: str, task_path: str = '') -> None ¶

Store intermediate writes linked to a checkpoint.

Parameters:

config (RunnableConfig) – 

Configuration of the related checkpoint.

writes (List[Tuple[str, Any]]) – 

List of writes to store.

task_id (str) – 

Identifier for the task creating the writes.

task_path (str, default: '' ) – 

Path of the task creating the writes.

Raises:

NotImplementedError – 

Implement this method in your custom checkpoint saver.

 aget(config: RunnableConfig) -> Optional[Checkpoint] async ¶

Asynchronously fetch a checkpoint using the given configuration.

Parameters:

config (RunnableConfig) – 

Configuration specifying which checkpoint to retrieve.

Returns:

Optional[Checkpoint] – 

Optional[Checkpoint]: The requested checkpoint, or None if not found.

 aget_tuple(config: RunnableConfig) -> Optional[CheckpointTuple] async ¶

Asynchronously fetch a checkpoint tuple using the given configuration.

Parameters:

config (RunnableConfig) – 

Configuration specifying which checkpoint to retrieve.

Returns:

Optional[CheckpointTuple] – 

Optional[CheckpointTuple]: The requested checkpoint tuple, or None if not found.

Raises:

NotImplementedError – 

Implement this method in your custom checkpoint saver.

 alist(config: Optional[RunnableConfig], *, filter: Optional[Dict[str, Any]] = None, before: Optional[RunnableConfig] = None, limit: Optional[int] = None) -> AsyncIterator[CheckpointTuple] async ¶

Asynchronously list checkpoints that match the given criteria.

Parameters:

config (Optional[RunnableConfig]) – 

Base configuration for filtering checkpoints.

filter (Optional[Dict[str, Any]], default: None ) – 

Additional filtering criteria for metadata.

before (Optional[RunnableConfig], default: None ) – 

List checkpoints created before this configuration.

limit (Optional[int], default: None ) – 

Maximum number of checkpoints to return.

Returns:

AsyncIterator[CheckpointTuple] – 

AsyncIterator[CheckpointTuple]: Async iterator of matching checkpoint tuples.

Raises:

NotImplementedError – 

Implement this method in your custom checkpoint saver.

 aput(config: RunnableConfig, checkpoint: Checkpoint, metadata: CheckpointMetadata, new_versions: ChannelVersions) -> RunnableConfig async ¶

Asynchronously store a checkpoint with its configuration and metadata.

Parameters:

config (RunnableConfig) – 

Configuration for the checkpoint.

checkpoint (Checkpoint) – 

The checkpoint to store.

metadata (CheckpointMetadata) – 

Additional metadata for the checkpoint.

new_versions (ChannelVersions) – 

New channel versions as of this write.

Returns:

RunnableConfig ( RunnableConfig ) – 

Updated configuration after storing the checkpoint.

Raises:

NotImplementedError – 

Implement this method in your custom checkpoint saver.

 aput_writes(config: RunnableConfig, writes: Sequence[Tuple[str, Any]], task_id: str, task_path: str = '') -> None async ¶

Asynchronously store intermediate writes linked to a checkpoint.

Parameters:

config (RunnableConfig) – 

Configuration of the related checkpoint.

writes (List[Tuple[str, Any]]) – 

List of writes to store.

task_id (str) – 

Identifier for the task creating the writes.

task_path (str, default: '' ) – 

Path of the task creating the writes.

Raises:

NotImplementedError – 

Implement this method in your custom checkpoint saver.

 ShallowPostgresSaver ¶

Bases: BasePostgresSaver

A checkpoint saver that uses Postgres to store checkpoints.

This checkpointer ONLY stores the most recent checkpoint and does NOT retain any history. It is meant to be a light-weight drop-in replacement for the PostgresSaver that supports most of the LangGraph persistence functionality with the exception of time travel.

 config_specs: list[ConfigurableFieldSpec] property ¶

Define the configuration options for the checkpoint saver.

Returns:

list[ConfigurableFieldSpec] – 

list[ConfigurableFieldSpec]: List of configuration field specs.

 get(config: RunnableConfig) -> Optional[Checkpoint] ¶

Fetch a checkpoint using the given configuration.

Parameters:

config (RunnableConfig) – 

Configuration specifying which checkpoint to retrieve.

Returns:

Optional[Checkpoint] – 

Optional[Checkpoint]: The requested checkpoint, or None if not found.

 aget(config: RunnableConfig) -> Optional[Checkpoint] async ¶

Asynchronously fetch a checkpoint using the given configuration.

Parameters:

config (RunnableConfig) – 

Configuration specifying which checkpoint to retrieve.

Returns:

Optional[Checkpoint] – 

Optional[Checkpoint]: The requested checkpoint, or None if not found.

 aget_tuple(config: RunnableConfig) -> Optional[CheckpointTuple] async ¶

Asynchronously fetch a checkpoint tuple using the given configuration.

Parameters:

config (RunnableConfig) – 

Configuration specifying which checkpoint to retrieve.

Returns:

Optional[CheckpointTuple] – 

Optional[CheckpointTuple]: The requested checkpoint tuple, or None if not found.

Raises:

NotImplementedError – 

Implement this method in your custom checkpoint saver.

 alist(config: Optional[RunnableConfig], *, filter: Optional[Dict[str, Any]] = None, before: Optional[RunnableConfig] = None, limit: Optional[int] = None) -> AsyncIterator[CheckpointTuple] async ¶

Asynchronously list checkpoints that match the given criteria.

Parameters:

config (Optional[RunnableConfig]) – 

Base configuration for filtering checkpoints.

filter (Optional[Dict[str, Any]], default: None ) – 

Additional filtering criteria for metadata.

before (Optional[RunnableConfig], default: None ) – 

List checkpoints created before this configuration.

limit (Optional[int], default: None ) – 

Maximum number of checkpoints to return.

Returns:

AsyncIterator[CheckpointTuple] – 

AsyncIterator[CheckpointTuple]: Async iterator of matching checkpoint tuples.

Raises:

NotImplementedError – 

Implement this method in your custom checkpoint saver.

 aput(config: RunnableConfig, checkpoint: Checkpoint, metadata: CheckpointMetadata, new_versions: ChannelVersions) -> RunnableConfig async ¶

Asynchronously store a checkpoint with its configuration and metadata.

Parameters:

config (RunnableConfig) – 

Configuration for the checkpoint.

checkpoint (Checkpoint) – 

The checkpoint to store.

metadata (CheckpointMetadata) – 

Additional metadata for the checkpoint.

new_versions (ChannelVersions) – 

New channel versions as of this write.

Returns:

RunnableConfig ( RunnableConfig ) – 

Updated configuration after storing the checkpoint.

Raises:

NotImplementedError – 

Implement this method in your custom checkpoint saver.

 aput_writes(config: RunnableConfig, writes: Sequence[Tuple[str, Any]], task_id: str, task_path: str = '') -> None async ¶

Asynchronously store intermediate writes linked to a checkpoint.

Parameters:

config (RunnableConfig) – 

Configuration of the related checkpoint.

writes (List[Tuple[str, Any]]) – 

List of writes to store.

task_id (str) – 

Identifier for the task creating the writes.

task_path (str, default: '' ) – 

Path of the task creating the writes.

Raises:

NotImplementedError – 

Implement this method in your custom checkpoint saver.

 from_conn_string(conn_string: str, *, pipeline: bool = False) -> Iterator[ShallowPostgresSaver] classmethod ¶

Create a new ShallowPostgresSaver instance from a connection string.

Parameters:

conn_string (str) – 

The Postgres connection info string.

pipeline (bool, default: False ) – 

whether to use Pipeline

Returns:

ShallowPostgresSaver ( Iterator[ShallowPostgresSaver] ) – 

A new ShallowPostgresSaver instance.

 setup() -> None ¶

Set up the checkpoint database asynchronously.

This method creates the necessary tables in the Postgres database if they don't already exist and runs database migrations. It MUST be called directly by the user the first time checkpointer is used.

 list(config: Optional[RunnableConfig], *, filter: Optional[dict[str, Any]] = None, before: Optional[RunnableConfig] = None, limit: Optional[int] = None) -> Iterator[CheckpointTuple] ¶

List checkpoints from the database.

This method retrieves a list of checkpoint tuples from the Postgres database based on the provided config. For ShallowPostgresSaver, this method returns a list with ONLY the most recent checkpoint.

 get_tuple(config: RunnableConfig) -> Optional[CheckpointTuple] ¶

Get a checkpoint tuple from the database.

This method retrieves a checkpoint tuple from the Postgres database based on the provided config (matching the thread ID in the config).

Parameters:

config (RunnableConfig) – 

The config to use for retrieving the checkpoint.

Returns:

Optional[CheckpointTuple] – 

Optional[CheckpointTuple]: The retrieved checkpoint tuple, or None if no matching checkpoint was found.

Examples:

Basic:
>>> config = {"configurable": {"thread_id": "1"}}
>>> checkpoint_tuple = memory.get_tuple(config)
>>> print(checkpoint_tuple)
CheckpointTuple(...)

With timestamp:

>>> config = {
...    "configurable": {
...        "thread_id": "1",
...        "checkpoint_ns": "",
...        "checkpoint_id": "1ef4f797-8335-6428-8001-8a1503f9b875",
...    }
... }
>>> checkpoint_tuple = memory.get_tuple(config)
>>> print(checkpoint_tuple)
CheckpointTuple(...)

 put(config: RunnableConfig, checkpoint: Checkpoint, metadata: CheckpointMetadata, new_versions: ChannelVersions) -> RunnableConfig ¶

Save a checkpoint to the database.

This method saves a checkpoint to the Postgres database. The checkpoint is associated with the provided config. For ShallowPostgresSaver, this method saves ONLY the most recent checkpoint and overwrites a previous checkpoint, if it exists.

Parameters:

config (RunnableConfig) – 

The config to associate with the checkpoint.

checkpoint (Checkpoint) – 

The checkpoint to save.

metadata (CheckpointMetadata) – 

Additional metadata to save with the checkpoint.

new_versions (ChannelVersions) – 

New channel versions as of this write.

Returns:

RunnableConfig ( RunnableConfig ) – 

Updated configuration after storing the checkpoint.

Examples:

>>> from langgraph.checkpoint.postgres import ShallowPostgresSaver
>>> DB_URI = "postgres://postgres:postgres@localhost:5432/postgres?sslmode=disable"
>>> with ShallowPostgresSaver.from_conn_string(DB_URI) as memory:
>>>     config = {"configurable": {"thread_id": "1", "checkpoint_ns": ""}}
>>>     checkpoint = {"ts": "2024-05-04T06:32:42.235444+00:00", "id": "1ef4f797-8335-6428-8001-8a1503f9b875", "channel_values": {"key": "value"}}
>>>     saved_config = memory.put(config, checkpoint, {"source": "input", "step": 1, "writes": {"key": "value"}}, {})
>>> print(saved_config)
{'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef4f797-8335-6428-8001-8a1503f9b875'}}

 put_writes(config: RunnableConfig, writes: Sequence[tuple[str, Any]], task_id: str, task_path: str = '') -> None ¶

Store intermediate writes linked to a checkpoint.

This method saves intermediate writes associated with a checkpoint to the Postgres database.

Parameters:

config (RunnableConfig) – 

Configuration of the related checkpoint.

writes (List[Tuple[str, Any]]) – 

List of writes to store.

task_id (str) – 

Identifier for the task creating the writes.

 PostgresSaver ¶

Bases: BasePostgresSaver

 config_specs: list[ConfigurableFieldSpec] property ¶

Define the configuration options for the checkpoint saver.

Returns:

list[ConfigurableFieldSpec] – 

list[ConfigurableFieldSpec]: List of configuration field specs.

 get(config: RunnableConfig) -> Optional[Checkpoint] ¶

Fetch a checkpoint using the given configuration.

Parameters:

config (RunnableConfig) – 

Configuration specifying which checkpoint to retrieve.

Returns:

Optional[Checkpoint] – 

Optional[Checkpoint]: The requested checkpoint, or None if not found.

 aget(config: RunnableConfig) -> Optional[Checkpoint] async ¶

Asynchronously fetch a checkpoint using the given configuration.

Parameters:

config (RunnableConfig) – 

Configuration specifying which checkpoint to retrieve.

Returns:

Optional[Checkpoint] – 

Optional[Checkpoint]: The requested checkpoint, or None if not found.

 aget_tuple(config: RunnableConfig) -> Optional[CheckpointTuple] async ¶

Asynchronously fetch a checkpoint tuple using the given configuration.

Parameters:

config (RunnableConfig) – 

Configuration specifying which checkpoint to retrieve.

Returns:

Optional[CheckpointTuple] – 

Optional[CheckpointTuple]: The requested checkpoint tuple, or None if not found.

Raises:

NotImplementedError – 

Implement this method in your custom checkpoint saver.

 alist(config: Optional[RunnableConfig], *, filter: Optional[Dict[str, Any]] = None, before: Optional[RunnableConfig] = None, limit: Optional[int] = None) -> AsyncIterator[CheckpointTuple] async ¶

Asynchronously list checkpoints that match the given criteria.

Parameters:

config (Optional[RunnableConfig]) – 

Base configuration for filtering checkpoints.

filter (Optional[Dict[str, Any]], default: None ) – 

Additional filtering criteria for metadata.

before (Optional[RunnableConfig], default: None ) – 

List checkpoints created before this configuration.

limit (Optional[int], default: None ) – 

Maximum number of checkpoints to return.

Returns:

AsyncIterator[CheckpointTuple] – 

AsyncIterator[CheckpointTuple]: Async iterator of matching checkpoint tuples.

Raises:

NotImplementedError – 

Implement this method in your custom checkpoint saver.

 aput(config: RunnableConfig, checkpoint: Checkpoint, metadata: CheckpointMetadata, new_versions: ChannelVersions) -> RunnableConfig async ¶

Asynchronously store a checkpoint with its configuration and metadata.

Parameters:

config (RunnableConfig) – 

Configuration for the checkpoint.

checkpoint (Checkpoint) – 

The checkpoint to store.

metadata (CheckpointMetadata) – 

Additional metadata for the checkpoint.

new_versions (ChannelVersions) – 

New channel versions as of this write.

Returns:

RunnableConfig ( RunnableConfig ) – 

Updated configuration after storing the checkpoint.

Raises:

NotImplementedError – 

Implement this method in your custom checkpoint saver.

 aput_writes(config: RunnableConfig, writes: Sequence[Tuple[str, Any]], task_id: str, task_path: str = '') -> None async ¶

Asynchronously store intermediate writes linked to a checkpoint.

Parameters:

config (RunnableConfig) – 

Configuration of the related checkpoint.

writes (List[Tuple[str, Any]]) – 

List of writes to store.

task_id (str) – 

Identifier for the task creating the writes.

task_path (str, default: '' ) – 

Path of the task creating the writes.

Raises:

NotImplementedError – 

Implement this method in your custom checkpoint saver.

 from_conn_string(conn_string: str, *, pipeline: bool = False) -> Iterator[PostgresSaver] classmethod ¶

Create a new PostgresSaver instance from a connection string.

Parameters:

conn_string (str) – 

The Postgres connection info string.

pipeline (bool, default: False ) – 

whether to use Pipeline

Returns:

PostgresSaver ( Iterator[PostgresSaver] ) – 

A new PostgresSaver instance.

 setup() -> None ¶

Set up the checkpoint database asynchronously.

This method creates the necessary tables in the Postgres database if they don't already exist and runs database migrations. It MUST be called directly by the user the first time checkpointer is used.

 list(config: Optional[RunnableConfig], *, filter: Optional[dict[str, Any]] = None, before: Optional[RunnableConfig] = None, limit: Optional[int] = None) -> Iterator[CheckpointTuple] ¶

List checkpoints from the database.

This method retrieves a list of checkpoint tuples from the Postgres database based on the provided config. The checkpoints are ordered by checkpoint ID in descending order (newest first).

Parameters:

config (RunnableConfig) – 

The config to use for listing the checkpoints.

filter (Optional[Dict[str, Any]], default: None ) – 

Additional filtering criteria for metadata. Defaults to None.

before (Optional[RunnableConfig], default: None ) – 

If provided, only checkpoints before the specified checkpoint ID are returned. Defaults to None.

limit (Optional[int], default: None ) – 

The maximum number of checkpoints to return. Defaults to None.

Yields:

CheckpointTuple – 

Iterator[CheckpointTuple]: An iterator of checkpoint tuples.

Examples:

>>> from langgraph.checkpoint.postgres import PostgresSaver

>>> DB_URI = "postgres://postgres:postgres@localhost:5432/postgres?sslmode=disable"

>>> with PostgresSaver.from_conn_string(DB_URI) as memory:

... # Run a graph, then list the checkpoints

>>>     config = {"configurable": {"thread_id": "1"}}

>>>     checkpoints = list(memory.list(config, limit=2))

>>> print(checkpoints)

[CheckpointTuple(...), CheckpointTuple(...)]

>>> config = {"configurable": {"thread_id": "1"}}

>>> before = {"configurable": {"checkpoint_id": "1ef4f797-8335-6428-8001-8a1503f9b875"}}

>>> with PostgresSaver.from_conn_string(DB_URI) as memory:

... # Run a graph, then list the checkpoints

>>>     checkpoints = list(memory.list(config, before=before))

>>> print(checkpoints)

[CheckpointTuple(...), ...]

 get_tuple(config: RunnableConfig) -> Optional[CheckpointTuple] ¶

Get a checkpoint tuple from the database.

This method retrieves a checkpoint tuple from the Postgres database based on the provided config. If the config contains a "checkpoint_id" key, the checkpoint with the matching thread ID and timestamp is retrieved. Otherwise, the latest checkpoint for the given thread ID is retrieved.

Parameters:

config (RunnableConfig) – 

The config to use for retrieving the checkpoint.

Returns:

Optional[CheckpointTuple] – 

Optional[CheckpointTuple]: The retrieved checkpoint tuple, or None if no matching checkpoint was found.

Examples:

Basic:
>>> config = {"configurable": {"thread_id": "1"}}
>>> checkpoint_tuple = memory.get_tuple(config)
>>> print(checkpoint_tuple)
CheckpointTuple(...)

With timestamp:

>>> config = {
...    "configurable": {
...        "thread_id": "1",
...        "checkpoint_ns": "",
...        "checkpoint_id": "1ef4f797-8335-6428-8001-8a1503f9b875",
...    }
... }
>>> checkpoint_tuple = memory.get_tuple(config)
>>> print(checkpoint_tuple)
CheckpointTuple(...)

 put(config: RunnableConfig, checkpoint: Checkpoint, metadata: CheckpointMetadata, new_versions: ChannelVersions) -> RunnableConfig ¶

Save a checkpoint to the database.

This method saves a checkpoint to the Postgres database. The checkpoint is associated with the provided config and its parent config (if any).

Parameters:

config (RunnableConfig) – 

The config to associate with the checkpoint.

checkpoint (Checkpoint) – 

The checkpoint to save.

metadata (CheckpointMetadata) – 

Additional metadata to save with the checkpoint.

new_versions (ChannelVersions) – 

New channel versions as of this write.

Returns:

RunnableConfig ( RunnableConfig ) – 

Updated configuration after storing the checkpoint.

Examples:

>>> from langgraph.checkpoint.postgres import PostgresSaver
>>> DB_URI = "postgres://postgres:postgres@localhost:5432/postgres?sslmode=disable"
>>> with PostgresSaver.from_conn_string(DB_URI) as memory:
>>>     config = {"configurable": {"thread_id": "1", "checkpoint_ns": ""}}
>>>     checkpoint = {"ts": "2024-05-04T06:32:42.235444+00:00", "id": "1ef4f797-8335-6428-8001-8a1503f9b875", "channel_values": {"key": "value"}}
>>>     saved_config = memory.put(config, checkpoint, {"source": "input", "step": 1, "writes": {"key": "value"}}, {})
>>> print(saved_config)
{'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef4f797-8335-6428-8001-8a1503f9b875'}}

 put_writes(config: RunnableConfig, writes: Sequence[tuple[str, Any]], task_id: str, task_path: str = '') -> None ¶

Store intermediate writes linked to a checkpoint.

This method saves intermediate writes associated with a checkpoint to the Postgres database.

Parameters:

config (RunnableConfig) – 

Configuration of the related checkpoint.

writes (List[Tuple[str, Any]]) – 

List of writes to store.

task_id (str) – 

Identifier for the task creating the writes.

 AsyncShallowPostgresSaver ¶

Bases: BasePostgresSaver

A checkpoint saver that uses Postgres to store checkpoints asynchronously.

This checkpointer ONLY stores the most recent checkpoint and does NOT retain any history. It is meant to be a light-weight drop-in replacement for the AsyncPostgresSaver that supports most of the LangGraph persistence functionality with the exception of time travel.

 config_specs: list[ConfigurableFieldSpec] property ¶

Define the configuration options for the checkpoint saver.

Returns:

list[ConfigurableFieldSpec] – 

list[ConfigurableFieldSpec]: List of configuration field specs.

 get(config: RunnableConfig) -> Optional[Checkpoint] ¶

Fetch a checkpoint using the given configuration.

Parameters:

config (RunnableConfig) – 

Configuration specifying which checkpoint to retrieve.

Returns:

Optional[Checkpoint] – 

Optional[Checkpoint]: The requested checkpoint, or None if not found.

 aget(config: RunnableConfig) -> Optional[Checkpoint] async ¶

Asynchronously fetch a checkpoint using the given configuration.

Parameters:

config (RunnableConfig) – 

Configuration specifying which checkpoint to retrieve.

Returns:

Optional[Checkpoint] – 

Optional[Checkpoint]: The requested checkpoint, or None if not found.

 from_conn_string(conn_string: str, *, pipeline: bool = False, serde: Optional[SerializerProtocol] = None) -> AsyncIterator[AsyncShallowPostgresSaver] async classmethod ¶

Create a new AsyncShallowPostgresSaver instance from a connection string.

Parameters:

conn_string (str) – 

The Postgres connection info string.

pipeline (bool, default: False ) – 

whether to use AsyncPipeline

Returns:

AsyncShallowPostgresSaver ( AsyncIterator[AsyncShallowPostgresSaver] ) – 

A new AsyncShallowPostgresSaver instance.

 setup() -> None async ¶

Set up the checkpoint database asynchronously.

This method creates the necessary tables in the Postgres database if they don't already exist and runs database migrations. It MUST be called directly by the user the first time checkpointer is used.

 alist(config: Optional[RunnableConfig], *, filter: Optional[dict[str, Any]] = None, before: Optional[RunnableConfig] = None, limit: Optional[int] = None) -> AsyncIterator[CheckpointTuple] async ¶

List checkpoints from the database asynchronously.

This method retrieves a list of checkpoint tuples from the Postgres database based on the provided config. For ShallowPostgresSaver, this method returns a list with ONLY the most recent checkpoint.

 aget_tuple(config: RunnableConfig) -> Optional[CheckpointTuple] async ¶

Get a checkpoint tuple from the database asynchronously.

This method retrieves a checkpoint tuple from the Postgres database based on the provided config (matching the thread ID in the config).

Parameters:

config (RunnableConfig) – 

The config to use for retrieving the checkpoint.

Returns:

Optional[CheckpointTuple] – 

Optional[CheckpointTuple]: The retrieved checkpoint tuple, or None if no matching checkpoint was found.

 aput(config: RunnableConfig, checkpoint: Checkpoint, metadata: CheckpointMetadata, new_versions: ChannelVersions) -> RunnableConfig async ¶

Save a checkpoint to the database asynchronously.

This method saves a checkpoint to the Postgres database. The checkpoint is associated with the provided config. For AsyncShallowPostgresSaver, this method saves ONLY the most recent checkpoint and overwrites a previous checkpoint, if it exists.

Parameters:

config (RunnableConfig) – 

The config to associate with the checkpoint.

checkpoint (Checkpoint) – 

The checkpoint to save.

metadata (CheckpointMetadata) – 

Additional metadata to save with the checkpoint.

new_versions (ChannelVersions) – 

New channel versions as of this write.

Returns:

RunnableConfig ( RunnableConfig ) – 

Updated configuration after storing the checkpoint.

 aput_writes(config: RunnableConfig, writes: Sequence[tuple[str, Any]], task_id: str, task_path: str = '') -> None async ¶

Store intermediate writes linked to a checkpoint asynchronously.

This method saves intermediate writes associated with a checkpoint to the database.

Parameters:

config (RunnableConfig) – 

Configuration of the related checkpoint.

writes (Sequence[Tuple[str, Any]]) – 

List of writes to store, each as (channel, value) pair.

task_id (str) – 

Identifier for the task creating the writes.

 list(config: Optional[RunnableConfig], *, filter: Optional[dict[str, Any]] = None, before: Optional[RunnableConfig] = None, limit: Optional[int] = None) -> Iterator[CheckpointTuple] ¶

List checkpoints from the database.

This method retrieves a list of checkpoint tuples from the Postgres database based on the provided config. For ShallowPostgresSaver, this method returns a list with ONLY the most recent checkpoint.

 get_tuple(config: RunnableConfig) -> Optional[CheckpointTuple] ¶

Get a checkpoint tuple from the database.

This method retrieves a checkpoint tuple from the Postgres database based on the provided config (matching the thread ID in the config).

Parameters:

config (RunnableConfig) – 

The config to use for retrieving the checkpoint.

Returns:

Optional[CheckpointTuple] – 

Optional[CheckpointTuple]: The retrieved checkpoint tuple, or None if no matching checkpoint was found.

 put(config: RunnableConfig, checkpoint: Checkpoint, metadata: CheckpointMetadata, new_versions: ChannelVersions) -> RunnableConfig ¶

Save a checkpoint to the database.

This method saves a checkpoint to the Postgres database. The checkpoint is associated with the provided config. For AsyncShallowPostgresSaver, this method saves ONLY the most recent checkpoint and overwrites a previous checkpoint, if it exists.

Parameters:

config (RunnableConfig) – 

The config to associate with the checkpoint.

checkpoint (Checkpoint) – 

The checkpoint to save.

metadata (CheckpointMetadata) – 

Additional metadata to save with the checkpoint.

new_versions (ChannelVersions) – 

New channel versions as of this write.

Returns:

RunnableConfig ( RunnableConfig ) – 

Updated configuration after storing the checkpoint.

 put_writes(config: RunnableConfig, writes: Sequence[tuple[str, Any]], task_id: str, task_path: str = '') -> None ¶

Store intermediate writes linked to a checkpoint.

This method saves intermediate writes associated with a checkpoint to the database.

Parameters:

config (RunnableConfig) – 

Configuration of the related checkpoint.

writes (Sequence[Tuple[str, Any]]) – 

List of writes to store, each as (channel, value) pair.

task_id (str) – 

Identifier for the task creating the writes.

task_path (str, default: '' ) – 

Path of the task creating the writes.

 AsyncPostgresSaver ¶

Bases: BasePostgresSaver

 config_specs: list[ConfigurableFieldSpec] property ¶

Define the configuration options for the checkpoint saver.

Returns:

list[ConfigurableFieldSpec] – 

list[ConfigurableFieldSpec]: List of configuration field specs.

 get(config: RunnableConfig) -> Optional[Checkpoint] ¶

Fetch a checkpoint using the given configuration.

Parameters:

config (RunnableConfig) – 

Configuration specifying which checkpoint to retrieve.

Returns:

Optional[Checkpoint] – 

Optional[Checkpoint]: The requested checkpoint, or None if not found.

 aget(config: RunnableConfig) -> Optional[Checkpoint] async ¶

Asynchronously fetch a checkpoint using the given configuration.

Parameters:

config (RunnableConfig) – 

Configuration specifying which checkpoint to retrieve.

Returns:

Optional[Checkpoint] – 

Optional[Checkpoint]: The requested checkpoint, or None if not found.

 from_conn_string(conn_string: str, *, pipeline: bool = False, serde: Optional[SerializerProtocol] = None) -> AsyncIterator[AsyncPostgresSaver] async classmethod ¶

Create a new AsyncPostgresSaver instance from a connection string.

Parameters:

conn_string (str) – 

The Postgres connection info string.

pipeline (bool, default: False ) – 

whether to use AsyncPipeline

Returns:

AsyncPostgresSaver ( AsyncIterator[AsyncPostgresSaver] ) – 

A new AsyncPostgresSaver instance.

 setup() -> None async ¶

Set up the checkpoint database asynchronously.

This method creates the necessary tables in the Postgres database if they don't already exist and runs database migrations. It MUST be called directly by the user the first time checkpointer is used.

 alist(config: Optional[RunnableConfig], *, filter: Optional[dict[str, Any]] = None, before: Optional[RunnableConfig] = None, limit: Optional[int] = None) -> AsyncIterator[CheckpointTuple] async ¶

List checkpoints from the database asynchronously.

This method retrieves a list of checkpoint tuples from the Postgres database based on the provided config. The checkpoints are ordered by checkpoint ID in descending order (newest first).

Parameters:

config (Optional[RunnableConfig]) – 

Base configuration for filtering checkpoints.

filter (Optional[Dict[str, Any]], default: None ) – 

Additional filtering criteria for metadata.

before (Optional[RunnableConfig], default: None ) – 

If provided, only checkpoints before the specified checkpoint ID are returned. Defaults to None.

limit (Optional[int], default: None ) – 

Maximum number of checkpoints to return.

Yields:

AsyncIterator[CheckpointTuple] – 

AsyncIterator[CheckpointTuple]: An asynchronous iterator of matching checkpoint tuples.

 aget_tuple(config: RunnableConfig) -> Optional[CheckpointTuple] async ¶

Get a checkpoint tuple from the database asynchronously.

This method retrieves a checkpoint tuple from the Postgres database based on the provided config. If the config contains a "checkpoint_id" key, the checkpoint with the matching thread ID and "checkpoint_id" is retrieved. Otherwise, the latest checkpoint for the given thread ID is retrieved.

Parameters:

config (RunnableConfig) – 

The config to use for retrieving the checkpoint.

Returns:

Optional[CheckpointTuple] – 

Optional[CheckpointTuple]: The retrieved checkpoint tuple, or None if no matching checkpoint was found.

 aput(config: RunnableConfig, checkpoint: Checkpoint, metadata: CheckpointMetadata, new_versions: ChannelVersions) -> RunnableConfig async ¶

Save a checkpoint to the database asynchronously.

This method saves a checkpoint to the Postgres database. The checkpoint is associated with the provided config and its parent config (if any).

Parameters:

config (RunnableConfig) – 

The config to associate with the checkpoint.

checkpoint (Checkpoint) – 

The checkpoint to save.

metadata (CheckpointMetadata) – 

Additional metadata to save with the checkpoint.

new_versions (ChannelVersions) – 

New channel versions as of this write.

Returns:

RunnableConfig ( RunnableConfig ) – 

Updated configuration after storing the checkpoint.

 aput_writes(config: RunnableConfig, writes: Sequence[tuple[str, Any]], task_id: str, task_path: str = '') -> None async ¶

Store intermediate writes linked to a checkpoint asynchronously.

This method saves intermediate writes associated with a checkpoint to the database.

Parameters:

config (RunnableConfig) – 

Configuration of the related checkpoint.

writes (Sequence[Tuple[str, Any]]) – 

List of writes to store, each as (channel, value) pair.

task_id (str) – 

Identifier for the task creating the writes.

 list(config: Optional[RunnableConfig], *, filter: Optional[dict[str, Any]] = None, before: Optional[RunnableConfig] = None, limit: Optional[int] = None) -> Iterator[CheckpointTuple] ¶

List checkpoints from the database.

This method retrieves a list of checkpoint tuples from the Postgres database based on the provided config. The checkpoints are ordered by checkpoint ID in descending order (newest first).

Parameters:

config (Optional[RunnableConfig]) – 

Base configuration for filtering checkpoints.

filter (Optional[Dict[str, Any]], default: None ) – 

Additional filtering criteria for metadata.

before (Optional[RunnableConfig], default: None ) – 

If provided, only checkpoints before the specified checkpoint ID are returned. Defaults to None.

limit (Optional[int], default: None ) – 

Maximum number of checkpoints to return.

Yields:

CheckpointTuple – 

Iterator[CheckpointTuple]: An iterator of matching checkpoint tuples.

 get_tuple(config: RunnableConfig) -> Optional[CheckpointTuple] ¶

Get a checkpoint tuple from the database.

This method retrieves a checkpoint tuple from the Postgres database based on the provided config. If the config contains a "checkpoint_id" key, the checkpoint with the matching thread ID and "checkpoint_id" is retrieved. Otherwise, the latest checkpoint for the given thread ID is retrieved.

Parameters:

config (RunnableConfig) – 

The config to use for retrieving the checkpoint.

Returns:

Optional[CheckpointTuple] – 

Optional[CheckpointTuple]: The retrieved checkpoint tuple, or None if no matching checkpoint was found.

 put(config: RunnableConfig, checkpoint: Checkpoint, metadata: CheckpointMetadata, new_versions: ChannelVersions) -> RunnableConfig ¶

Save a checkpoint to the database.

This method saves a checkpoint to the Postgres database. The checkpoint is associated with the provided config and its parent config (if any).

Parameters:

config (RunnableConfig) – 

The config to associate with the checkpoint.

checkpoint (Checkpoint) – 

The checkpoint to save.

metadata (CheckpointMetadata) – 

Additional metadata to save with the checkpoint.

new_versions (ChannelVersions) – 

New channel versions as of this write.

Returns:

RunnableConfig ( RunnableConfig ) – 

Updated configuration after storing the checkpoint.

 put_writes(config: RunnableConfig, writes: Sequence[tuple[str, Any]], task_id: str, task_path: str = '') -> None ¶

Store intermediate writes linked to a checkpoint.

This method saves intermediate writes associated with a checkpoint to the database.

Parameters:

config (RunnableConfig) – 

Configuration of the related checkpoint.

writes (Sequence[Tuple[str, Any]]) – 

List of writes to store, each as (channel, value) pair.

task_id (str) – 

Identifier for the task creating the writes.

task_path (str, default: '' ) – 

Path of the task creating the writes.

Comments
