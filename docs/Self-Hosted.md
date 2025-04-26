# Self-Hosted



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/concepts/self_hosted/
- **html**: Home
Guides
Concepts
LangGraph Platform
Deployment Options
Self-Hosted¶

Note

LangGraph Platform
Deployment Options
Versions¶

There are two versions of the self-hosted deployment: Self-Hosted Enterprise and Self-Hosted Lite.

Self-Hosted Lite¶

The Self-Hosted Lite version is a limited version of LangGraph Platform that you can run locally or in a self-hosted manner (up to 1 million nodes executed per year).

When using the Self-Hosted Lite version, you authenticate with a LangSmith API key.

Self-Hosted Enterprise¶

The Self-Hosted Enterprise version is the full version of LangGraph Platform.

To use the Self-Hosted Enterprise version, you must acquire a license key that you will need to pass in when running the Docker image. To acquire a license key, please email sales@langchain.dev.

Requirements¶
You use langgraph-cli and/or LangGraph Studio app to test graph locally.
You use langgraph build command to build image.
How it works¶
Deploy Redis and Postgres instances on your own infrastructure.
Build the docker image for LangGraph Server using the LangGraph CLI.
Deploy a web server that will run the docker image and pass in the necessary environment variables.

Note

The LangGraph Platform Deployments view is optionally available for Self-Hosted LangGraph deployments. With one click, self-hosted LangGraph deployments can be deployed in the same Kubernetes cluster where a self-hosted LangSmith instance is deployed.

For step-by-step instructions, see How to set up a self-hosted deployment of LangGraph.

Helm Chart¶

If you would like to deploy LangGraph Cloud on Kubernetes, you can use this Helm chart.

Related¶
How to set up a self-hosted deployment of LangGraph.
Comments
