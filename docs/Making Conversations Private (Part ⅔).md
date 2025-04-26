# Making Conversations Private (Part ⅔)

## Metadata
- **url**: https://langchain-ai.github.io/langgraph/tutorials/auth/resource_auth/

## Overview
This guide explains how to implement resource-level authorization in LangGraph applications, allowing users to have private conversations that only they can access. Building on the authentication system from Part 1, this tutorial focuses on restricting access to resources based on user identity, ensuring that each user can only interact with their own threads and messages.

## Key Concepts
- **Resource Authorization**: Controlling access to specific resources (threads, assistants, etc.) based on user identity
- **Ownership Metadata**: Tagging resources with metadata to track who created them
- **Access Filters**: Rules that determine which resources a user can view or modify
- **Authorization Context**: Information about the current user and the action they're trying to perform
- **Scoped Authorization Handlers**: Specialized handlers that target specific resources and actions

## Prerequisites
- Completed setup from "Setting up Custom Authentication (Part ⅓)"
- Running LangGraph application with basic authentication
- Understanding of LangGraph resources (threads, assistants, etc.)
- Python environment with LangGraph SDK installed

## Implementation Steps

### 1. Create a Global Authorization Handler
Add an authorization handler to your `auth.py` file that will run on all resource access:

```python
from langgraph_sdk import Auth

# Keep our test users from the previous tutorial
VALID_TOKENS = {
    "user1-token": {"id": "user1", "name": "Alice"},
    "user2-token": {"id": "user2", "name": "Bob"},
}

auth = Auth()

@auth.authenticate
async def get_current_user(authorization: str | None) -> Auth.types.MinimalUserDict:
    """Our authentication handler from the previous tutorial."""
    assert authorization
    scheme, token = authorization.split()
    assert scheme.lower() == "bearer"
    
    if token not in VALID_TOKENS:
        raise Auth.exceptions.HTTPException(status_code=401, detail="Invalid token")
    
    user_data = VALID_TOKENS[token]
    return {
        "identity": user_data["id"],
    }

@auth.on
async def add_owner(
    ctx: Auth.types.AuthContext,  # Contains info about the current user
    value: dict,  # The resource being created/accessed
):
    """Make resources private to their creator."""
    # Add the user's ID to the resource's metadata
    filters = {"owner": ctx.user.identity}
    metadata = value.setdefault("metadata", {})
    metadata.update(filters)
    
    # Only let users see their own resources
    return filters
```

### 2. Test the Authorization System
Create a test script to verify that the resource authorization works:

```python
from langgraph_sdk import get_client

# Create clients for both users
alice = get_client(
    url="http://localhost:2024",
    headers={"Authorization": "Bearer user1-token"}
)

bob = get_client(
    url="http://localhost:2024",
    headers={"Authorization": "Bearer user2-token"}
)

# Alice creates a thread and chats
alice_thread = await alice.threads.create()
print(f"✅ Alice created thread: {alice_thread['thread_id']}")

await alice.runs.create(
    thread_id=alice_thread["thread_id"],
    assistant_id="agent",
    input={"messages": [{"role": "user", "content": "Hi, this is Alice's private chat"}]}
)

# Bob tries to access Alice's thread (should fail)
try:
    await bob.threads.get(alice_thread["thread_id"])
    print("❌ Bob shouldn't see Alice's thread!")
except Exception as e:
    print("✅ Bob correctly denied access:", e)

# Bob creates his own thread
bob_thread = await bob.threads.create()
await bob.runs.create(
    thread_id=bob_thread["thread_id"],
    assistant_id="agent",
    input={"messages": [{"role": "user", "content": "Hi, this is Bob's private chat"}]}
)
print(f"✅ Bob created his own thread: {bob_thread['thread_id']}")

# List threads - each user only sees their own
alice_threads = await alice.threads.search()
bob_threads = await bob.threads.search()
print(f"✅ Alice sees {len(alice_threads)} thread")
print(f"✅ Bob sees {len(bob_threads)} thread")
```

### 3. Implementing Fine-Grained Authorization with Scoped Handlers
For more precise control, replace the global handler with resource-specific handlers:

```python
from langgraph_sdk import Auth

# Keep our previous authentication handler...

@auth.on.threads.create
async def on_thread_create(
    ctx: Auth.types.AuthContext,
    value: Auth.types.on.threads.create.value,
):
    """Add owner when creating threads."""
    # Add owner metadata to the thread being created
    metadata = value.setdefault("metadata", {})
    metadata["owner"] = ctx.user.identity
    
    # Return filter to restrict access to just the creator
    return {"owner": ctx.user.identity}

@auth.on.threads.read
async def on_thread_read(
    ctx: Auth.types.AuthContext,
    value: Auth.types.on.threads.read.value,
):
    """Only let users read their own threads."""
    return {"owner": ctx.user.identity}

@auth.on.assistants
async def on_assistants(
    ctx: Auth.types.AuthContext,
    value: Auth.types.on.assistants.value,
):
    # For illustration purposes, we will deny all requests
    # that touch the assistants resource
    raise Auth.exceptions.HTTPException(
        status_code=403,
        detail="User lacks the required permissions.",
    )
```

### 4. Testing Fine-Grained Authorization
Add the following test code to verify fine-grained access control:

```python
# Try creating an assistant (should fail with our restrictions)
try:
    await alice.assistants.create("agent")
    print("❌ Alice shouldn't be able to create assistants!")
except Exception as e:
    print("✅ Alice correctly denied access:", e)

# Try searching for assistants (should also fail)
try:
    await alice.assistants.search()
    print("❌ Alice shouldn't be able to search assistants!")
except Exception as e:
    print("✅ Alice correctly denied access to searching assistants:", e)

# Alice can still create threads
alice_thread = await alice.threads.create()
print(f"✅ Alice created thread: {alice_thread['thread_id']}")
```

## Authorization Flow
1. User makes a request to access a resource
2. Authentication handler verifies the user's identity
3. LangGraph selects the most specific authorization handler for the resource and action
4. Authorization handler evaluates whether the user should have access
5. If allowed, the handler adds ownership metadata and returns access filters
6. The operation proceeds with the applied filters, showing only authorized resources

## Benefits
- **Data Privacy**: Each user can only see and interact with their own resources
- **Multi-tenant Security**: Prevents information leakage between users of the same application
- **Flexible Access Control**: Can be customized per resource type and action
- **Structured Data Organization**: Automatically tags resources with ownership metadata
- **Scalable Architecture**: Authorization logic is centralized and consistently applied

## Next Steps
- Proceed to Part 3 to implement production-grade authentication using OAuth2
- Explore more complex authorization patterns based on roles and permissions
- Implement custom logic for sharing resources between specific users
- Learn about integration with external identity providers like Auth0 or Cognito
