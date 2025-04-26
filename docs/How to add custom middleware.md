# How to add custom middleware



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/how-tos/http/custom_middleware/
- **html**: How to add custom middleware¶

When deploying agents on the LangGraph platform, you can add custom middleware to your server to handle cross-cutting concerns like logging request metrics, injecting or checking headers, and enforcing security policies without modifying core server logic. This works the same way as adding custom routes - you just need to provide your own Starlette app (including FastAPI, FastHTML and other compatible apps).

Adding middleware lets you intercept and modify requests and responses globally across your deployment, whether they're hitting your custom endpoints or the built-in LangGraph Platform APIs.

Below is an example using FastAPI.

Python only

We currently only support custom middleware in Python deployments with langgraph-api>=0.0.26.

Create app¶

Starting from an existing LangGraph Platform application, add the following middleware code to your webapp.py file. If you are starting from scratch, you can create a new app from a template using the CLI.

langgraph new --template=new-langgraph-project-python my_new_project


Once you have a LangGraph project, add the following app code:

# ./src/agent/webapp.py

from fastapi import FastAPI, Request

from starlette.middleware.base import BaseHTTPMiddleware



app = FastAPI()



class CustomHeaderMiddleware(BaseHTTPMiddleware):

    async def dispatch(self, request: Request, call_next):

        response = await call_next(request)

        response.headers['X-Custom-Header'] = 'Hello from middleware!'

        return response



# Add the middleware to the app

app.add_middleware(CustomHeaderMiddleware)

Configure langgraph.json¶

Add the following to your langgraph.json file. Make sure the path points to the webapp.py file you created above.

{

  "dependencies": ["."],

  "graphs": {

    "agent": "./src/agent/graph.py:graph"

  },

  "env": ".env",

  "http": {

    "app": "./src/agent/webapp.py:app"

  }

  // Other configuration options like auth, store, etc.

}

Start server¶

Test the server out locally:

langgraph dev --no-browser


Now any request to your server will include the custom header X-Custom-Header in its response.

Deploying¶

You can deploy this app as-is to the managed langgraph cloud or to your self-hosted platform.

Next steps¶

Now that you've added custom middleware to your deployment, you can use similar techniques to add custom routes or define custom lifespan events to further customize your server's behavior.

Comments
