# LangGraph Server



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/concepts/langgraph_server/?q=
- **html**: Home
Guides
Concepts
LangGraph Platform
Components
LangGraph Server¶

Prerequisites

LangGraph Platform
LangGraph Glossary
Overview¶

LangGraph Server offers an API for creating and managing agent-based applications. It is built on the concept of assistants, which are agents configured for specific tasks, and includes built-in persistence and a task queue. This versatile API supports a wide range of agentic application use cases, from background processing to real-time interactions.

Key Features¶

The LangGraph Platform incorporates best practices for agent deployment, so you can focus on building your agent logic.

Streaming endpoints: Endpoints that expose multiple different streaming modes. We've made these work even for long-running agents that may go minutes between consecutive stream events.
Background runs: The LangGraph Server supports launching assistants in the background with endpoints for polling the status of the assistant's run and webhooks to monitor run status effectively.
Support for long runs: Our blocking endpoints for running assistants send regular heartbeat signals, preventing unexpected connection closures when handling requests that take a long time to complete.
Task queue: We've added a task queue to make sure we don't drop any requests if they arrive in a bursty nature.
Horizontally scalable infrastructure: LangGraph Server is designed to be horizontally scalable, allowing you to scale up and down your usage as needed.
Double texting support: Many times users might interact with your graph in unintended ways. For instance, a user may send one message and before the graph has finished running send a second message. We call this "double texting" and have added four different ways to handle this.
Optimized checkpointer: LangGraph Platform comes with a built-in checkpointer optimized for LangGraph applications.
Human-in-the-loop endpoints: We've exposed all endpoints needed to support human-in-the-loop features.
Memory: In addition to thread-level persistence (covered above by [checkpointers]l(./persistence.md#checkpoints)), LangGraph Platform also comes with a built-in memory store.
Cron jobs: Built-in support for scheduling tasks, enabling you to automate regular actions like data clean-up or batch processing within your applications.
Webhooks: Allows your application to send real-time notifications and data updates to external systems, making it easy to integrate with third-party services and trigger actions based on specific events.
Monitoring: LangGraph Server integrates seamlessly with the LangSmith monitoring platform, providing real-time insights into your application's performance and health.
What are you deploying?¶

When you deploy a LangGraph Server, you are deploying one or more graphs, a database for persistence, and a task queue.

Graphs¶

When you deploy a graph with LangGraph Server, you are deploying a "blueprint" for an Assistant.

An Assistant is a graph paired with specific configuration settings. You can create multiple assistants per graph, each with unique settings to accommodate different use cases that can be served by the same graph.

Upon deployment, LangGraph Server will automatically create a default assistant for each graph using the graph's default configuration settings.

You can interact with assistants through the LangGraph Server API.

Note

We often think of a graph as implementing an agent, but a graph does not necessarily need to implement an agent. For example, a graph could implement a simple chatbot that only supports back-and-forth conversation, without the ability to influence any application control flow. In reality, as applications get more complex, a graph will often implement a more complex flow that may use multiple agents working in tandem.

Persistence and Task Queue¶

The LangGraph Server leverages a database for persistence and a task queue.

Currently, only Postgres is supported as a database for LangGraph Server and Redis as the task queue.

If you're deploying using LangGraph Cloud, these components are managed for you. If you're deploying LangGraph Server on your own infrastructure, you'll need to set up and manage these components yourself.

Please review the deployment options guide for more information on how these components are set up and managed.

Application Structure¶

To deploy a LangGraph Server application, you need to specify the graph(s) you want to deploy, as well as any relevant configuration settings, such as dependencies and environment variables.

Read the application structure guide to learn how to structure your LangGraph application for deployment.

LangGraph Server API¶

The LangGraph Server API allows you to create and manage assistants, threads, runs, cron jobs, and more.

The LangGraph Cloud API Reference provides detailed information on the API endpoints and data models.

Assistants¶

An Assistant refers to a graph plus specific configuration settings for that graph.

You can think of an assistant as a saved configuration of an agent.

When building agents, it is fairly common to make rapid changes that do not alter the graph logic. For example, simply changing prompts or the LLM selection can have significant impacts on the behavior of the agents. Assistants offer an easy way to make and save these types of changes to agent configuration.

Threads¶

A thread contains the accumulated state of a sequence of runs. If a run is executed on a thread, then the state of the underlying graph of the assistant will be persisted to the thread.

A thread's current and historical state can be retrieved. To persist state, a thread must be created prior to executing a run.

The state of a thread at a particular point in time is called a checkpoint. Checkpoints can be used to restore the state of a thread at a later time.

For more on threads and checkpoints, see this section of the LangGraph conceptual guide.

The LangGraph Cloud API provides several endpoints for creating and managing threads and thread state. See the API reference for more details.

Runs¶

A run is an invocation of an assistant. Each run may have its own input, configuration, and metadata, which may affect execution and output of the underlying graph. A run can optionally be executed on a thread.

The LangGraph Cloud API provides several endpoints for creating and managing runs. See the API reference for more details.

Store¶

Store is an API for managing persistent key-value store that is available from any thread.

Stores are useful for implementing memory in your LangGraph application.

Cron Jobs¶

There are many situations in which it is useful to run an assistant on a schedule.

For example, say that you're building an assistant that runs daily and sends an email summary of the day's news. You could use a cron job to run the assistant every day at 8:00 PM.

LangGraph Cloud supports cron jobs, which run on a user-defined schedule. The user specifies a schedule, an assistant, and some input. After that, on the specified schedule, the server will:

Create a new thread with the specified assistant
Send the specified input to that thread

Note that this sends the same input to the thread every time. See the how-to guide for creating cron jobs.

The LangGraph Cloud API provides several endpoints for creating and managing cron jobs. See the API reference for more details.

Webhooks¶

Webhooks enable event-driven communication from your LangGraph Cloud application to external services. For example, you may want to issue an update to a separate service once an API call to LangGraph Cloud has finished running.

Many LangGraph Cloud endpoints accept a webhook parameter. If this parameter is specified by a an endpoint that can accept POST requests, LangGraph Cloud will send a request at the completion of a run.

See the corresponding how-to guide for more detail.

Related¶
LangGraph Application Structure guide explains how to structure your LangGraph application for deployment.
How-to guides for the LangGraph Platform.
The LangGraph Cloud API Reference provides detailed information on the API endpoints and data models.
Comments
