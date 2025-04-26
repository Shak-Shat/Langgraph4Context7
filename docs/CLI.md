# CLI



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/cloud/reference/cli/
- **html**: API reference
LangGraph Platform
LangGraph CLI¶

The LangGraph command line interface includes commands to build and run a LangGraph Cloud API server locally in Docker. For development and testing, you can use the CLI to deploy a local API server.

Installation¶
Ensure that Docker is installed (e.g. docker --version).

Install the CLI package:

Python
JS
pip install langgraph-cli



# Install via Homebrew

brew install langgraph-cli


Run the command langgraph --help or npx @langchain/langgraph-cli --help to confirm that the CLI is working correctly.

Configuration File¶

The LangGraph CLI requires a JSON configuration file with the following keys:

Note

The LangGraph CLI defaults to using the configuration file langgraph.json in the current directory.

Python
JS
Key	Description
dependencies	Required. Array of dependencies for LangGraph Cloud API server. Dependencies can be one of the following: (1) ".", which will look for local Python packages, (2) pyproject.toml, setup.py or requirements.txt in the app directory "./local_package", or (3) a package name.
graphs	Required. Mapping from graph ID to path where the compiled graph or a function that makes a graph is defined. Example:
./your_package/your_file.py:variable, where variable is an instance of langgraph.graph.state.CompiledStateGraph
./your_package/your_file.py:make_graph, where make_graph is a function that takes a config dictionary (langchain_core.runnables.RunnableConfig) and creates an instance of langgraph.graph.state.StateGraph / langgraph.graph.state.CompiledStateGraph.

auth	(Added in v0.0.11) Auth configuration containing the path to your authentication handler. Example: ./your_package/auth.py:auth, where auth is an instance of langgraph_sdk.Auth. See authentication guide for details.
env	Path to .env file or a mapping from environment variable to its value.
store	Configuration for adding semantic search to the BaseStore. Contains the following fields:
index: Configuration for semantic search indexing with fields:
embed: Embedding provider (e.g., "openai:text-embedding-3-small") or path to custom embedding function
dims: Dimension size of the embedding model. Used to initialize the vector table.
fields (optional): List of fields to index. Defaults to ["$"], which means to index entire documents. Can be specific fields like ["text", "summary", "some.value"]

python_version	3.11 or 3.12. Defaults to 3.11.
node_version	Specify node_version: 20 to use LangGraph.js.
pip_config_file	Path to pip config file.
dockerfile_lines	Array of additional lines to add to Dockerfile following the import from parent image.
http	HTTP server configuration with the following fields:
app: Path to custom Starlette/FastAPI app (e.g., "./src/agent/webapp.py:app"). See custom routes guide.
disable_assistants: Disable /assistants routes
disable_threads: Disable /threads routes
disable_runs: Disable /runs routes
disable_store: Disable /store routes
disable_meta: Disable /ok, /info, /metrics, and /docs routes
cors: CORS configuration with fields for allow_origins, allow_methods, allow_headers, etc.
Examples¶
Python
JS
Basic Configuration¶
{

  "dependencies": ["."],

  "graphs": {

    "chat": "./chat/graph.py:graph"

  }

}

Adding semantic search to the store¶

All deployments come with a DB-backed BaseStore. Adding an "index" configuration to your langgraph.json will enable semantic search within the BaseStore of your deployment.

The fields configuration determines which parts of your documents to embed:

If omitted or set to ["$"], the entire document will be embedded
To embed specific fields, use JSON path notation: ["metadata.title", "content.text"]
Documents missing specified fields will still be stored but won't have embeddings for those fields
You can still override which fields to embed on a specific item at put time using the index parameter
{

  "dependencies": ["."],

  "graphs": {

    "memory_agent": "./agent/graph.py:graph"

  },

  "store": {

    "index": {

      "embed": "openai:text-embedding-3-small",

      "dims": 1536,

      "fields": ["$"]

    }

  }

}


Common model dimensions

openai:text-embedding-3-large: 3072
openai:text-embedding-3-small: 1536
openai:text-embedding-ada-002: 1536
cohere:embed-english-v3.0: 1024
cohere:embed-english-light-v3.0: 384
cohere:embed-multilingual-v3.0: 1024
cohere:embed-multilingual-light-v3.0: 384
Semantic search with a custom embedding function¶

If you want to use semantic search with a custom embedding function, you can pass a path to a custom embedding function:

{

  "dependencies": ["."],

  "graphs": {

    "memory_agent": "./agent/graph.py:graph"

  },

  "store": {

    "index": {

      "embed": "./embeddings.py:embed_texts",

      "dims": 768,

      "fields": ["text", "summary"]

    }

  }

}


The embed field in store configuration can reference a custom function that takes a list of strings and returns a list of embeddings. Example implementation:

# embeddings.py

def embed_texts(texts: list[str]) -> list[list[float]]:

    """Custom embedding function for semantic search."""

    # Implementation using your preferred embedding model

    return [[0.1, 0.2, ...] for _ in texts]  # dims-dimensional vectors

Adding custom authentication¶
{

  "dependencies": ["."],

  "graphs": {

    "chat": "./chat/graph.py:graph"

  },

  "auth": {

    "path": "./auth.py:auth",

    "openapi": {

      "securitySchemes": {

        "apiKeyAuth": {

          "type": "apiKey",

          "in": "header",

          "name": "X-API-Key"

        }

      },

      "security": [{ "apiKeyAuth": [] }]

    },

    "disable_studio_auth": false

  }

}


See the authentication conceptual guide for details, and the setting up custom authentication guide for a practical walk through of the process.

Commands¶

Usage

Python
JS

The base command for the LangGraph CLI is langgraph.

langgraph [OPTIONS] COMMAND [ARGS]

dev¶
Python
JS

Run LangGraph API server in development mode with hot reloading and debugging capabilities. This lightweight server requires no Docker installation and is suitable for development and testing. State is persisted to a local directory.

Note

Currently, the CLI only supports Python >= 3.11.

Installation

This command requires the "inmem" extra to be installed:

pip install -U "langgraph-cli[inmem]"


Usage

langgraph dev [OPTIONS]


Options

Option	Default	Description
-c, --config FILE	langgraph.json	Path to configuration file declaring dependencies, graphs and environment variables
--host TEXT	127.0.0.1	Host to bind the server to
--port INTEGER	2024	Port to bind the server to
--no-reload		Disable auto-reload
--n-jobs-per-worker INTEGER		Number of jobs per worker. Default is 10
--debug-port INTEGER		Port for debugger to listen on
--help		Display command documentation
build¶
Python
JS

Build LangGraph Cloud API server Docker image.

Usage

langgraph build [OPTIONS]


Options

Option	Default	Description
--platform TEXT		Target platform(s) to build the Docker image for. Example: langgraph build --platform linux/amd64,linux/arm64
-t, --tag TEXT		Required. Tag for the Docker image. Example: langgraph build -t my-image
--pull / --no-pull	--pull	Build with latest remote Docker image. Use --no-pull for running the LangGraph Cloud API server with locally built images.
-c, --config FILE	langgraph.json	Path to configuration file declaring dependencies, graphs and environment variables.
--help		Display command documentation.
up¶
Python
JS

Start LangGraph API server. For local testing, requires a LangSmith API key with access to LangGraph Cloud closed beta. Requires a license key for production use.

Usage

langgraph up [OPTIONS]


Options

Option	Default	Description
--wait		Wait for services to start before returning. Implies --detach
--postgres-uri TEXT	Local database	Postgres URI to use for the database.
--watch		Restart on file changes
--debugger-base-url TEXT	http://127.0.0.1:[PORT]	URL used by the debugger to access LangGraph API.
--debugger-port INTEGER		Pull the debugger image locally and serve the UI on specified port
--verbose		Show more output from the server logs.
-c, --config FILE	langgraph.json	Path to configuration file declaring dependencies, graphs and environment variables.
-d, --docker-compose FILE		Path to docker-compose.yml file with additional services to launch.
-p, --port INTEGER	8123	Port to expose. Example: langgraph up --port 8000
--pull / --no-pull	pull	Pull latest images. Use --no-pull for running the server with locally-built images. Example: langgraph up --no-pull
--recreate / --no-recreate	no-recreate	Recreate containers even if their configuration and image haven't changed
--help		Display command documentation.
dockerfile¶
Python
JS

Generate a Dockerfile for building a LangGraph Cloud API server Docker image.

Usage

langgraph dockerfile [OPTIONS] SAVE_PATH


Options

Option	Default	Description
-c, --config FILE	langgraph.json	Path to the configuration file declaring dependencies, graphs and environment variables.
--help		Show this message and exit.

Example:

langgraph dockerfile -c langgraph.json Dockerfile


This generates a Dockerfile that looks similar to:

FROM langchain/langgraph-api:3.11



ADD ./pipconf.txt /pipconfig.txt



RUN PIP_CONFIG_FILE=/pipconfig.txt PYTHONDONTWRITEBYTECODE=1 pip install --no-cache-dir -c /api/constraints.txt langchain_community langchain_anthropic langchain_openai wikipedia scikit-learn



ADD ./graphs /deps/__outer_graphs/src

RUN set -ex && \

    for line in '[project]' \

                'name = "graphs"' \

                'version = "0.1"' \

                '[tool.setuptools.package-data]' \

                '"*" = ["**/*"]'; do \

        echo "$line" >> /deps/__outer_graphs/pyproject.toml; \

    done



RUN PIP_CONFIG_FILE=/pipconfig.txt PYTHONDONTWRITEBYTECODE=1 pip install --no-cache-dir -c /api/constraints.txt -e /deps/*



ENV LANGSERVE_GRAPHS='{"agent": "/deps/__outer_graphs/src/agent.py:graph", "storm": "/deps/__outer_graphs/src/storm.py:graph"}'

Updating your langgraph.json file

The langgraph dockerfile command translates all the configuration in your langgraph.json file into Dockerfile commands. When using this command, you will have to re-run it whenever you update your langgraph.json file. Otherwise, your changes will not be reflected when you build or run the dockerfile.

Comments
