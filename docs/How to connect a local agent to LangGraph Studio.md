# How to connect a local agent to LangGraph Studio



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/how-tos/local-studio/
- **html**: How to connect a local agent to LangGraph Studio¶

This guide shows you how to connect your local agent to LangGraph Studio for visualization, interaction, and debugging using the development server.

Setup your application¶

First, you will need to setup your application in the proper format. This means defining a langgraph.json file which contains paths to your agent(s). See this guide for information on how to do so.

Install langgraph-cli¶

You will need to install langgraph-cli (version 0.1.55 or higher). You will need to make sure to install the inmem extras.

Minimum version

The minimum version to use the inmem extra with langgraph-cli is 0.1.55. Python 3.11 or higher is required.

pip install -U "langgraph-cli[inmem]"

Run the development server¶

Navigate to your project directory (where langgraph.json is located)

Start the server:

langgraph dev


This will look for the langgraph.json file in your current directory. In there, it will find the paths to the graph(s), and start those up. It will then automatically connect to the cloud-hosted studio.

Use the studio¶

After connecting to the studio, a browser window should automatically pop up. This will use the cloud hosted studio UI to connect to your local development server. Your graph is still running locally, the UI is connecting to visualizing the agent and threads that are defined locally.

The graph will always use the most up-to-date code, so you will be able to change the underlying code and have it automatically reflected in the studio. This is useful for debugging workflows. You can run your graph in the UI until it messes up, go in and change your code, and then rerun from the node that failed.

(Optional) Attach a debugger¶

For step-by-step debugging with breakpoints and variable inspection:

# Install debugpy package

pip install debugpy



# Start server with debugging enabled

langgraph dev --debug-port 5678


Then attach your preferred debugger:

VS Code
PyCharm

Add this configuration to launch.json:

{

  "name": "Attach to LangGraph",

  "type": "debugpy",

  "request": "attach",

  "connect": {

    "host": "0.0.0.0",

    "port": 5678

  }

}

Specify the port number you chose in the previous step.

Comments
