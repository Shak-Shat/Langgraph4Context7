# How to add custom lifespan events



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/how-tos/http/custom_lifespan/
- **html**: How to add custom lifespan events¶

When deploying agents on the LangGraph platform, you often need to initialize resources like database connections when your server starts up, and ensure they're properly closed when it shuts down. Lifespan events let you hook into your server's startup and shutdown sequence to handle these critical setup and teardown tasks.

This works the same way as adding custom routes - you just need to provide your own Starlette app (including FastAPI, FastHTML and other compatible apps).

Below is an example using FastAPI.

Python only

We currently only support custom lifespan events in Python deployments with langgraph-api>=0.0.26.

Create app¶

Starting from an existing LangGraph Platform application, add the following lifespan code to your webapp.py file. If you are starting from scratch, you can create a new app from a template using the CLI.

langgraph new --template=new-langgraph-project-python my_new_project


Once you have a LangGraph project, add the following app code:

# ./src/agent/webapp.py

from contextlib import asynccontextmanager

from fastapi import FastAPI

from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

from sqlalchemy.orm import sessionmaker



@asynccontextmanager

async def lifespan(app: FastAPI):

    # for example...

    engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/db")

    # Create reusable session factory

    async_session = sessionmaker(engine, class_=AsyncSession)

    # Store in app state

    app.state.db_session = async_session

    yield

    # Clean up connections

    await engine.dispose()



app = FastAPI(lifespan=lifespan)



# ... can add custom routes if needed.

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


You should see your startup message printed when the server starts, and your cleanup message when you stop it with Ctrl+C.

Deploying¶

You can deploy your app as-is to the managed langgraph cloud or to your self-hosted platform.

Next steps¶

Now that you've added lifespan events to your deployment, you can use similar techniques to add custom routes or custom middleware to further customize your server's behavior.

Comments
