# How to Kick Off Background Runs in LangGraph

## Overview
This guide explains how to initiate and manage background runs for LangGraph agents, which is particularly useful for long-running processes that need to operate asynchronously.

## Key Concepts
- Background runs for asynchronous agent execution
- Thread creation and management
- Run status monitoring and state inspection
- Thread state persistence and retrieval

## Prerequisites
```python
# Import the necessary client library
from langgraph_sdk import get_client

# Connect to your LangGraph deployment
client = get_client(url=<DEPLOYMENT_URL>)
```

## Implementation

### Creating a Thread
First, create a thread to manage the conversation and run history:

```python
# Create a new thread using the deployed agent
assistant_id = "agent"
thread = await client.threads.create()

# The response contains thread information
print(thread)
```

Output:
```json
{
    "thread_id": "5cb1e8a1-34b3-4a61-a34e-71a9799bd00d",
    "created_at": "2024-08-30T20:35:52.062934+00:00",
    "updated_at": "2024-08-30T20:35:52.062934+00:00",
    "metadata": {},
    "status": "idle",
    "config": {},
    "values": null
}
```

### Checking Current Runs
Before starting a new run, you can check if there are any existing runs on the thread:

```python
# List all runs on the thread
runs = await client.runs.list(thread["thread_id"])
print(runs)
```

A newly created thread will have no runs:
```
[]
```

### Starting a Background Run
To kick off a background run with a user query:

```python
# Prepare input with the user message
input = {"messages": [{"role": "user", "content": "what's the weather in sf"}]}

# Create a new run on the thread
run = await client.runs.create(thread["thread_id"], assistant_id, input=input)
```

### Monitoring Run Status
Check the run's status immediately after creation:

```python
# Get the current run status
print(await client.runs.get(thread["thread_id"], run["run_id"]))
```

Initially, the run will show a `pending` status:
```json
{
    "run_id": "1ef6a5f8-bd86-6763-bbd6-bff042db7b1b",
    "thread_id": "7885f0cf-94ad-4040-91d7-73f7ba007c8a",
    "assistant_id": "fe096781-5601-53d2-b2f6-0d3403f7e9ca",
    "created_at": "2024-09-04T01:46:47.244887+00:00",
    "updated_at": "2024-09-04T01:46:47.244887+00:00",
    "metadata": {},
    "status": "pending",
    "kwargs": {
        "input": {
            "messages": [
                {
                    "role": "user",
                    "content": "what's the weather in sf"
                }
            ]
        },
        "config": {
            "metadata": {
                "created_by": "system"
            },
            "configurable": {
                "run_id": "1ef6a5f8-bd86-6763-bbd6-bff042db7b1b",
                "user_id": "",
                "graph_id": "agent",
                "thread_id": "7885f0cf-94ad-4040-91d7-73f7ba007c8a",
                "assistant_id": "fe096781-5601-53d2-b2f6-0d3403f7e9ca",
                "checkpoint_id": null
            }
        },
        "webhook": null,
        "temporary": false,
        "stream_mode": [
            "values"
        ],
        "feedback_keys": null,
        "interrupt_after": null,
        "interrupt_before": null
    },
    "multitask_strategy": "reject"
}
```

### Waiting for Run Completion
To wait for the run to complete:

```python
# Wait for the run to finish
await client.runs.join(thread["thread_id"], run["run_id"])

# Check the final status
print(await client.runs.get(thread["thread_id"], run["run_id"]))
```

When complete, the status will change to `success`:
```json
{
    "run_id": "1ef6a5f8-bd86-6763-bbd6-bff042db7b1b",
    "thread_id": "7885f0cf-94ad-4040-91d7-73f7ba007c8a",
    "assistant_id": "fe096781-5601-53d2-b2f6-0d3403f7e9ca",
    "created_at": "2024-09-04T01:46:47.244887+00:00",
    "updated_at": "2024-09-04T01:46:47.244887+00:00",
    "metadata": {},
    "status": "success",
    "kwargs": {
        "input": {
            "messages": [
                {
                    "role": "user",
                    "content": "what's the weather in sf"
                }
            ]
        },
        "config": {
            "metadata": {
                "created_by": "system"
            },
            "configurable": {
                "run_id": "1ef6a5f8-bd86-6763-bbd6-bff042db7b1b",
                "user_id": "",
                "graph_id": "agent",
                "thread_id": "7885f0cf-94ad-4040-91d7-73f7ba007c8a",
                "assistant_id": "fe096781-5601-53d2-b2f6-0d3403f7e9ca",
                "checkpoint_id": null
            }
        },
        "webhook": null,
        "temporary": false,
        "stream_mode": [
            "values"
        ],
        "feedback_keys": null,
        "interrupt_after": null,
        "interrupt_before": null
    },
    "multitask_strategy": "reject"
}
```

### Retrieving Final Results
After the run completes, you can access the final state of the thread:

```python
# Get the current state of the thread
final_result = await client.threads.get_state(thread["thread_id"])
print(final_result)
```

To extract just the agent's response:
```python
# Get the content of the last AI message
print(final_result['values']['messages'][-1]['content'][0]['text'])
```

Output:
```
The search results provide the current weather conditions in San Francisco. According to the data, as of 2:00 PM on August 30, 2024, the temperature in San Francisco is 70°F (21.1°C) with partly cloudy skies. The wind is blowing from the west-northwest at around 12 mph (19 km/h). The humidity is 59% and visibility is 9 miles (16 km). Overall, it looks like a nice late summer day in San Francisco with comfortable temperatures and partly sunny conditions.
```

## Benefits
- Asynchronous processing for long-running operations
- Persistent thread state across multiple interactions
- Status monitoring for tracking run progress
- Ability to access detailed run information and results
- Support for complex multi-step agent processes

## Considerations
- Background runs are ideal for operations that take significant time to complete
- Implement proper error handling for run failures
- Use thread and run IDs to maintain context across user sessions
- Monitor run status to provide appropriate feedback to users
- Consider implementing webhooks for notification when runs complete
