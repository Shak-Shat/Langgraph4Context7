# How to document API authentication in OpenAPI



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/how-tos/auth/openapi_security/
- **html**: How to document API authentication in OpenAPI¶

This guide shows how to customize the OpenAPI security schema for your LangGraph Platform API documentation. A well-documented security schema helps API consumers understand how to authenticate with your API and even enables automatic client generation. See the Authentication & Access Control conceptual guide for more details about LangGraph's authentication system.

Implementation vs Documentation

This guide only covers how to document your security requirements in OpenAPI. To implement the actual authentication logic, see How to add custom authentication.

This guide applies to all LangGraph Platform deployments (Cloud, BYOC, and self-hosted). It does not apply to usage of the LangGraph open source library if you are not using LangGraph Platform.

Default Schema¶

The default security scheme varies by deployment type:

LangGraph Cloud

By default, LangGraph Cloud requires a LangSmith API key in the x-api-key header:

components:

  securitySchemes:

    apiKeyAuth:

      type: apiKey

      in: header

      name: x-api-key

security:

  - apiKeyAuth: []


When using one of the LangGraph SDK's, this can be inferred from environment variables.

Self-hosted

By default, self-hosted deployments have no security scheme. This means they are to be deployed only on a secured network or with authentication. To add custom authentication, see How to add custom authentication.

Custom Security Schema¶

To customize the security schema in your OpenAPI documentation, add an openapi field to your auth configuration in langgraph.json. Remember that this only updates the API documentation - you must also implement the corresponding authentication logic as shown in How to add custom authentication.

Note that LangGraph Platform does not provide authentication endpoints - you'll need to handle user authentication in your client application and pass the resulting credentials to the LangGraph API.

OAuth2 with Bearer Token
API Key
{

  "auth": {

    "path": "./auth.py:my_auth",  // Implement auth logic here

    "openapi": {

      "securitySchemes": {

        "OAuth2": {

          "type": "oauth2",

          "flows": {

            "implicit": {

              "authorizationUrl": "https://your-auth-server.com/oauth/authorize",

              "scopes": {

                "me": "Read information about the current user",

                "threads": "Access to create and manage threads"

              }

            }

          }

        }

      },

      "security": [

        {"OAuth2": ["me", "threads"]}

      ]

    }

  }

}

Testing¶

After updating your configuration:

Deploy your application
Visit /docs to see the updated OpenAPI documentation
Try out the endpoints using credentials from your authentication server (make sure you've implemented the authentication logic first)
Comments
