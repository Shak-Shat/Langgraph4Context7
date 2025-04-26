# How to use MongoDB checkpointer for persistence



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/how-tos/persistence_mongodb/
- **html**: Home
Guides
How-to Guides
LangGraph
Persistence
How to use MongoDB checkpointer for persistence¶

Prerequisites

This guide assumes familiarity with the following:

Persistence
MongoDB

When creating LangGraph agents, you can also set them up so that they persist their state. This allows you to do things like interact with an agent multiple times and have it remember previous interactions.

This reference implementation shows how to use MongoDB as the backend for persisting checkpoint state using the langgraph-checkpoint-mongodb library.

For demonstration purposes we add persistence to a prebuilt ReAct agent.

In general, you can add a checkpointer to any custom graph that you build like this:

from langgraph.graph import StateGraph



builder = StateGraph(...)

# ... define the graph

checkpointer = # mongodb checkpointer (see examples below)

graph = builder.compile(checkpointer=checkpointer)

...

Setup¶

To use the MongoDB checkpointer, you will need a MongoDB cluster. Follow this guide to create a cluster if you don't already have one.

Next, let's install the required packages and set our API keys

%%capture --no-stderr

%pip install -U pymongo langgraph langgraph-checkpoint-mongodb


import getpass

import os





def _set_env(var: str):

    if not os.environ.get(var):

        os.environ[var] = getpass.getpass(f"{var}: ")





_set_env("OPENAI_API_KEY")

OPENAI_API_KEY:  ········


Set up LangSmith for LangGraph development

Sign up for LangSmith to quickly spot issues and improve the performance of your LangGraph projects. LangSmith lets you use trace data to debug, test, and monitor your LLM apps built with LangGraph — read more about how to get started here.

Define model and tools for the graph¶
from typing import Literal



from langchain_core.tools import tool

from langchain_openai import ChatOpenAI

from langgraph.prebuilt import create_react_agent





@tool

def get_weather(city: Literal["nyc", "sf"]):

    """Use this to get weather information."""

    if city == "nyc":

        return "It might be cloudy in nyc"

    elif city == "sf":

        return "It's always sunny in sf"

    else:

        raise AssertionError("Unknown city")





tools = [get_weather]

model = ChatOpenAI(model_name="gpt-4o-mini", temperature=0)


API Reference: tool | ChatOpenAI | create_react_agent

MongoDB checkpointer usage¶
With a connection string¶

This creates a connection to MongoDB directly using the connection string of your cluster. This is ideal for use in scripts, one-off operations and short-lived applications.

from langgraph.checkpoint.mongodb import MongoDBSaver



MONGODB_URI = "localhost:27017"  # replace this with your connection string



with MongoDBSaver.from_conn_string(MONGODB_URI) as checkpointer:

    graph = create_react_agent(model, tools=tools, checkpointer=checkpointer)

    config = {"configurable": {"thread_id": "1"}}

    response = graph.invoke(

        {"messages": [("human", "what's the weather in sf")]}, config

    )

response

{'messages': [HumanMessage(content="what's the weather in sf", additional_kwargs={}, response_metadata={}, id='729afd6a-fdc0-4192-a255-1dac065c79b2'),

  AIMessage(content='', additional_kwargs={'tool_calls': [{'id': 'call_YqaO8oU3BhGmIz9VHTxqGyyN', 'function': {'arguments': '{"city":"sf"}', 'name': 'get_weather'}, 'type': 'function'}], 'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 14, 'prompt_tokens': 57, 'total_tokens': 71, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_name': 'gpt-4o-mini-2024-07-18', 'system_fingerprint': 'fp_39a40c96a0', 'finish_reason': 'tool_calls', 'logprobs': None}, id='run-b45c0c12-c68e-4392-92dd-5d325d0a9f60-0', tool_calls=[{'name': 'get_weather', 'args': {'city': 'sf'}, 'id': 'call_YqaO8oU3BhGmIz9VHTxqGyyN', 'type': 'tool_call'}], usage_metadata={'input_tokens': 57, 'output_tokens': 14, 'total_tokens': 71, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 0}}),

  ToolMessage(content="It's always sunny in sf", name='get_weather', id='0c72eb29-490b-44df-898f-8454c314eac1', tool_call_id='call_YqaO8oU3BhGmIz9VHTxqGyyN'),

  AIMessage(content='The weather in San Francisco is always sunny!', additional_kwargs={'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 10, 'prompt_tokens': 84, 'total_tokens': 94, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_name': 'gpt-4o-mini-2024-07-18', 'system_fingerprint': 'fp_818c284075', 'finish_reason': 'stop', 'logprobs': None}, id='run-33f54c91-0ba9-48b7-9b25-5a972bbdeea9-0', usage_metadata={'input_tokens': 84, 'output_tokens': 10, 'total_tokens': 94, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 0}})]}

Using the MongoDB client¶

This creates a connection to MongoDB using the MongoDB client. This is ideal for long-running applications since it allows you to reuse the client instance for multiple database operations without needing to reinitialize the connection each time.

from pymongo import MongoClient



mongodb_client = MongoClient(MONGODB_URI)



checkpointer = MongoDBSaver(mongodb_client)

graph = create_react_agent(model, tools=tools, checkpointer=checkpointer)

config = {"configurable": {"thread_id": "2"}}

response = graph.invoke({"messages": [("user", "What's the weather in sf?")]}, config)

response

{'messages': [HumanMessage(content="What's the weather in sf?", additional_kwargs={}, response_metadata={}, id='4ce68bee-a843-4b08-9c02-7a0e3b010110'),

  AIMessage(content='', additional_kwargs={'tool_calls': [{'id': 'call_MvGxq9IU9wvW9mfYKSALHtGu', 'function': {'arguments': '{"city":"sf"}', 'name': 'get_weather'}, 'type': 'function'}], 'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 14, 'prompt_tokens': 57, 'total_tokens': 71, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_name': 'gpt-4o-mini-2024-07-18', 'system_fingerprint': 'fp_6fc10e10eb', 'finish_reason': 'tool_calls', 'logprobs': None}, id='run-9712c5a4-376c-4812-a0c4-1b522334a59d-0', tool_calls=[{'name': 'get_weather', 'args': {'city': 'sf'}, 'id': 'call_MvGxq9IU9wvW9mfYKSALHtGu', 'type': 'tool_call'}], usage_metadata={'input_tokens': 57, 'output_tokens': 14, 'total_tokens': 71, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 0}}),

  ToolMessage(content="It's always sunny in sf", name='get_weather', id='b4eed38d-bcaf-4497-ad08-f21ccd6a8c30', tool_call_id='call_MvGxq9IU9wvW9mfYKSALHtGu'),

  AIMessage(content='The weather in San Francisco is always sunny!', additional_kwargs={'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 10, 'prompt_tokens': 84, 'total_tokens': 94, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_name': 'gpt-4o-mini-2024-07-18', 'system_fingerprint': 'fp_6fc10e10eb', 'finish_reason': 'stop', 'logprobs': None}, id='run-c6c4ad75-89ef-4b4f-9ca4-bd52ccb0729b-0', usage_metadata={'input_tokens': 84, 'output_tokens': 10, 'total_tokens': 94, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 0}})]}

# Retrieve the latest checkpoint for the given thread ID

# To retrieve a specific checkpoint, pass the checkpoint_id in the config

checkpointer.get_tuple(config)

CheckpointTuple(config={'configurable': {'thread_id': '2', 'checkpoint_ns': '', 'checkpoint_id': '1efb8c75-9262-68b4-8003-1ac1ef198757'}}, checkpoint={'v': 1, 'ts': '2024-12-12T20:26:20.545003+00:00', 'id': '1efb8c75-9262-68b4-8003-1ac1ef198757', 'channel_values': {'messages': [HumanMessage(content="What's the weather in sf?", additional_kwargs={}, response_metadata={}, id='4ce68bee-a843-4b08-9c02-7a0e3b010110'), AIMessage(content='', additional_kwargs={'tool_calls': [{'id': 'call_MvGxq9IU9wvW9mfYKSALHtGu', 'function': {'arguments': '{"city":"sf"}', 'name': 'get_weather'}, 'type': 'function'}], 'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 14, 'prompt_tokens': 57, 'total_tokens': 71, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_name': 'gpt-4o-mini-2024-07-18', 'system_fingerprint': 'fp_6fc10e10eb', 'finish_reason': 'tool_calls', 'logprobs': None}, id='run-9712c5a4-376c-4812-a0c4-1b522334a59d-0', tool_calls=[{'name': 'get_weather', 'args': {'city': 'sf'}, 'id': 'call_MvGxq9IU9wvW9mfYKSALHtGu', 'type': 'tool_call'}], usage_metadata={'input_tokens': 57, 'output_tokens': 14, 'total_tokens': 71, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 0}}), ToolMessage(content="It's always sunny in sf", name='get_weather', id='b4eed38d-bcaf-4497-ad08-f21ccd6a8c30', tool_call_id='call_MvGxq9IU9wvW9mfYKSALHtGu'), AIMessage(content='The weather in San Francisco is always sunny!', additional_kwargs={'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 10, 'prompt_tokens': 84, 'total_tokens': 94, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_name': 'gpt-4o-mini-2024-07-18', 'system_fingerprint': 'fp_6fc10e10eb', 'finish_reason': 'stop', 'logprobs': None}, id='run-c6c4ad75-89ef-4b4f-9ca4-bd52ccb0729b-0', usage_metadata={'input_tokens': 84, 'output_tokens': 10, 'total_tokens': 94, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 0}})], 'agent': 'agent'}, 'channel_versions': {'__start__': 2, 'messages': 5, 'start:agent': 3, 'agent': 5, 'branch:agent:should_continue:tools': 4, 'tools': 5}, 'versions_seen': {'__input__': {}, '__start__': {'__start__': 1}, 'agent': {'start:agent': 2, 'tools': 4}, 'tools': {'branch:agent:should_continue:tools': 3}}, 'pending_sends': []}, metadata={'source': 'loop', 'writes': {'agent': {'messages': [AIMessage(content='The weather in San Francisco is always sunny!', additional_kwargs={'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 10, 'prompt_tokens': 84, 'total_tokens': 94, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_name': 'gpt-4o-mini-2024-07-18', 'system_fingerprint': 'fp_6fc10e10eb', 'finish_reason': 'stop', 'logprobs': None}, id='run-c6c4ad75-89ef-4b4f-9ca4-bd52ccb0729b-0', usage_metadata={'input_tokens': 84, 'output_tokens': 10, 'total_tokens': 94, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 0}})]}}, 'thread_id': '2', 'step': 3, 'parents': {}}, parent_config={'configurable': {'thread_id': '2', 'checkpoint_ns': '', 'checkpoint_id': '1efb8c75-8d89-6ffe-8002-84a4312c4fed'}}, pending_writes=[])

# Remember to close the connection after you're done

mongodb_client.close()

Using an async connection¶

This creates a short-lived asynchronous connection to MongoDB.

Async connections allow non-blocking database operations. This means other parts of your application can continue running while waiting for database operations to complete. It's particularly useful in high-concurrency scenarios or when dealing with I/O-bound operations.

from langgraph.checkpoint.mongodb.aio import AsyncMongoDBSaver



async with AsyncMongoDBSaver.from_conn_string(MONGODB_URI) as checkpointer:

    graph = create_react_agent(model, tools=tools, checkpointer=checkpointer)

    config = {"configurable": {"thread_id": "3"}}

    response = await graph.ainvoke(

        {"messages": [("user", "What's the weather in sf?")]}, config

    )

response

{'messages': [HumanMessage(content="What's the weather in sf?", additional_kwargs={}, response_metadata={}, id='fed70fe6-1b2e-4481-9bfc-063df3b587dc'),

  AIMessage(content='', additional_kwargs={'tool_calls': [{'id': 'call_miRiF3vPQv98wlDHl6CeRxBy', 'function': {'arguments': '{"city":"sf"}', 'name': 'get_weather'}, 'type': 'function'}], 'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 14, 'prompt_tokens': 57, 'total_tokens': 71, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_name': 'gpt-4o-mini-2024-07-18', 'system_fingerprint': 'fp_6fc10e10eb', 'finish_reason': 'tool_calls', 'logprobs': None}, id='run-7f2d5153-973e-4a9e-8b71-a77625c342cf-0', tool_calls=[{'name': 'get_weather', 'args': {'city': 'sf'}, 'id': 'call_miRiF3vPQv98wlDHl6CeRxBy', 'type': 'tool_call'}], usage_metadata={'input_tokens': 57, 'output_tokens': 14, 'total_tokens': 71, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 0}}),

  ToolMessage(content="It's always sunny in sf", name='get_weather', id='49035e8e-8aee-4d9d-88ab-9a1bc10ecbd3', tool_call_id='call_miRiF3vPQv98wlDHl6CeRxBy'),

  AIMessage(content='The weather in San Francisco is always sunny!', additional_kwargs={'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 10, 'prompt_tokens': 84, 'total_tokens': 94, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_name': 'gpt-4o-mini-2024-07-18', 'system_fingerprint': 'fp_6fc10e10eb', 'finish_reason': 'stop', 'logprobs': None}, id='run-9403d502-391e-4407-99fd-eec8ed184e50-0', usage_metadata={'input_tokens': 84, 'output_tokens': 10, 'total_tokens': 94, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 0}})]}

Using the async MongoDB client¶

This routes connections to MongoDB through an asynchronous MongoDB client.

from pymongo import AsyncMongoClient



async_mongodb_client = AsyncMongoClient(MONGODB_URI)



checkpointer = AsyncMongoDBSaver(async_mongodb_client)

graph = create_react_agent(model, tools=tools, checkpointer=checkpointer)

config = {"configurable": {"thread_id": "4"}}

response = await graph.ainvoke(

    {"messages": [("user", "What's the weather in sf?")]}, config

)

response

{'messages': [HumanMessage(content="What's the weather in sf?", additional_kwargs={}, response_metadata={}, id='58282e2b-4cc1-40a1-8e65-420a2177bbd6'),

  AIMessage(content='', additional_kwargs={'tool_calls': [{'id': 'call_SJFViVHl1tYTZDoZkNN3ePhJ', 'function': {'arguments': '{"city":"sf"}', 'name': 'get_weather'}, 'type': 'function'}], 'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 14, 'prompt_tokens': 57, 'total_tokens': 71, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_name': 'gpt-4o-mini-2024-07-18', 'system_fingerprint': 'fp_bba3c8e70b', 'finish_reason': 'tool_calls', 'logprobs': None}, id='run-131af8c1-d388-4d7f-9137-da59ebd5fefd-0', tool_calls=[{'name': 'get_weather', 'args': {'city': 'sf'}, 'id': 'call_SJFViVHl1tYTZDoZkNN3ePhJ', 'type': 'tool_call'}], usage_metadata={'input_tokens': 57, 'output_tokens': 14, 'total_tokens': 71, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 0}}),

  ToolMessage(content="It's always sunny in sf", name='get_weather', id='6090a56f-177b-4d3f-b16a-9c05f23800e3', tool_call_id='call_SJFViVHl1tYTZDoZkNN3ePhJ'),

  AIMessage(content='The weather in San Francisco is always sunny!', additional_kwargs={'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 10, 'prompt_tokens': 84, 'total_tokens': 94, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_name': 'gpt-4o-mini-2024-07-18', 'system_fingerprint': 'fp_6fc10e10eb', 'finish_reason': 'stop', 'logprobs': None}, id='run-6ff5ddf5-6e13-4126-8df9-81c8638355fc-0', usage_metadata={'input_tokens': 84, 'output_tokens': 10, 'total_tokens': 94, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 0}})]}


# Retrieve the latest checkpoint for the given thread ID

# To retrieve a specific checkpoint, pass the checkpoint_id in the config

latest_checkpoint = await checkpointer.aget_tuple(config)

print(latest_checkpoint)

CheckpointTuple(config={'configurable': {'thread_id': '4', 'checkpoint_ns': '', 'checkpoint_id': '1efb8c76-21f4-6d10-8003-9496e1754e93'}}, checkpoint={'v': 1, 'ts': '2024-12-12T20:26:35.599560+00:00', 'id': '1efb8c76-21f4-6d10-8003-9496e1754e93', 'channel_values': {'messages': [HumanMessage(content="What's the weather in sf?", additional_kwargs={}, response_metadata={}, id='58282e2b-4cc1-40a1-8e65-420a2177bbd6'), AIMessage(content='', additional_kwargs={'tool_calls': [{'id': 'call_SJFViVHl1tYTZDoZkNN3ePhJ', 'function': {'arguments': '{"city":"sf"}', 'name': 'get_weather'}, 'type': 'function'}], 'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 14, 'prompt_tokens': 57, 'total_tokens': 71, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_name': 'gpt-4o-mini-2024-07-18', 'system_fingerprint': 'fp_bba3c8e70b', 'finish_reason': 'tool_calls', 'logprobs': None}, id='run-131af8c1-d388-4d7f-9137-da59ebd5fefd-0', tool_calls=[{'name': 'get_weather', 'args': {'city': 'sf'}, 'id': 'call_SJFViVHl1tYTZDoZkNN3ePhJ', 'type': 'tool_call'}], usage_metadata={'input_tokens': 57, 'output_tokens': 14, 'total_tokens': 71, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 0}}), ToolMessage(content="It's always sunny in sf", name='get_weather', id='6090a56f-177b-4d3f-b16a-9c05f23800e3', tool_call_id='call_SJFViVHl1tYTZDoZkNN3ePhJ'), AIMessage(content='The weather in San Francisco is always sunny!', additional_kwargs={'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 10, 'prompt_tokens': 84, 'total_tokens': 94, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_name': 'gpt-4o-mini-2024-07-18', 'system_fingerprint': 'fp_6fc10e10eb', 'finish_reason': 'stop', 'logprobs': None}, id='run-6ff5ddf5-6e13-4126-8df9-81c8638355fc-0', usage_metadata={'input_tokens': 84, 'output_tokens': 10, 'total_tokens': 94, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 0}})], 'agent': 'agent'}, 'channel_versions': {'__start__': 2, 'messages': 5, 'start:agent': 3, 'agent': 5, 'branch:agent:should_continue:tools': 4, 'tools': 5}, 'versions_seen': {'__input__': {}, '__start__': {'__start__': 1}, 'agent': {'start:agent': 2, 'tools': 4}, 'tools': {'branch:agent:should_continue:tools': 3}}, 'pending_sends': []}, metadata={'source': 'loop', 'writes': {'agent': {'messages': [AIMessage(content='The weather in San Francisco is always sunny!', additional_kwargs={'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 10, 'prompt_tokens': 84, 'total_tokens': 94, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_name': 'gpt-4o-mini-2024-07-18', 'system_fingerprint': 'fp_6fc10e10eb', 'finish_reason': 'stop', 'logprobs': None}, id='run-6ff5ddf5-6e13-4126-8df9-81c8638355fc-0', usage_metadata={'input_tokens': 84, 'output_tokens': 10, 'total_tokens': 94, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 0}})]}}, 'thread_id': '4', 'step': 3, 'parents': {}}, parent_config={'configurable': {'thread_id': '4', 'checkpoint_ns': '', 'checkpoint_id': '1efb8c76-1c6c-6474-8002-9c2595cd481c'}}, pending_writes=[])


# Remember to close the connection after you're done

await async_mongodb_client.close()

Comments
