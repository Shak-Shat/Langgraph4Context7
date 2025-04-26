# Connecting an Authentication Provider (Part 3/3)

## Metadata
- **url**: https://langchain-ai.github.io/langgraph/tutorials/auth/add_auth_server/

## Overview
This guide explains how to upgrade a LangGraph application from using hard-coded tokens to implementing production-grade authentication with OAuth2. Building upon the previous two parts of the authentication series, this tutorial demonstrates how to integrate with an identity provider (Supabase) to validate JWT tokens, manage user sessions, and maintain the resource-level authorization implemented earlier.

## Key Concepts
- **OAuth2**: Industry-standard protocol for authorization and authentication
- **JWT (JSON Web Tokens)**: Secure method for representing claims between two parties
- **Identity Provider**: External service that manages user authentication (Supabase in this example)
- **Token Validation**: Process of verifying that a token is valid and extracting user information
- **Service Role Key**: A private key used by your application to make authenticated requests to the auth provider

## Prerequisites
- Completed setup from Parts 1 and 2 of the authentication series
- Supabase account (or another OAuth2 provider)
- Python environment with LangGraph SDK installed
- Basic understanding of authentication concepts

## Implementation Steps

### 1. Set Up Your Authentication Provider

Create a Supabase project and configure environment variables:

```bash
# Add these to your .env file
SUPABASE_URL=your-project-url
SUPABASE_SERVICE_KEY=your-service-role-key
```

Make note of your project's public "anon key" for client-side authentication.

### 2. Update the Authentication Handler

Modify your `auth.py` file to validate JWT tokens with Supabase:

```python
import os
import httpx
from langgraph_sdk import Auth

auth = Auth()

# Load from environment variables
SUPABASE_URL = os.environ["SUPABASE_URL"]
SUPABASE_SERVICE_KEY = os.environ["SUPABASE_SERVICE_KEY"]

@auth.authenticate
async def get_current_user(authorization: str | None):
    """Validate JWT tokens and extract user information."""
    assert authorization
    scheme, token = authorization.split()
    assert scheme.lower() == "bearer"
    
    try:
        # Verify token with auth provider
        async with httpx.AsyncClient() as client:
            response = await client.get(
                f"{SUPABASE_URL}/auth/v1/user",
                headers={
                    "Authorization": authorization,
                    "apiKey": SUPABASE_SERVICE_KEY,
                },
            )
            assert response.status_code == 200
            user = response.json()
            return {
                "identity": user["id"],  # Unique user identifier
                "email": user["email"],
                "is_authenticated": True,
            }
    except Exception as e:
        raise Auth.exceptions.HTTPException(status_code=401, detail=str(e))

# Keep the resource authorization from the previous tutorial
@auth.on
async def add_owner(ctx, value):
    """Make resources private to their creator using resource metadata."""
    filters = {"owner": ctx.user.identity}
    metadata = value.setdefault("metadata", {})
    metadata.update(filters)
    return filters
```

### 3. Create Test User Accounts

Set up a utility script to create and authenticate test users:

```python
import os
import httpx
from getpass import getpass
from langgraph_sdk import get_client

# Get email from command line
email = getpass("Enter your email: ")
base_email = email.split("@")
password = "secure-password"  # CHANGEME
email1 = f"{base_email[0]}+1@{base_email[1]}"
email2 = f"{base_email[0]}+2@{base_email[1]}"

SUPABASE_URL = os.environ.get("SUPABASE_URL")
if not SUPABASE_URL:
    SUPABASE_URL = getpass("Enter your Supabase project URL: ")

# This is your PUBLIC anon key (safe for client-side use)
SUPABASE_ANON_KEY = os.environ.get("SUPABASE_ANON_KEY")
if not SUPABASE_ANON_KEY:
    SUPABASE_ANON_KEY = getpass("Enter your public Supabase anon key: ")

async def sign_up(email: str, password: str):
    """Create a new user account."""
    async with httpx.AsyncClient() as client:
        response = await client.post(
            f"{SUPABASE_URL}/auth/v1/signup",
            json={"email": email, "password": password},
            headers={"apiKey": SUPABASE_ANON_KEY},
        )
        assert response.status_code == 200
        return response.json()

# Create two test users
print(f"Creating test users: {email1} and {email2}")
await sign_up(email1, password)
await sign_up(email2, password)
```

### 4. Test the Authentication System

After confirming the email addresses, test that resource authorization works with the new authentication:

```python
async def login(email: str, password: str):
    """Get an access token for an existing user."""
    async with httpx.AsyncClient() as client:
        response = await client.post(
            f"{SUPABASE_URL}/auth/v1/token?grant_type=password",
            json={
                "email": email,
                "password": password
            },
            headers={
                "apikey": SUPABASE_ANON_KEY,
                "Content-Type": "application/json"
            },
        )
        assert response.status_code == 200
        return response.json()["access_token"]

# Log in as user 1
user1_token = await login(email1, password)
user1_client = get_client(
    url="http://localhost:2024", headers={"Authorization": f"Bearer {user1_token}"}
)

# Create a thread as user 1
thread = await user1_client.threads.create()
print(f"✅ User 1 created thread: {thread['thread_id']}")

# Try to access without a token
unauthenticated_client = get_client(url="http://localhost:2024")
try:
    await unauthenticated_client.threads.create()
    print("❌ Unauthenticated access should fail!")
except Exception as e:
    print("✅ Unauthenticated access blocked:", e)

# Try to access user 1's thread as user 2
user2_token = await login(email2, password)
user2_client = get_client(
    url="http://localhost:2024", headers={"Authorization": f"Bearer {user2_token}"}
)

try:
    await user2_client.threads.get(thread["thread_id"])
    print("❌ User 2 shouldn't see User 1's thread!")
except Exception as e:
    print("✅ User 2 blocked from User 1's thread:", e)
```

## Authentication Flow
1. User authenticates with the identity provider (Supabase) and receives a JWT token
2. Client includes the token in the `Authorization` header when making requests to LangGraph
3. LangGraph validates the token by calling the identity provider's API
4. If valid, the user's information is extracted and used for authorization
5. Resource authorization then filters which resources the user can access

## Benefits
- **Production Security**: Uses industry-standard OAuth2 protocols instead of hard-coded tokens
- **User Management**: Delegates user account management to a specialized identity provider
- **Scalability**: Easily scales to handle more users and more complex authentication scenarios
- **Flexibility**: Can be adapted to work with any OAuth2-compatible provider
- **Maintainability**: Separates authentication concerns from your application logic

## Extensions and Customizations
- Add support for different authentication methods (social logins, 2FA)
- Implement role-based access control for more complex authorization requirements
- Add token refresh flows for better user experience
- Integrate with single sign-on (SSO) providers for enterprise use cases

## Next Steps
- Build a web UI to create a complete user experience
- Learn more about authentication and authorization in the conceptual guide
- Explore additional features of your chosen identity provider
