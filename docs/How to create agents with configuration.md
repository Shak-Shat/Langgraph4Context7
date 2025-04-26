# How to create agents with configuration



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/cloud/how-tos/configuration_cloud/
- **html**: Home
Guides
How-to Guides
LangGraph Platform
Assistants
How to create agents with configuration¶

One of the benefits of LangGraph API is that it lets you create agents with different configurations. This is useful when you want to:

Define a cognitive architecture once as a LangGraph
Let that LangGraph be configurable across some attributes (for example, system message or LLM to use)
Let users create agents with arbitrary configurations, save them, and then use them in the future

In this guide we will show how to do that for the default agent we have built in.

If you look at the agent we defined, you can see that inside the call_model node we have created the model based on some configuration. That node looks like:

Python
Javascript
def call_model(state, config):

    messages = state["messages"]

    model_name = config.get('configurable', {}).get("model_name", "anthropic")

    model = _get_model(model_name)

    response = model.invoke(messages)

    # We return a list, because this will get added to the existing list

    return {"messages": [response]}


We are looking inside the config for a model_name parameter (which defaults to anthropic if none is found). That means that by default we are using Anthropic as our model provider. In this example we will see an example of how to create an example agent that is configured to use OpenAI.

First let's set up our client and thread:

Python
Javascript
CURL
from langgraph_sdk import get_client



client = get_client(url=<DEPLOYMENT_URL>)

# Select an assistant that is not configured

assistants = await client.assistants.search()

assistant = [a for a in assistants if not a["config"]][0]


We can now call .get_schemas to get schemas associated with this graph:

Python
Javascript
CURL
schemas = await client.assistants.get_schemas(

    assistant_id=assistant["assistant_id"]

)

# There are multiple types of schemas

# We can get the `config_schema` to look at the configurable parameters

print(schemas["config_schema"])


Output:

{
    'model_name': 
        {
            'title': 'Model Name',
            'enum': ['anthropic', 'openai'],
            'type': 'string'
        }
}


Now we can initialize an assistant with config:

Python
Javascript
CURL
openai_assistant = await client.assistants.create(

    # "agent" is the name of a graph we deployed

    "agent", config={"configurable": {"model_name": "openai"}}

)



print(openai_assistant)


Output:

{
    "assistant_id": "62e209ca-9154-432a-b9e9-2d75c7a9219b",
    "graph_id": "agent",
    "created_at": "2024-08-31T03:09:10.230718+00:00",
    "updated_at": "2024-08-31T03:09:10.230718+00:00",
    "config": {
        "configurable": {
            "model_name": "open_ai"
        }
    },
    "metadata": {}
}


We can verify the config is indeed taking effect:

Python
Javascript
CURL
thread = await client.threads.create()

input = {"messages": [{"role": "user", "content": "who made you?"}]}

async for event in client.runs.stream(

    thread["thread_id"],

    openai_assistant["assistant_id"],

    input=input,

    stream_mode="updates",

):

    print(f"Receiving event of type: {event.event}")

    print(event.data)

    print("\n\n")


Output:

Receiving event of type: metadata
{'run_id': '1ef6746e-5893-67b1-978a-0f1cd4060e16'}



Receiving event of type: updates
{'agent': {'messages': [{'content': 'I was created by OpenAI, a research organization focused on developing and advancing artificial intelligence technology.', 'additional_kwargs': {}, 'response_metadata': {'finish_reason': 'stop', 'model_name': 'gpt-4o-2024-05-13', 'system_fingerprint': 'fp_157b3831f5'}, 'type': 'ai', 'name': None, 'id': 'run-e1a6b25c-8416-41f2-9981-f9cfe043f414', 'example': False, 'tool_calls': [], 'invalid_tool_calls': [], 'usage_metadata': None}]}}

Comments
