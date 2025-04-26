# Setting up Custom Authentication (Part ⅓)

## Metadata
- **url**: https://langchain-ai.github.io/langgraph/tutorials/auth/getting_started/

## Overview
This guide explains how to implement custom authentication for LangGraph Platform applications. It focuses on controlling access to your application by verifying user identities through token-based authentication. This is the first part of a three-part series on authentication and authorization in LangGraph.

## Key Concepts
- **Authentication**: Verifying the identity of users before allowing access to your application
- **Token-based Auth**: Using tokens in request headers to authenticate users
- **Authorization Handler**: A function that validates credentials and returns user identity information
- **LangGraph Auth Object**: Container that registers authentication middleware for the platform

## Prerequisites
- Basic understanding of authentication and access control concepts
- Familiarity with LangGraph Platform
- Python environment (custom auth currently only supports Python deployments)
- LangGraph CLI installed (`pip install -U "langgraph-cli[inmem]"`)
- LangGraph API version 0.0.11 or higher

## Implementation Steps

### 1. Create a Basic LangGraph Project
```bash
# Create a new project from template
pip install -U "langgraph-cli[inmem]"
langgraph new --template=new-langgraph-project-python custom-auth
cd custom-auth

# Install local dependencies
pip install -e .
```

### 2. Create an Authentication Handler
Create a file `src/security/auth.py` with the following code:

```python
from langgraph_sdk import Auth

# Demo user database (for illustration only)
VALID_TOKENS = {
    "user1-token": {"id": "user1", "name": "Alice"},
    "user2-token": {"id": "user2", "name": "Bob"},
}

# Create Auth container
auth = Auth()

# Define authentication handler
@auth.authenticate
async def get_current_user(authorization: str | None) -> Auth.types.MinimalUserDict:
    """Check if the user's token is valid."""
    assert authorization
    scheme, token = authorization.split()
    assert scheme.lower() == "bearer"
    
    # Check if token is valid
    if token not in VALID_TOKENS:
        raise Auth.exceptions.HTTPException(status_code=401, detail="Invalid token")

    # Return user info if valid
    user_data = VALID_TOKENS[token]
    return {
        "identity": user_data["id"],
    }
```

### 3. Configure the Project to Use Authentication
Update the `langgraph.json` configuration file:

```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./src/agent/graph.py:graph"
  },
  "env": ".env",
  "auth": {
    "path": "src/security/auth.py:auth"
  }
}
```

### 4. Start the Development Server
```bash
langgraph dev --no-browser
```

### 5. Test the Authentication System
Create a test script to verify the authentication works:

```python
from langgraph_sdk import get_client

# Try without a token (should fail)
client = get_client(url="http://localhost:2024")
try:
    thread = await client.threads.create()
    print("❌ Should have failed without token!")
except Exception as e:
    print("✅ Correctly blocked access:", e)

# Try with a valid token
client = get_client(
    url="http://localhost:2024", headers={"Authorization": "Bearer user1-token"}
)

# Create a thread and chat
thread = await client.threads.create()
print(f"✅ Created thread as Alice: {thread['thread_id']}")

response = await client.runs.create(
    thread_id=thread["thread_id"],
    assistant_id="agent",
    input={"messages": [{"role": "user", "content": "Hello!"}]},
)
print("✅ Bot responded:")
print(response)
```

## Authentication Process Flow
1. Client sends a request with an `Authorization` header containing a bearer token
2. LangGraph platform invokes the registered authentication handler
3. Handler validates the token and extracts user identity
4. If token is valid, the request proceeds with the authenticated user context
5. If token is invalid, a 401 Unauthorized response is returned

## Deployment Support
- **Supported**: All deployments in managed LangGraph Cloud and Enterprise self-hosted plans
- **Not Supported**: Lite self-hosted plans and LangGraph.JS (coming soon)

## Benefits
- **Access Control**: Restrict access to your LangGraph applications to authorized users only
- **User Identity**: Associate actions and resources with specific user identities
- **Security Foundation**: Provides the basis for more advanced security features like resource authorization
- **API Integration**: Simplifies integration with other services and frontends that need secure access

## Next Steps
- Proceed to Part 2 to learn about resource authorization and private conversations
- Explore more advanced authentication schemes using OAuth2 in Part 3
- Review the LangGraph SDK documentation for additional authentication capabilities
