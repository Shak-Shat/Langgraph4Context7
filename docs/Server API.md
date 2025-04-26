# Server API

## Metadata
- **url**: https://langchain-ai.github.io/langgraph/cloud/reference/api/api_ref/

## Overview
The LangGraph Platform provides a comprehensive REST API for interacting with deployed graph applications. This API allows developers to create, manage, and interact with assistants, threads, and graph runs programmatically.

## Key Concepts
- **API Reference**: Each LangGraph deployment includes interactive API documentation accessible at the `/docs` URL path
- **Authentication**: Access to the API requires authentication using LangSmith API keys
- **Endpoints**: The API provides endpoints for managing assistants, handling conversations, and controlling graph execution

## Authentication
All requests to the LangGraph Cloud API must include authentication using the `X-Api-Key` header. The value should be a valid LangSmith API key for the organization where the API is deployed.

## Common Endpoints

### Assistants Management
- **GET /assistants**: List all available assistants
- **POST /assistants/search**: Search for assistants with specific criteria
- **GET /assistants/{assistant_id}**: Retrieve details of a specific assistant
- **PUT /assistants/{assistant_id}**: Update an existing assistant
- **DELETE /assistants/{assistant_id}**: Delete an assistant

### Threads and Messages
- **POST /threads**: Create a new conversation thread
- **GET /threads/{thread_id}**: Retrieve a specific thread
- **POST /threads/{thread_id}/messages**: Add a message to a thread
- **GET /threads/{thread_id}/messages**: Get all messages in a thread

### Runs
- **POST /threads/{thread_id}/runs**: Start a new run on a thread
- **GET /threads/{thread_id}/runs/{run_id}**: Get the status of a specific run
- **POST /threads/{thread_id}/runs/{run_id}/cancel**: Cancel an in-progress run

## Usage Examples

### Searching for Assistants
```bash
curl --request POST \
  --url http://localhost:8124/assistants/search \
  --header 'Content-Type: application/json' \
  --header 'X-Api-Key: LANGSMITH_API_KEY' \
  --data '{
  "metadata": {},
  "limit": 10,
  "offset": 0
}'
```

### Creating a Thread
```bash
curl --request POST \
  --url http://localhost:8124/threads \
  --header 'Content-Type: application/json' \
  --header 'X-Api-Key: LANGSMITH_API_KEY' \
  --data '{
  "metadata": {
    "user_id": "user_123"
  }
}'
```

### Adding a Message to a Thread
```bash
curl --request POST \
  --url http://localhost:8124/threads/thread_abc123/messages \
  --header 'Content-Type: application/json' \
  --header 'X-Api-Key: LANGSMITH_API_KEY' \
  --data '{
  "role": "user",
  "content": "Hello, can you help me with a task?"
}'
```

### Running an Assistant on a Thread
```bash
curl --request POST \
  --url http://localhost:8124/threads/thread_abc123/runs \
  --header 'Content-Type: application/json' \
  --header 'X-Api-Key: LANGSMITH_API_KEY' \
  --data '{
  "assistant_id": "assistant_xyz789"
}'
```

## Accessing the API Documentation
For complete and up-to-date API documentation, access the interactive Swagger UI available at the `/docs` endpoint of your LangGraph deployment (e.g., `http://localhost:8124/docs`).



Comments
