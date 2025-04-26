# Double Texting



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/concepts/double_texting/
- **html**: Home
Guides
Concepts
LangGraph Platform
LangGraph Server
Double Texting¶

Prerequisites

LangGraph Server

Many times users might interact with your graph in unintended ways. For instance, a user may send one message and before the graph has finished running send a second message. More generally, users may invoke the graph a second time before the first run has finished. We call this "double texting".

Currently, LangGraph only addresses this as part of LangGraph Platform, not in the open source. The reason for this is that in order to handle this we need to know how the graph is deployed, and since LangGraph Platform deals with deployment the logic needs to live there. If you do not want to use LangGraph Platform, we describe the options we have implemented in detail below.

Reject¶

This is the simplest option, this just rejects any follow-up runs and does not allow double texting. See the how-to guide for configuring the reject double text option.

Enqueue¶

This is a relatively simple option which continues the first run until it completes the whole run, then sends the new input as a separate run. See the how-to guide for configuring the enqueue double text option.

Interrupt¶

This option interrupts the current execution but saves all the work done up until that point. It then inserts the user input and continues from there.

If you enable this option, your graph should be able to handle weird edge cases that may arise. For example, you could have called a tool but not yet gotten back a result from running that tool. You may need to remove that tool call in order to not have a dangling tool call.

See the how-to guide for configuring the interrupt double text option.

Rollback¶

This option interrupts the current execution AND rolls back all work done up until that point, including the original run input. It then sends the new user input in, basically as if it was the original input.

See the how-to guide for configuring the rollback double text option.

Comments
