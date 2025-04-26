# Channels



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/reference/channels/
- **html**: API reference
Library
Channels¶
 BaseChannel ¶

Bases: Generic[Value, Update, C], ABC

 ValueType: Any abstractmethod property ¶

The type of the value stored in the channel.

 UpdateType: Any abstractmethod property ¶

The type of the update received by the channel.

 checkpoint() -> Optional[C] ¶

Return a serializable representation of the channel's current state. Raises EmptyChannelError if the channel is empty (never updated yet), or doesn't support checkpoints.

 from_checkpoint(checkpoint: Optional[C]) -> Self abstractmethod ¶

Return a new identical channel, optionally initialized from a checkpoint. If the checkpoint contains complex data structures, they should be copied.

 update(values: Sequence[Update]) -> bool abstractmethod ¶

Update the channel's value with the given sequence of updates. The order of the updates in the sequence is arbitrary. This method is called by Pregel for all channels at the end of each step. If there are no updates, it is called with an empty sequence. Raises InvalidUpdateError if the sequence of updates is invalid. Returns True if the channel was updated, False otherwise.

 get() -> Value abstractmethod ¶

Return the current value of the channel.

Raises EmptyChannelError if the channel is empty (never updated yet).

 consume() -> bool ¶

Mark the current value of the channel as consumed. By default, no-op. This is called by Pregel before the start of the next step, for all channels that triggered a node. If the channel was updated, return True.

 Topic ¶

Bases: Generic[Value], BaseChannel[Sequence[Value], Union[Value, list[Value]], tuple[set[Value], list[Value]]]

A configurable PubSub Topic.

Parameters:

typ (Type[Value]) – 

The type of the value stored in the channel.

accumulate (bool, default: False ) – 

Whether to accumulate values across steps. If False, the channel will be emptied after each step.

 ValueType: Any property ¶

The type of the value stored in the channel.

 UpdateType: Any property ¶

The type of the update received by the channel.

 consume() -> bool ¶

Mark the current value of the channel as consumed. By default, no-op. This is called by Pregel before the start of the next step, for all channels that triggered a node. If the channel was updated, return True.

 LastValue ¶

Bases: Generic[Value], BaseChannel[Value, Value, Value]

Stores the last value received, can receive at most one value per step.

 ValueType: Type[Value] property ¶

The type of the value stored in the channel.

 UpdateType: Type[Value] property ¶

The type of the update received by the channel.

 checkpoint() -> Optional[C] ¶

Return a serializable representation of the channel's current state. Raises EmptyChannelError if the channel is empty (never updated yet), or doesn't support checkpoints.

 consume() -> bool ¶

Mark the current value of the channel as consumed. By default, no-op. This is called by Pregel before the start of the next step, for all channels that triggered a node. If the channel was updated, return True.

 EphemeralValue ¶

Bases: Generic[Value], BaseChannel[Value, Value, Value]

Stores the value received in the step immediately preceding, clears after.

 ValueType: Type[Value] property ¶

The type of the value stored in the channel.

 UpdateType: Type[Value] property ¶

The type of the update received by the channel.

 checkpoint() -> Optional[C] ¶

Return a serializable representation of the channel's current state. Raises EmptyChannelError if the channel is empty (never updated yet), or doesn't support checkpoints.

 consume() -> bool ¶

Mark the current value of the channel as consumed. By default, no-op. This is called by Pregel before the start of the next step, for all channels that triggered a node. If the channel was updated, return True.

 BinaryOperatorAggregate ¶

Bases: Generic[Value], BaseChannel[Value, Value, Value]

Stores the result of applying a binary operator to the current value and each new value.

import operator



total = Channels.BinaryOperatorAggregate(int, operator.add)

 ValueType: Type[Value] property ¶

The type of the value stored in the channel.

 UpdateType: Type[Value] property ¶

The type of the update received by the channel.

 checkpoint() -> Optional[C] ¶

Return a serializable representation of the channel's current state. Raises EmptyChannelError if the channel is empty (never updated yet), or doesn't support checkpoints.

 consume() -> bool ¶

Mark the current value of the channel as consumed. By default, no-op. This is called by Pregel before the start of the next step, for all channels that triggered a node. If the channel was updated, return True.

 AnyValue ¶

Bases: Generic[Value], BaseChannel[Value, Value, Value]

Stores the last value received, assumes that if multiple values are received, they are all equal.

 ValueType: Type[Value] property ¶

The type of the value stored in the channel.

 UpdateType: Type[Value] property ¶

The type of the update received by the channel.

 checkpoint() -> Optional[C] ¶

Return a serializable representation of the channel's current state. Raises EmptyChannelError if the channel is empty (never updated yet), or doesn't support checkpoints.

 consume() -> bool ¶

Mark the current value of the channel as consumed. By default, no-op. This is called by Pregel before the start of the next step, for all channels that triggered a node. If the channel was updated, return True.

Comments
