# How to add custom authentication



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/how-tos/auth/custom_auth/
- **html**: How to add custom authentication¶

Prerequisites

This guide assumes familiarity with the following concepts:

Authentication & Access Control
LangGraph Platform

For a more guided walkthrough, see setting up custom authentication tutorial.

Python only

We currently only support custom authentication and authorization in Python deployments with langgraph-api>=0.0.11. Support for LangGraph.JS will be added soon.

Support by deployment type

Custom auth is supported for all deployments in the managed LangGraph Cloud, as well as Enterprise self-hosted plans. It is not supported for Lite self-hosted plans.

This guide shows how to add custom authentication to your LangGraph Platform application. This guide applies to both LangGraph Cloud, BYOC, and self-hosted deployments. It does not apply to isolated usage of the LangGraph open source library in your own custom server.

1. Implement authentication¶
from langgraph_sdk import Auth



my_auth = Auth()



@my_auth.authenticate

async def authenticate(authorization: str) -> str:

    token = authorization.split(" ", 1)[-1] # "Bearer <token>"

    try:

        # Verify token with your auth provider

        user_id = await verify_token(token)

        return user_id

    except Exception:

        raise Auth.exceptions.HTTPException(

            status_code=401,

            detail="Invalid token"

        )



# Add authorization rules to actually control access to resources

@my_auth.on

async def add_owner(

    ctx: Auth.types.AuthContext,

    value: dict,

):

    """Add owner to resource metadata and filter by owner."""

    filters = {"owner": ctx.user.identity}

    metadata = value.setdefault("metadata", {})

    metadata.update(filters)

    return filters



# Assumes you organize information in store like (user_id, resource_type, resource_id)

@my_auth.on.store()

async def authorize_store(ctx: Auth.types.AuthContext, value: dict):

    namespace: tuple = value["namespace"]

    assert namespace[0] == ctx.user.identity, "Not authorized"

2. Update configuration¶

In your langgraph.json, add the path to your auth file:

{

  "dependencies": ["."],

  "graphs": {

    "agent": "./agent.py:graph"

  },

  "env": ".env",

  "auth": {

    "path": "./auth.py:my_auth"

  }

}

3. Connect from the client¶

Once you've set up authentication in your server, requests must include the the required authorization information based on your chosen scheme. Assuming you are using JWT token authentication, you could access your deployments using any of the following methods:

Python Client
Python RemoteGraph
JavaScript Client
JavaScript RemoteGraph
CURL
from langgraph_sdk import get_client



my_token = "your-token" # In practice, you would generate a signed token with your auth provider

client = get_client(

    url="http://localhost:2024",

    headers={"Authorization": f"Bearer {my_token}"}

)

threads = await client.threads.search()

Comments
