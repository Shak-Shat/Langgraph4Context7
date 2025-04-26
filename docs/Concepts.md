# Concepts



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/concepts/?q=
- **html**: Home
Guides
Concepts
Conceptual Guide¶

This guide provides explanations of the key concepts behind the LangGraph framework and AI applications more broadly.

We recommend that you go through at least the Quickstart before diving into the conceptual guide. This will provide practical context that will make it easier to understand the concepts discussed here.

The conceptual guide does not cover step-by-step instructions or specific implementation examples — those are found in the Tutorials and How-to guides. For detailed reference material, please see the API reference.

LangGraph¶
High Level¶
Why LangGraph?: A high-level overview of LangGraph and its goals.
Concepts¶
LangGraph Glossary: LangGraph workflows are designed as graphs, with nodes representing different components and edges representing the flow of information between them. This guide provides an overview of the key concepts associated with LangGraph graph primitives.
Common Agentic Patterns: An agent uses an LLM to pick its own control flow to solve more complex problems! Agents are a key building block in many LLM applications. This guide explains the different types of agent architectures and how they can be used to control the flow of an application.
Multi-Agent Systems: Complex LLM applications can often be broken down into multiple agents, each responsible for a different part of the application. This guide explains common patterns for building multi-agent systems.
Breakpoints: Breakpoints allow pausing the execution of a graph at specific points. Breakpoints allow stepping through graph execution for debugging purposes.
Human-in-the-Loop: Explains different ways of integrating human feedback into a LangGraph application.
Time Travel: Time travel allows you to replay past actions in your LangGraph application to explore alternative paths and debug issues.
Persistence: LangGraph has a built-in persistence layer, implemented through checkpointers. This persistence layer helps to support powerful capabilities like human-in-the-loop, memory, time travel, and fault-tolerance.
Memory: Memory in AI applications refers to the ability to process, store, and effectively recall information from past interactions. With memory, your agents can learn from feedback and adapt to users' preferences.
Streaming: Streaming is crucial for enhancing the responsiveness of applications built on LLMs. By displaying output progressively, even before a complete response is ready, streaming significantly improves user experience (UX), particularly when dealing with the latency of LLMs.
Functional API: @entrypoint and @task decorators that allow you to add LangGraph functionality to an existing codebase.
Durable Execution: LangGraph's built-in persistence layer provides durable execution for workflows, ensuring that the state of each execution step is saved to a durable store.
Pregel: Pregel is LangGraph's runtime, which is responsible for managing the execution of LangGraph applications.
FAQ: Frequently asked questions about LangGraph.
LangGraph Platform¶

LangGraph Platform is a commercial solution for deploying agentic applications in production, built on the open-source LangGraph framework.

The LangGraph Platform offers a few different deployment options described in the deployment options guide.

Tip

LangGraph is an MIT-licensed open-source library, which we are committed to maintaining and growing for the community.
You can always deploy LangGraph applications on your own infrastructure using the open-source LangGraph project without using LangGraph Platform.
High Level¶
Why LangGraph Platform?: The LangGraph platform is an opinionated way to deploy and manage LangGraph applications. This guide provides an overview of the key features and concepts behind LangGraph Platform.
Platform Architecture: A high-level overview of the architecture of the LangGraph Platform.
Scalability and Resilience: LangGraph Platform is designed to be scalable and resilient. This document explains how the platform achieves this.
Deployment Options: LangGraph Platform offers four deployment options: Self-Hosted Lite, Self-Hosted Enterprise, bring your own cloud (BYOC), and Cloud SaaS. This guide explains the differences between these options, and which Plans they are available on.
Plans: LangGraph Platforms offer three different plans: Developer, Plus, Enterprise. This guide explains the differences between these options, what deployment options are available for each, and how to sign up for each one.
Template Applications: Reference applications designed to help you get started quickly when building with LangGraph.
Components¶

The LangGraph Platform comprises several components that work together to support the deployment and management of LangGraph applications:

LangGraph Server: The LangGraph Server is designed to support a wide range of agentic application use cases, from background processing to real-time interactions.
LangGraph Studio: LangGraph Studio is a specialized IDE that can connect to a LangGraph Server to enable visualization, interaction, and debugging of the application locally.
LangGraph CLI: LangGraph CLI is a command-line interface that helps to interact with a local LangGraph
Python/JS SDK: The Python/JS SDK provides a programmatic way to interact with deployed LangGraph Applications.
Remote Graph: A RemoteGraph allows you to interact with any deployed LangGraph application as though it were running locally.
LangGraph Server¶
Application Structure: A LangGraph application consists of one or more graphs, a LangGraph API Configuration file (langgraph.json), a file that specifies dependencies, and environment variables.
Assistants: Assistants are a way to save and manage different configurations of your LangGraph applications.
Web-hooks: Webhooks allow your running LangGraph application to send data to external services on specific events.
Cron Jobs: Cron jobs are a way to schedule tasks to run at specific times in your LangGraph application.
Double Texting: Double texting is a common issue in LLM applications where users may send multiple messages before the graph has finished running. This guide explains how to handle double texting with LangGraph Deploy.
Authentication & Access Control: Learn about options for authentication and access control when deploying the LangGraph Platform.
Deployment Options¶
Self-Hosted Lite: A free (up to 1 million nodes executed per year), limited version of LangGraph Platform that you can run locally or in a self-hosted manner
Cloud SaaS: Hosted as part of LangSmith.
Bring Your Own Cloud: We manage the infrastructure, so you don't have to, but the infrastructure all runs within your cloud.
Self-Hosted Enterprise: Completely managed by you.
Comments
