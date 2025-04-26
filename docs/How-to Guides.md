# How-to Guides



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/how-tos/?q=
- **html**: Home
Guides
How-to Guides
How-to Guides¶

Here you’ll find answers to “How do I...?” types of questions. These guides are goal-oriented and concrete; they're meant to help you complete a specific task. For conceptual explanations see the Conceptual guide. For end-to-end walk-throughs see Tutorials. For comprehensive descriptions of every class and function see the API Reference.

LangGraph¶
Graph API Basics¶
How to update graph state from nodes
How to create a sequence of steps
How to create branches for parallel execution
How to create and control loops with recursion limits
How to visualize your graph
Fine-grained Control¶

These guides demonstrate LangGraph features that grant fine-grained control over the execution of your graph.

How to create map-reduce branches for parallel execution
How to update state and jump to nodes in graphs and subgraphs
How to add runtime configuration to your graph
How to add node retries
How to return state before hitting recursion limit
Persistence¶

LangGraph Persistence makes it easy to persist state across graph runs (per-thread persistence) and across threads (cross-thread persistence). These how-to guides show how to add persistence to your graph.

How to add thread-level persistence to your graph
How to add thread-level persistence to a subgraph
How to add cross-thread persistence to your graph
How to use Postgres checkpointer for persistence
How to use MongoDB checkpointer for persistence
How to create a custom checkpointer using Redis

See the below guides for how-to add persistence to your workflow using the Functional API:

How to add thread-level persistence (functional API)
How to add cross-thread persistence (functional API)
Memory¶

LangGraph makes it easy to manage conversation memory in your graph. These how-to guides show how to implement different strategies for that.

How to manage conversation history
How to delete messages
How to add summary conversation memory
How to add long-term memory (cross-thread)
How to use semantic search for long-term memory
Human-in-the-loop¶

Human-in-the-loop functionality allows you to involve humans in the decision-making process of your graph. These how-to guides show how to implement human-in-the-loop workflows in your graph.

Key workflows:

How to wait for user input: A basic example that shows how to implement a human-in-the-loop workflow in your graph using the interrupt function.
How to review tool calls: Incorporate human-in-the-loop for reviewing/editing/accepting tool call requests before they executed using the interrupt function.

Other methods:

How to add static breakpoints: Use for debugging purposes. For human-in-the-loop workflows, we recommend the interrupt function instead.
How to edit graph state: Edit graph state using graph.update_state method. Use this if implementing a human-in-the-loop workflow via static breakpoints.
How to add dynamic breakpoints with NodeInterrupt: Not recommended: Use the interrupt function instead.

See the below guides for how-to implement human-in-the-loop workflows with the Functional API:

How to wait for user input (Functional API)
How to review tool calls (Functional API)
Time Travel¶

Time travel allows you to replay past actions in your LangGraph application to explore alternative paths and debug issues. These how-to guides show how to use time travel in your graph.

How to view and update past graph state
Streaming¶

Streaming is crucial for enhancing the responsiveness of applications built on LLMs. By displaying output progressively, even before a complete response is ready, streaming significantly improves user experience (UX), particularly when dealing with the latency of LLMs.

How to stream
How to stream LLM tokens
How to stream LLM tokens from specific nodes
How to stream data from within a tool
How to stream from subgraphs
How to disable streaming for models that don't support it
Tool calling¶

Tool calling is a type of chat model API that accepts tool schemas, along with messages, as input and returns invocations of those tools as part of the output message.

These how-to guides show common patterns for tool calling with LangGraph:

How to call tools using ToolNode
How to handle tool calling errors
How to pass runtime values to tools
How to pass config to tools
How to update graph state from tools
How to handle large numbers of tools
Subgraphs¶

Subgraphs allow you to reuse an existing graph from another graph. These how-to guides show how to use subgraphs:

How to use subgraphs
How to view and update state in subgraphs
How to transform inputs and outputs of a subgraph
Multi-agent¶

Multi-agent systems are useful to break down complex LLM applications into multiple agents, each responsible for a different part of the application. These how-to guides show how to implement multi-agent systems in LangGraph:

How to implement handoffs between agents
How to build a multi-agent network
How to add multi-turn conversation in a multi-agent application

See the multi-agent tutorials for implementations of other multi-agent architectures.

See the below guides for how to implement multi-agent workflows with the Functional API:

How to build a multi-agent network (functional API)
How to add multi-turn conversation in a multi-agent application (functional API)
State Management¶
How to use Pydantic model as graph state
How to define input/output schema for your graph
How to pass private state between nodes inside the graph
Other¶
How to run graph asynchronously
How to force tool-calling agent to structure output
How to pass custom LangSmith run ID for graph runs
How to integrate LangGraph with AutoGen, CrewAI, and other frameworks

See the below guide for how to integrate with other frameworks using the Functional API:

How to integrate LangGraph (functional API) with AutoGen, CrewAI, and other frameworks
Prebuilt ReAct Agent¶

The LangGraph prebuilt ReAct agent is pre-built implementation of a tool calling agent.

One of the big benefits of LangGraph is that you can easily create your own agent architectures. So while it's fine to start here to build an agent quickly, we would strongly recommend learning how to build your own agent so that you can take full advantage of LangGraph.

These guides show how to use the prebuilt ReAct agent:

How to use the pre-built ReAct agent
How to add thread-level memory to a ReAct Agent
How to add a custom system prompt to a ReAct agent
How to add human-in-the-loop processes to a ReAct agent
How to return structured output from a ReAct agent
How to add semantic search for long-term memory to a ReAct agent

Interested in further customizing the ReAct agent? This guide provides an overview of its underlying implementation to help you customize for your own needs:

How to create prebuilt ReAct agent from scratch

See the below guide for how-to build ReAct agents with the Functional API:

How to create a ReAct agent from scratch (Functional API)
LangGraph Platform¶

This section includes how-to guides for LangGraph Platform.

LangGraph Platform is a commercial solution for deploying agentic applications in production, built on the open-source LangGraph framework.

The LangGraph Platform offers a few different deployment options described in the deployment options guide.

Tip

LangGraph is an MIT-licensed open-source library, which we are committed to maintaining and growing for the community.
You can always deploy LangGraph applications on your own infrastructure using the open-source LangGraph project without using LangGraph Platform.
Application Structure¶

Learn how to set up your app for deployment to LangGraph Platform:

How to set up app for deployment (requirements.txt)
How to set up app for deployment (pyproject.toml)
How to set up app for deployment (JavaScript)
How to add semantic search
How to customize Dockerfile
How to test locally
How to rebuild graph at runtime
How to use LangGraph Platform to deploy CrewAI, AutoGen, and other frameworks
How to integrate LangGraph into your React application
Deployment¶

LangGraph applications can be deployed using LangGraph Cloud, which provides a range of services to help you deploy, manage, and scale your applications.

How to deploy to LangGraph cloud
How to deploy to a self-hosted environment
How to interact with the deployment using RemoteGraph
Authentication & Access Control¶
How to add custom authentication
How to update the security schema of your OpenAPI spec
Modifying the API¶
How to add custom routes
How to add custom middleware
How to add custom lifespan events
Assistants¶

Assistants is a configured instance of a template.

See SDK Reference for supported endpoints and other details.

How to configure agents
How to version assistants
Threads¶

See SDK Reference for supported endpoints and other details.

How to copy threads
How to check status of your threads
Runs¶

LangGraph Platform supports multiple types of runs besides streaming runs.

How to run an agent in the background
How to run multiple agents in the same thread
How to create cron jobs
How to create stateless runs
Streaming¶

Streaming the results of your LLM application is vital for ensuring a good user experience, especially when your graph may call multiple models and take a long time to fully complete a run. Read about how to stream values from your graph in these how to guides:

How to stream values
How to stream updates
How to stream messages
How to stream events
How to stream in debug mode
How to stream multiple modes
Human-in-the-loop¶

When designing complex graphs, relying entirely on the LLM for decision-making can be risky, particularly when it involves tools that interact with files, APIs, or databases. These interactions may lead to unintended data access or modifications, depending on the use case. To mitigate these risks, LangGraph allows you to integrate human-in-the-loop behavior, ensuring your LLM applications operate as intended without undesirable outcomes.

How to add a breakpoint
How to wait for user input
How to edit graph state
How to replay and branch from prior states
How to review tool calls
Double-texting¶

Graph execution can take a while, and sometimes users may change their mind about the input they wanted to send before their original input has finished running. For example, a user might notice a typo in their original request and will edit the prompt and resend it. Deciding what to do in these cases is important for ensuring a smooth user experience and preventing your graphs from behaving in unexpected ways.

How to use the interrupt option
How to use the rollback option
How to use the reject option
How to use the enqueue option
Webhooks¶
How to integrate webhooks
Cron Jobs¶
How to create cron jobs
LangGraph Studio¶

LangGraph Studio is a built-in UI for visualizing, testing, and debugging your agents.

How to connect to a LangGraph Platform deployment
How to connect to a local dev server
How to connect to a local deployment (Docker)
How to interact with threads in LangGraph Studio
How to add nodes as dataset examples in LangGraph Studio
How to engineer prompts in LangGraph Studio
Troubleshooting¶

These are the guides for resolving common errors you may find while building with LangGraph. Errors referenced below will have an lc_error_code property corresponding to one of the below codes when they are thrown in code.

GRAPH_RECURSION_LIMIT
INVALID_CONCURRENT_GRAPH_UPDATE
INVALID_GRAPH_NODE_RETURN_VALUE
MULTIPLE_SUBGRAPHS
INVALID_CHAT_HISTORY
LangGraph Platform Troubleshooting¶

These guides provide troubleshooting information for errors that are specific to the LangGraph Platform.

INVALID_LICENSE
Comments
