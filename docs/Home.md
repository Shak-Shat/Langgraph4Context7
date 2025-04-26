# Home



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/?q=
- **html**: 🦜🕸️LangGraph¶

   

⚡ Building language agents as graphs ⚡

Note

Looking for the JS version? See the JS repo and the JS docs.

Overview¶

LangGraph is a library for building stateful, multi-actor applications with LLMs, used to create agent and multi-agent workflows. Check out an introductory tutorial here.

LangGraph is inspired by Pregel and Apache Beam. The public interface draws inspiration from NetworkX. LangGraph is built by LangChain Inc, the creators of LangChain, but can be used without LangChain.

Why use LangGraph?¶

LangGraph powers production-grade agents, trusted by Linkedin, Uber, Klarna, GitLab, and many more. LangGraph provides fine-grained control over both the flow and state of your agent applications. It implements a central persistence layer, enabling features that are common to most agent architectures:

Memory: LangGraph persists arbitrary aspects of your application's state, supporting memory of conversations and other updates within and across user interactions;
Human-in-the-loop: Because state is checkpointed, execution can be interrupted and resumed, allowing for decisions, validation, and corrections at key stages via human input.

Standardizing these components allows individuals and teams to focus on the behavior of their agent, instead of its supporting infrastructure.

Through LangGraph Platform, LangGraph also provides tooling for the development, deployment, debugging, and monitoring of your applications.

LangGraph integrates seamlessly with LangChain and LangSmith (but does not require them).

To learn more about LangGraph, check out our first LangChain Academy course, Introduction to LangGraph, available for free here.

LangGraph Platform¶

LangGraph Platform is infrastructure for deploying LangGraph agents. It is a commercial solution for deploying agentic applications to production, built on the open-source LangGraph framework. The LangGraph Platform consists of several components that work together to support the development, deployment, debugging, and monitoring of LangGraph applications: LangGraph Server (APIs), LangGraph SDKs (clients for the APIs), LangGraph CLI (command line tool for building the server), and LangGraph Studio (UI/debugger).

See deployment options here (includes a free tier).

Here are some common issues that arise in complex deployments, which LangGraph Platform addresses:

Streaming support: LangGraph Server provides multiple streaming modes optimized for various application needs
Background runs: Runs agents asynchronously in the background
Support for long running agents: Infrastructure that can handle long running processes
Double texting: Handle the case where you get two messages from the user before the agent can respond
Handle burstiness: Task queue for ensuring requests are handled consistently without loss, even under heavy loads
Installation¶
pip install -U langgraph

Example¶

Let's build a tool-calling ReAct-style agent that uses a search tool!

pip install langchain-anthropic

export ANTHROPIC_API_KEY=sk-...


Optionally, we can set up LangSmith for best-in-class observability.

export LANGSMITH_TRACING=true

export LANGSMITH_API_KEY=lsv2_sk_...


The simplest way to create a tool-calling agent in LangGraph is to use create_react_agent:

High-level implementation
from langgraph.prebuilt import create_react_agent

from langgraph.checkpoint.memory import MemorySaver

from langchain_anthropic import ChatAnthropic

from langchain_core.tools import tool



# Define the tools for the agent to use

@tool

def search(query: str):

    """Call to surf the web."""

    # This is a placeholder, but don't tell the LLM that...

    if "sf" in query.lower() or "san francisco" in query.lower():

        return "It's 60 degrees and foggy."

    return "It's 90 degrees and sunny."





tools = [search]

model = ChatAnthropic(model="claude-3-5-sonnet-latest", temperature=0)



# Initialize memory to persist state between graph runs

checkpointer = MemorySaver()



app = create_react_agent(model, tools, checkpointer=checkpointer)



# Use the agent

final_state = app.invoke(

    {"messages": [{"role": "user", "content": "what is the weather in sf"}]},

    config={"configurable": {"thread_id": 42}}

)

final_state["messages"][-1].content

"Based on the search results, I can tell you that the current weather in San Francisco is:\n\nTemperature: 60 degrees Fahrenheit\nConditions: Foggy\n\nSan Francisco is known for its microclimates and frequent fog, especially during the summer months. The temperature of 60°F (about 15.5°C) is quite typical for the city, which tends to have mild temperatures year-round. The fog, often referred to as "Karl the Fog" by locals, is a characteristic feature of San Francisco\'s weather, particularly in the mornings and evenings.\n\nIs there anything else you\'d like to know about the weather in San Francisco or any other location?"

Now when we pass the same "thread_id", the conversation context is retained via the saved state (i.e. stored list of messages)
final_state = app.invoke(

    {"messages": [{"role": "user", "content": "what about ny"}]},

    config={"configurable": {"thread_id": 42}}

)

final_state["messages"][-1].content

"Based on the search results, I can tell you that the current weather in New York City is:\n\nTemperature: 90 degrees Fahrenheit (approximately 32.2 degrees Celsius)\nConditions: Sunny\n\nThis weather is quite different from what we just saw in San Francisco. New York is experiencing much warmer temperatures right now. Here are a few points to note:\n\n1. The temperature of 90°F is quite hot, typical of summer weather in New York City.\n2. The sunny conditions suggest clear skies, which is great for outdoor activities but also means it might feel even hotter due to direct sunlight.\n3. This kind of weather in New York often comes with high humidity, which can make it feel even warmer than the actual temperature suggests.\n\nIt's interesting to see the stark contrast between San Francisco's mild, foggy weather and New York's hot, sunny conditions. This difference illustrates how varied weather can be across different parts of the United States, even on the same day.\n\nIs there anything else you'd like to know about the weather in New York or any other location?"


Tip

LangGraph is a low-level framework that allows you to implement any custom agent architectures. Click on the low-level implementation below to see how to implement a tool-calling agent from scratch.

Low-level implementation
Documentation¶
Tutorials: Learn to build with LangGraph through guided examples.
How-to Guides: Accomplish specific things within LangGraph, from streaming, to adding memory & persistence, to common design patterns (branching, subgraphs, etc.), these are the place to go if you want to copy and run a specific code snippet.
Conceptual Guides: In-depth explanations of the key concepts and principles behind LangGraph, such as nodes, edges, state and more.
API Reference: Review important classes and methods, simple examples of how to use the graph and checkpointing APIs, higher-level prebuilt components and more.
LangGraph Platform: LangGraph Platform is a commercial solution for deploying agentic applications in production, built on the open-source LangGraph framework.
Resources¶
Built with LangGraph: Hear how industry leaders use LangGraph to ship powerful, production-ready AI applications.
Contributing¶

For more information on how to contribute, see here.
