# LangGraph SDK



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/concepts/sdk/
- **html**: Home
Guides
Concepts
LangGraph Platform
Components
LangGraph SDK¶

Prerequisites

LangGraph Platform
LangGraph Server

The LangGraph Platform provides both a Python and JS SDK for interacting with the LangGraph Server API.

Installation¶

You can install the packages using the appropriate package manager for your language.

Python
JS
pip install langgraph-sdk

API Reference¶

You can find the API reference for the SDKs here:

Python SDK Reference
JS/TS SDK Reference
Python Sync vs. Async¶

The Python SDK provides both synchronous (get_sync_client) and asynchronous (get_client) clients for interacting with the LangGraph Server API.

Async
Sync
from langgraph_sdk import get_client



client = get_client(url=..., api_key=...)

await client.assistants.search()

Related¶
LangGraph CLI API Reference
Python SDK Reference
JS/TS SDK Reference
Comments
