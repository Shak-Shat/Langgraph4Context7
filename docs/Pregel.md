# Pregel



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/reference/pregel/
- **html**: API reference
Library
Pregel¶
 Pregel ¶

Bases: PregelProtocol

Pregel manages the runtime behavior for LangGraph applications.

Overview¶

Pregel combines actors and channels into a single application. Actors read data from channels and write data to channels. Pregel organizes the execution of the application into multiple steps, following the Pregel Algorithm/Bulk Synchronous Parallel model.

Each step consists of three phases:

Plan: Determine which actors to execute in this step. For example, in the first step, select the actors that subscribe to the special input channels; in subsequent steps, select the actors that subscribe to channels updated in the previous step.
Execution: Execute all selected actors in parallel, until all complete, or one fails, or a timeout is reached. During this phase, channel updates are invisible to actors until the next step.
Update: Update the channels with the values written by the actors in this step.

Repeat until no actors are selected for execution, or a maximum number of steps is reached.

Actors¶

An actor is a PregelNode. It subscribes to channels, reads data from them, and writes data to them. It can be thought of as an actor in the Pregel algorithm. PregelNodes implement LangChain's Runnable interface.

Channels¶

Channels are used to communicate between actors (PregelNodes). Each channel has a value type, an update type, and an update function – which takes a sequence of updates and modifies the stored value. Channels can be used to send data from one chain to another, or to send data from a chain to itself in a future step. LangGraph provides a number of built-in channels:

BASIC CHANNELS: LASTVALUE AND TOPIC¶
LastValue: The default channel, stores the last value sent to the channel, useful for input and output values, or for sending data from one step to the next
Topic: A configurable PubSub Topic, useful for sending multiple values between actors, or for accumulating output. Can be configured to deduplicate values, and/or to accumulate values over the course of multiple steps.
ADVANCED CHANNELS: CONTEXT AND BINARYOPERATORAGGREGATE¶
Context: exposes the value of a context manager, managing its lifecycle. Useful for accessing external resources that require setup and/or teardown. eg. client = Context(httpx.Client)
BinaryOperatorAggregate: stores a persistent value, updated by applying a binary operator to the current value and each update sent to the channel, useful for computing aggregates over multiple steps. eg. total = BinaryOperatorAggregate(int, operator.add)
Examples¶

Most users will interact with Pregel via a StateGraph (Graph API) or via an entrypoint (Functional API).

However, for advanced use cases, Pregel can be used directly. If you're not sure whether you need to use Pregel directly, then the answer is probably no – you should use the Graph API or Functional API instead. These are higher-level interfaces that will compile down to Pregel under the hood.

Here are some examples to give you a sense of how it works:

Single node application
from langgraph.channels import EphemeralValue

from langgraph.pregel import Pregel, Channel, ChannelWriteEntry



node1 = (

    Channel.subscribe_to("a")

    | (lambda x: x + x)

    | Channel.write_to("b")

)



app = Pregel(

    nodes={"node1": node1},

    channels={

        "a": EphemeralValue(str),

        "b": EphemeralValue(str),

    },

    input_channels=["a"],

    output_channels=["b"],

)



app.invoke({"a": "foo"})

{'b': 'foofoo'}

Using multiple nodes and multiple output channels
from langgraph.channels import LastValue, EphemeralValue

from langgraph.pregel import Pregel, Channel, ChannelWriteEntry



node1 = (

    Channel.subscribe_to("a")

    | (lambda x: x + x)

    | Channel.write_to("b")

)



node2 = (

    Channel.subscribe_to("b")

    | (lambda x: x + x)

    | Channel.write_to("c")

)





app = Pregel(

    nodes={"node1": node1, "node2": node2},

    channels={

        "a": EphemeralValue(str),

        "b": LastValue(str),

        "c": EphemeralValue(str),

    },

    input_channels=["a"],

    output_channels=["b", "c"],

)



app.invoke({"a": "foo"})

{'b': 'foofoo', 'c': 'foofoofoofoo'}

Using a Topic channel
from langgraph.channels import LastValue, EphemeralValue, Topic

from langgraph.pregel import Pregel, Channel, ChannelWriteEntry



node1 = (

    Channel.subscribe_to("a")

    | (lambda x: x + x)

    | {

        "b": Channel.write_to("b"),

        "c": Channel.write_to("c")

    }

)



node2 = (

    Channel.subscribe_to("b")

    | (lambda x: x + x)

    | {

        "c": Channel.write_to("c"),

    }

)





app = Pregel(

    nodes={"node1": node1, "node2": node2},

    channels={

        "a": EphemeralValue(str),

        "b": EphemeralValue(str),

        "c": Topic(str, accumulate=True),

    },

    input_channels=["a"],

    output_channels=["c"],

)



app.invoke({"a": "foo"})

{'c': ['foofoo', 'foofoofoofoo']}

Using a BinaryOperatorAggregate channel
from langgraph.channels import EphemeralValue, BinaryOperatorAggregate

from langgraph.pregel import Pregel, Channel





node1 = (

    Channel.subscribe_to("a")

    | (lambda x: x + x)

    | {

        "b": Channel.write_to("b"),

        "c": Channel.write_to("c")

    }

)



node2 = (

    Channel.subscribe_to("b")

    | (lambda x: x + x)

    | {

        "c": Channel.write_to("c"),

    }

)





def reducer(current, update):

    if current:

        return current + " | " + "update"

    else:

        return update



app = Pregel(

    nodes={"node1": node1, "node2": node2},

    channels={

        "a": EphemeralValue(str),

        "b": EphemeralValue(str),

        "c": BinaryOperatorAggregate(str, operator=reducer),

    },

    input_channels=["a"],

    output_channels=["c"]

)



app.invoke({"a": "foo"})

{'c': 'foofoo | foofoofoofoo'}

Introducing a cycle

This example demonstrates how to introduce a cycle in the graph, by having a chain write to a channel it subscribes to. Execution will continue until a None value is written to the channel.

from langgraph.channels import EphemeralValue

from langgraph.pregel import Pregel, Channel, ChannelWrite, ChannelWriteEntry



example_node = (

    Channel.subscribe_to("value")

    | (lambda x: x + x if len(x) < 10 else None)

    | ChannelWrite(writes=[ChannelWriteEntry(channel="value", skip_none=True)])

)



app = Pregel(

    nodes={"example_node": example_node},

    channels={

        "value": EphemeralValue(str),

    },

    input_channels=["value"],

    output_channels=["value"]

)



app.invoke({"value": "a"})

{'value': 'aaaaaaaaaaaaaaaa'}

 stream_mode: StreamMode = stream_mode class-attribute instance-attribute ¶

Mode to stream output, defaults to 'values'.

 stream_eager: bool = stream_eager class-attribute instance-attribute ¶

Whether to force emitting stream events eagerly, automatically turned on for stream_mode "messages" and "custom".

 stream_channels: Optional[Union[str, Sequence[str]]] = stream_channels class-attribute instance-attribute ¶

Channels to stream, defaults to all channels not in reserved channels

 step_timeout: Optional[float] = step_timeout class-attribute instance-attribute ¶

Maximum time to wait for a step to complete, in seconds. Defaults to None.

 debug: bool = debug if debug is not None else get_debug() instance-attribute ¶

Whether to print debug information during execution. Defaults to False.

 checkpointer: Checkpointer = checkpointer class-attribute instance-attribute ¶

Checkpointer used to save and load graph state. Defaults to None.

 store: Optional[BaseStore] = store class-attribute instance-attribute ¶

Memory store to use for SharedValues. Defaults to None.

 retry_policy: Optional[RetryPolicy] = retry_policy class-attribute instance-attribute ¶

Retry policy to use when running tasks. Set to None to disable.

 get_state(config: RunnableConfig, *, subgraphs: bool = False) -> StateSnapshot ¶

Get the current state of the graph.

 aget_state(config: RunnableConfig, *, subgraphs: bool = False) -> StateSnapshot async ¶

Get the current state of the graph.

 update_state(config: RunnableConfig, values: Optional[Union[dict[str, Any], Any]], as_node: Optional[str] = None) -> RunnableConfig ¶

Update the state of the graph with the given values, as if they came from node as_node. If as_node is not provided, it will be set to the last node that updated the state, if not ambiguous.

 aupdate_state(config: RunnableConfig, values: dict[str, Any] | Any, as_node: Optional[str] = None) -> RunnableConfig async ¶

Update the state of the graph asynchronously with the given values, as if they came from node as_node. If as_node is not provided, it will be set to the last node that updated the state, if not ambiguous.

 stream(input: Union[dict[str, Any], Any], config: Optional[RunnableConfig] = None, *, stream_mode: Optional[Union[StreamMode, list[StreamMode]]] = None, output_keys: Optional[Union[str, Sequence[str]]] = None, interrupt_before: Optional[Union[All, Sequence[str]]] = None, interrupt_after: Optional[Union[All, Sequence[str]]] = None, debug: Optional[bool] = None, subgraphs: bool = False) -> Iterator[Union[dict[str, Any], Any]] ¶

Stream graph steps for a single input.

Parameters:

input (Union[dict[str, Any], Any]) – 

The input to the graph.

config (Optional[RunnableConfig], default: None ) – 

The configuration to use for the run.

stream_mode (Optional[Union[StreamMode, list[StreamMode]]], default: None ) – 

The mode to stream output, defaults to self.stream_mode. Options are:

"values": Emit all values in the state after each step. When used with functional API, values are emitted once at the end of the workflow.
"updates": Emit only the node or task names and updates returned by the nodes or tasks after each step. If multiple updates are made in the same step (e.g. multiple nodes are run) then those updates are emitted separately.
"custom": Emit custom data from inside nodes or tasks using StreamWriter.
"messages": Emit LLM messages token-by-token together with metadata for any LLM invocations inside nodes or tasks.
"debug": Emit debug events with as much information as possible for each step.
output_keys (Optional[Union[str, Sequence[str]]], default: None ) – 

The keys to stream, defaults to all non-context channels.

interrupt_before (Optional[Union[All, Sequence[str]]], default: None ) – 

Nodes to interrupt before, defaults to all nodes in the graph.

interrupt_after (Optional[Union[All, Sequence[str]]], default: None ) – 

Nodes to interrupt after, defaults to all nodes in the graph.

debug (Optional[bool], default: None ) – 

Whether to print debug information during execution, defaults to False.

subgraphs (bool, default: False ) – 

Whether to stream subgraphs, defaults to False.

Yields:

Union[dict[str, Any], Any] – 

The output of each step in the graph. The output shape depends on the stream_mode.

Examples:

Using different stream modes with a graph:

>>> import operator

>>> from typing_extensions import Annotated, TypedDict

>>> from langgraph.graph import StateGraph, START

...

>>> class State(TypedDict):

...     alist: Annotated[list, operator.add]

...     another_list: Annotated[list, operator.add]

...

>>> builder = StateGraph(State)

>>> builder.add_node("a", lambda _state: {"another_list": ["hi"]})

>>> builder.add_node("b", lambda _state: {"alist": ["there"]})

>>> builder.add_edge("a", "b")

>>> builder.add_edge(START, "a")

>>> graph = builder.compile()

With stream_mode="values":

>>> for event in graph.stream({"alist": ['Ex for stream_mode="values"']}, stream_mode="values"):

...     print(event)

{'alist': ['Ex for stream_mode="values"'], 'another_list': []}

{'alist': ['Ex for stream_mode="values"'], 'another_list': ['hi']}

{'alist': ['Ex for stream_mode="values"', 'there'], 'another_list': ['hi']}

With stream_mode="updates":

>>> for event in graph.stream({"alist": ['Ex for stream_mode="updates"']}, stream_mode="updates"):

...     print(event)

{'a': {'another_list': ['hi']}}

{'b': {'alist': ['there']}}

With stream_mode="debug":

>>> for event in graph.stream({"alist": ['Ex for stream_mode="debug"']}, stream_mode="debug"):

...     print(event)

{'type': 'task', 'timestamp': '2024-06-23T...+00:00', 'step': 1, 'payload': {'id': '...', 'name': 'a', 'input': {'alist': ['Ex for stream_mode="debug"'], 'another_list': []}, 'triggers': ['start:a']}}

{'type': 'task_result', 'timestamp': '2024-06-23T...+00:00', 'step': 1, 'payload': {'id': '...', 'name': 'a', 'result': [('another_list', ['hi'])]}}

{'type': 'task', 'timestamp': '2024-06-23T...+00:00', 'step': 2, 'payload': {'id': '...', 'name': 'b', 'input': {'alist': ['Ex for stream_mode="debug"'], 'another_list': ['hi']}, 'triggers': ['a']}}

{'type': 'task_result', 'timestamp': '2024-06-23T...+00:00', 'step': 2, 'payload': {'id': '...', 'name': 'b', 'result': [('alist', ['there'])]}}


With stream_mode="custom":

>>> from langgraph.types import StreamWriter

...

>>> def node_a(state: State, writer: StreamWriter):

...     writer({"custom_data": "foo"})

...     return {"alist": ["hi"]}

...

>>> builder = StateGraph(State)

>>> builder.add_node("a", node_a)

>>> builder.add_edge(START, "a")

>>> graph = builder.compile()

...

>>> for event in graph.stream({"alist": ['Ex for stream_mode="custom"']}, stream_mode="custom"):

...     print(event)

{'custom_data': 'foo'}


With stream_mode="messages":

>>> from typing_extensions import Annotated, TypedDict

>>> from langgraph.graph import StateGraph, START

>>> from langchain_openai import ChatOpenAI

...

>>> llm = ChatOpenAI(model="gpt-4o-mini")

...

>>> class State(TypedDict):

...     question: str

...     answer: str

...

>>> def node_a(state: State):

...     response = llm.invoke(state["question"])

...     return {"answer": response.content}

...

>>> builder = StateGraph(State)

>>> builder.add_node("a", node_a)

>>> builder.add_edge(START, "a")

>>> graph = builder.compile()



>>> for event in graph.stream({"question": "What is the capital of France?"}, stream_mode="messages"):

...     print(event)

(AIMessageChunk(content='The', additional_kwargs={}, response_metadata={}, id='...'), {'langgraph_step': 1, 'langgraph_node': 'a', 'langgraph_triggers': ['start:a'], 'langgraph_path': ('__pregel_pull', 'a'), 'langgraph_checkpoint_ns': '...', 'checkpoint_ns': '...', 'ls_provider': 'openai', 'ls_model_name': 'gpt-4o-mini', 'ls_model_type': 'chat', 'ls_temperature': 0.7})

(AIMessageChunk(content=' capital', additional_kwargs={}, response_metadata={}, id='...'), {'langgraph_step': 1, 'langgraph_node': 'a', 'langgraph_triggers': ['start:a'], ...})

(AIMessageChunk(content=' of', additional_kwargs={}, response_metadata={}, id='...'), {...})

(AIMessageChunk(content=' France', additional_kwargs={}, response_metadata={}, id='...'), {...})

(AIMessageChunk(content=' is', additional_kwargs={}, response_metadata={}, id='...'), {...})

(AIMessageChunk(content=' Paris', additional_kwargs={}, response_metadata={}, id='...'), {...})

 astream(input: Union[dict[str, Any], Any], config: Optional[RunnableConfig] = None, *, stream_mode: Optional[Union[StreamMode, list[StreamMode]]] = None, output_keys: Optional[Union[str, Sequence[str]]] = None, interrupt_before: Optional[Union[All, Sequence[str]]] = None, interrupt_after: Optional[Union[All, Sequence[str]]] = None, debug: Optional[bool] = None, subgraphs: bool = False) -> AsyncIterator[Union[dict[str, Any], Any]] async ¶

Stream graph steps for a single input.

Parameters:

input (Union[dict[str, Any], Any]) – 

The input to the graph.

config (Optional[RunnableConfig], default: None ) – 

The configuration to use for the run.

stream_mode (Optional[Union[StreamMode, list[StreamMode]]], default: None ) – 

The mode to stream output, defaults to self.stream_mode. Options are:

"values": Emit all values in the state after each step. When used with functional API, values are emitted once at the end of the workflow.
"updates": Emit only the node or task names and updates returned by the nodes or tasks after each step. If multiple updates are made in the same step (e.g. multiple nodes are run) then those updates are emitted separately.
"custom": Emit custom data from inside nodes or tasks using StreamWriter.
"messages": Emit LLM messages token-by-token together with metadata for any LLM invocations inside nodes or tasks.
"debug": Emit debug events with as much information as possible for each step.
output_keys (Optional[Union[str, Sequence[str]]], default: None ) – 

The keys to stream, defaults to all non-context channels.

interrupt_before (Optional[Union[All, Sequence[str]]], default: None ) – 

Nodes to interrupt before, defaults to all nodes in the graph.

interrupt_after (Optional[Union[All, Sequence[str]]], default: None ) – 

Nodes to interrupt after, defaults to all nodes in the graph.

debug (Optional[bool], default: None ) – 

Whether to print debug information during execution, defaults to False.

subgraphs (bool, default: False ) – 

Whether to stream subgraphs, defaults to False.

Yields:

AsyncIterator[Union[dict[str, Any], Any]] – 

The output of each step in the graph. The output shape depends on the stream_mode.

Examples:

Using different stream modes with a graph:

>>> import operator

>>> from typing_extensions import Annotated, TypedDict

>>> from langgraph.graph import StateGraph, START

...

>>> class State(TypedDict):

...     alist: Annotated[list, operator.add]

...     another_list: Annotated[list, operator.add]

...

>>> builder = StateGraph(State)

>>> builder.add_node("a", lambda _state: {"another_list": ["hi"]})

>>> builder.add_node("b", lambda _state: {"alist": ["there"]})

>>> builder.add_edge("a", "b")

>>> builder.add_edge(START, "a")

>>> graph = builder.compile()

With stream_mode="values":

>>> async for event in graph.astream({"alist": ['Ex for stream_mode="values"']}, stream_mode="values"):

...     print(event)

{'alist': ['Ex for stream_mode="values"'], 'another_list': []}

{'alist': ['Ex for stream_mode="values"'], 'another_list': ['hi']}

{'alist': ['Ex for stream_mode="values"', 'there'], 'another_list': ['hi']}

With stream_mode="updates":

>>> async for event in graph.astream({"alist": ['Ex for stream_mode="updates"']}, stream_mode="updates"):

...     print(event)

{'a': {'another_list': ['hi']}}

{'b': {'alist': ['there']}}

With stream_mode="debug":

>>> async for event in graph.astream({"alist": ['Ex for stream_mode="debug"']}, stream_mode="debug"):

...     print(event)

{'type': 'task', 'timestamp': '2024-06-23T...+00:00', 'step': 1, 'payload': {'id': '...', 'name': 'a', 'input': {'alist': ['Ex for stream_mode="debug"'], 'another_list': []}, 'triggers': ['start:a']}}

{'type': 'task_result', 'timestamp': '2024-06-23T...+00:00', 'step': 1, 'payload': {'id': '...', 'name': 'a', 'result': [('another_list', ['hi'])]}}

{'type': 'task', 'timestamp': '2024-06-23T...+00:00', 'step': 2, 'payload': {'id': '...', 'name': 'b', 'input': {'alist': ['Ex for stream_mode="debug"'], 'another_list': ['hi']}, 'triggers': ['a']}}

{'type': 'task_result', 'timestamp': '2024-06-23T...+00:00', 'step': 2, 'payload': {'id': '...', 'name': 'b', 'result': [('alist', ['there'])]}}


With stream_mode="custom":

>>> from langgraph.types import StreamWriter

...

>>> async def node_a(state: State, writer: StreamWriter):

...     writer({"custom_data": "foo"})

...     return {"alist": ["hi"]}

...

>>> builder = StateGraph(State)

>>> builder.add_node("a", node_a)

>>> builder.add_edge(START, "a")

>>> graph = builder.compile()

...

>>> async for event in graph.astream({"alist": ['Ex for stream_mode="custom"']}, stream_mode="custom"):

...     print(event)

{'custom_data': 'foo'}


With stream_mode="messages":

>>> from typing_extensions import Annotated, TypedDict

>>> from langgraph.graph import StateGraph, START

>>> from langchain_openai import ChatOpenAI

...

>>> llm = ChatOpenAI(model="gpt-4o-mini")

...

>>> class State(TypedDict):

...     question: str

...     answer: str

...

>>> async def node_a(state: State):

...     response = await llm.ainvoke(state["question"])

...     return {"answer": response.content}

...

>>> builder = StateGraph(State)

>>> builder.add_node("a", node_a)

>>> builder.add_edge(START, "a")

>>> graph = builder.compile()



>>> for event in graph.stream({"question": "What is the capital of France?"}, stream_mode="messages"):

...     print(event)

(AIMessageChunk(content='The', additional_kwargs={}, response_metadata={}, id='...'), {'langgraph_step': 1, 'langgraph_node': 'a', 'langgraph_triggers': ['start:a'], 'langgraph_path': ('__pregel_pull', 'a'), 'langgraph_checkpoint_ns': '...', 'checkpoint_ns': '...', 'ls_provider': 'openai', 'ls_model_name': 'gpt-4o-mini', 'ls_model_type': 'chat', 'ls_temperature': 0.7})

(AIMessageChunk(content=' capital', additional_kwargs={}, response_metadata={}, id='...'), {'langgraph_step': 1, 'langgraph_node': 'a', 'langgraph_triggers': ['start:a'], ...})

(AIMessageChunk(content=' of', additional_kwargs={}, response_metadata={}, id='...'), {...})

(AIMessageChunk(content=' France', additional_kwargs={}, response_metadata={}, id='...'), {...})

(AIMessageChunk(content=' is', additional_kwargs={}, response_metadata={}, id='...'), {...})

(AIMessageChunk(content=' Paris', additional_kwargs={}, response_metadata={}, id='...'), {...})

 invoke(input: Union[dict[str, Any], Any], config: Optional[RunnableConfig] = None, *, stream_mode: StreamMode = 'values', output_keys: Optional[Union[str, Sequence[str]]] = None, interrupt_before: Optional[Union[All, Sequence[str]]] = None, interrupt_after: Optional[Union[All, Sequence[str]]] = None, debug: Optional[bool] = None, **kwargs: Any) -> Union[dict[str, Any], Any] ¶

Run the graph with a single input and config.

Parameters:

input (Union[dict[str, Any], Any]) – 

The input data for the graph. It can be a dictionary or any other type.

config (Optional[RunnableConfig], default: None ) – 

Optional. The configuration for the graph run.

stream_mode (StreamMode, default: 'values' ) – 

Optional[str]. The stream mode for the graph run. Default is "values".

output_keys (Optional[Union[str, Sequence[str]]], default: None ) – 

Optional. The output keys to retrieve from the graph run.

interrupt_before (Optional[Union[All, Sequence[str]]], default: None ) – 

Optional. The nodes to interrupt the graph run before.

interrupt_after (Optional[Union[All, Sequence[str]]], default: None ) – 

Optional. The nodes to interrupt the graph run after.

debug (Optional[bool], default: None ) – 

Optional. Enable debug mode for the graph run.

**kwargs (Any, default: {} ) – 

Additional keyword arguments to pass to the graph run.

Returns:

Union[dict[str, Any], Any] – 

The output of the graph run. If stream_mode is "values", it returns the latest output.

Union[dict[str, Any], Any] – 

If stream_mode is not "values", it returns a list of output chunks.

 ainvoke(input: Union[dict[str, Any], Any], config: Optional[RunnableConfig] = None, *, stream_mode: StreamMode = 'values', output_keys: Optional[Union[str, Sequence[str]]] = None, interrupt_before: Optional[Union[All, Sequence[str]]] = None, interrupt_after: Optional[Union[All, Sequence[str]]] = None, debug: Optional[bool] = None, **kwargs: Any) -> Union[dict[str, Any], Any] async ¶

Asynchronously invoke the graph on a single input.

Parameters:

input (Union[dict[str, Any], Any]) – 

The input data for the computation. It can be a dictionary or any other type.

config (Optional[RunnableConfig], default: None ) – 

Optional. The configuration for the computation.

stream_mode (StreamMode, default: 'values' ) – 

Optional. The stream mode for the computation. Default is "values".

output_keys (Optional[Union[str, Sequence[str]]], default: None ) – 

Optional. The output keys to include in the result. Default is None.

interrupt_before (Optional[Union[All, Sequence[str]]], default: None ) – 

Optional. The nodes to interrupt before. Default is None.

interrupt_after (Optional[Union[All, Sequence[str]]], default: None ) – 

Optional. The nodes to interrupt after. Default is None.

debug (Optional[bool], default: None ) – 

Optional. Whether to enable debug mode. Default is None.

**kwargs (Any, default: {} ) – 

Additional keyword arguments.

Returns:

Union[dict[str, Any], Any] – 

The result of the computation. If stream_mode is "values", it returns the latest value.

Union[dict[str, Any], Any] – 

If stream_mode is "chunks", it returns a list of chunks.

 PregelNode ¶

Bases: Runnable

A node in a Pregel graph. This won't be invoked as a runnable by the graph itself, but instead acts as a container for the components necessary to make a PregelExecutableTask for a node.

 channels: Union[list[str], Mapping[str, str]] = channels instance-attribute ¶

The channels that will be passed as input to bound. If a list, the node will be invoked with the first of that isn't empty. If a dict, the keys are the names of the channels, and the values are the keys to use in the input to bound.

 triggers: list[str] = list(triggers) instance-attribute ¶

If any of these channels is written to, this node will be triggered in the next step.

 mapper: Optional[Callable[[Any], Any]] = mapper instance-attribute ¶

A function to transform the input before passing it to bound.

 writers: list[Runnable] = writers or [] instance-attribute ¶

A list of writers that will be executed after bound, responsible for taking the output of bound and writing it to the appropriate channels.

 bound: Runnable[Any, Any] = bound if bound is not None else DEFAULT_BOUND instance-attribute ¶

The main logic of the node. This will be invoked with the input from channels.

 retry_policy: Optional[RetryPolicy] = retry_policy instance-attribute ¶

The retry policy to use when invoking the node.

 tags: Optional[Sequence[str]] = tags instance-attribute ¶

Tags to attach to the node for tracing.

 metadata: Optional[Mapping[str, Any]] = metadata instance-attribute ¶

Metadata to attach to the node for tracing.

 subgraphs: Sequence[PregelProtocol] instance-attribute ¶

Subgraphs used by the node.

 flat_writers: list[Runnable] cached property ¶

Get writers with optimizations applied. Dedupes consecutive ChannelWrites.

 node: Optional[Runnable[Any, Any]] cached property ¶

Get a runnable that combines bound and writers.

Comments
