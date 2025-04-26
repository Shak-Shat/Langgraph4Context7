# Stateless Runs in LangGraph

## Overview
Stateless runs provide a way to execute LangGraph workflows without persisting state between interactions. Unlike the typical approach that uses a `thread_id` to maintain conversation context, stateless runs are independent executions that don't require persistence, making them useful for one-off interactions or when state management isn't necessary.

## Key Concepts
- Stateless execution of LangGraph workflows
- No persistence of conversation history between runs
- Simplified execution for one-time interactions
- Compatible with streaming and synchronous execution patterns

## Prerequisites
- A deployed LangGraph application
- LangGraph SDK for client interaction
- Understanding of basic LangGraph concepts

## Implementation

### Setting Up the Client

```python
from langgraph_sdk import get_client

# Initialize client with deployment URL
client = get_client(url="<DEPLOYMENT_URL>")

# Reference the deployed graph by its name
assistant_id = "agent"

# Create a thread object (though we won't use thread_id for stateless runs)
thread = await client.threads.create()
```

### Executing Stateless Streaming Runs

For streaming output from a stateless run, use the `stream` method with `thread_id=None`:

```python
# Prepare input for the graph
input = {
    "messages": [
        {"role": "user", "content": "Hello! My name is Bagatur and I am 26 years old."}
    ]
}

# Stream results with thread_id=None for stateless execution
async for chunk in client.runs.stream(
    None,  # Passing None instead of thread_id makes the run stateless
    assistant_id,
    input=input,
    stream_mode="updates",
):
    if chunk.data and "run_id" not in chunk.data:
        print(chunk.data)
```

### Executing Synchronous Stateless Runs

For a non-streaming stateless run that returns the final result, use the `wait` method:

```python
# Execute stateless run and wait for completion
stateless_run_result = await client.runs.wait(
    None,  # Passing None instead of thread_id makes the run stateless
    assistant_id,
    input=input,
)

print(stateless_run_result)
```

## Usage Example

```python
from langgraph_sdk import get_client

async def stateless_interaction():
    # Initialize client
    client = get_client(url="https://example-deployment.langchain.dev")
    
    # Define the assistant to use
    assistant_id = "customer_support_agent"
    
    # Define input for a one-time query
    input_data = {
        "messages": [
            {"role": "user", "content": "What are your business hours?"}
        ]
    }
    
    # Option 1: Stream the response
    print("Streaming response:")
    async for chunk in client.runs.stream(
        None,
        assistant_id,
        input=input_data,
        stream_mode="updates",
    ):
        if chunk.data and "run_id" not in chunk.data:
            print(chunk.data)
    
    # Option 2: Wait for complete response
    print("\nComplete response:")
    complete_response = await client.runs.wait(
        None,
        assistant_id,
        input=input_data,
    )
    print(complete_response)

# Run the example
import asyncio
asyncio.run(stateless_interaction())
```

## Benefits
- Reduced resource usage by avoiding state persistence
- Simplified architecture for one-off interactions
- No need to manage thread lifecycle
- Faster execution without state loading/saving operations
- Compatible with both streaming and synchronous execution patterns

## Considerations
- No conversation history between stateless runs
- Not suitable for multi-turn conversations requiring context
- Each run starts with a clean slate without previous context
- May result in repetitive information requests if used for interactions that would benefit from context

## Metadata

- **url**: https://langchain-ai.github.io/langgraph/cloud/how-tos/stateless_runs/
- **html**: Home
Guides
How-to Guides
LangGraph Platform
Runs
Stateless Runs¶

Most of the time, you provide a thread_id to your client when you run your graph in order to keep track of prior runs through the persistent state implemented in LangGraph Cloud. However, if you don't need to persist the runs you don't need to use the built in persistent state and can create stateless runs.

Setup¶

First, let's setup our client:

Python
Javascript
CURL
from langgraph_sdk import get_client



client = get_client(url=<DEPLOYMENT_URL>)

# Using the graph deployed with the name "agent"

assistant_id = "agent"

# create thread

thread = await client.threads.create()

Stateless streaming¶

We can stream the results of a stateless run in an almost identical fashion to how we stream from a run with the state attribute, but instead of passing a value to the thread_id parameter, we pass None:

Python
Javascript
CURL
input = {

    "messages": [

        {"role": "user", "content": "Hello! My name is Bagatur and I am 26 years old."}

    ]

}



async for chunk in client.runs.stream(

    # Don't pass in a thread_id and the stream will be stateless

    None,

    assistant_id,

    input=input,

    stream_mode="updates",

):

    if chunk.data and "run_id" not in chunk.data:

        print(chunk.data)


Output:

{'agent': {'messages': [{'content': "Hello Bagatur! It's nice to meet you. Thank you for introducing yourself and sharing your age. Is there anything specific you'd like to know or discuss? I'm here to help with any questions or topics you're interested in.", 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'ai', 'name': None, 'id': 'run-489ec573-1645-4ce2-a3b8-91b391d50a71', 'example': False, 'tool_calls': [], 'invalid_tool_calls': [], 'usage_metadata': None}]}}

Waiting for stateless results¶

In addition to streaming, you can also wait for a stateless result by using the .wait function like follows:

Python
Javascript
CURL
stateless_run_result = await client.runs.wait(

    None,

    assistant_id,

    input=input,

)

print(stateless_run_result)


