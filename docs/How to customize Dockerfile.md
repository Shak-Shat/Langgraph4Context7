# How to customize Dockerfile



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/cloud/deployment/custom_docker/
- **html**: Home
Guides
How-to Guides
LangGraph Platform
Application Structure
How to customize Dockerfile¶

Users can add an array of additional lines to add to the Dockerfile following the import from the parent LangGraph image. In order to do this, you simply need to modify your langgraph.json file by passing in the commands you want run to the dockerfile_lines key. For example, if we wanted to use Pillow in our graph you would need to add the following dependencies:

{

    "dependencies": ["."],

    "graphs": {

        "openai_agent": "./openai_agent.py:agent",

    },

    "env": "./.env",

    "dockerfile_lines": [

        "RUN apt-get update && apt-get install -y libjpeg-dev zlib1g-dev libpng-dev",

        "RUN pip install Pillow"

    ]

}


This would install the system packages required to use Pillow if we were working with jpeq or png image formats.

Comments
