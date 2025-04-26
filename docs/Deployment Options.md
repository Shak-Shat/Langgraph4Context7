# Deployment Options



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/concepts/deployment_options/
- **html**: Home
Guides
Concepts
LangGraph Platform
High Level
Deployment Options¶

Prerequisites

LangGraph Platform
LangGraph Server
LangGraph Platform Plans
Overview¶

There are 4 main options for deploying with the LangGraph Platform:

Self-Hosted Lite: Available for all plans.

Self-Hosted Enterprise: Available for the Enterprise plan.

Cloud SaaS: Available for Plus and Enterprise plans.

Bring Your Own Cloud: Available only for Enterprise plans and only on AWS.

Please see the LangGraph Platform Plans for more information on the different plans.

The guide below will explain the differences between the deployment options.

Self-Hosted Enterprise¶

Important

The Self-Hosted Enterprise version is only available for the Enterprise plan.

Note

The LangGraph Platform Deployments view is optionally available for Self-Hosted Enterprise LangGraph deployments. With one click, self-hosted LangGraph deployments can be deployed in the same Kubernetes cluster where a self-hosted LangSmith instance is deployed.

With a Self-Hosted Enterprise deployment, you are responsible for managing the infrastructure, including setting up and maintaining required databases and Redis instances.

You’ll build a Docker image using the LangGraph CLI, which can then be deployed on your own infrastructure.

For more information, please see:

Self-Hosted conceptual guide
Self-Hosted Deployment how-to guide
Self-Hosted Lite¶

Important

The Self-Hosted Lite version is available for all plans.

Note

The LangGraph Platform Deployments view is optionally available for Self-Hosted Lite LangGraph deployments. With one click, self-hosted LangGraph deployments can be deployed in the same Kubernetes cluster where a self-hosted LangSmith instance is deployed.

The Self-Hosted Lite deployment option is a free (up to 1 million nodes executed per year), limited version of LangGraph Platform that you can run locally or in a self-hosted manner.

With a Self-Hosted Lite deployment, you are responsible for managing the infrastructure, including setting up and maintaining required databases and Redis instances.

You’ll build a Docker image using the LangGraph CLI, which can then be deployed on your own infrastructure.

Cron jobs are not available for Self-Hosted Lite deployments.

For more information, please see:

Self-Hosted conceptual guide
Self-Hosted deployment how-to guide
Cloud SaaS¶

Important

The Cloud SaaS version of LangGraph Platform is only available for Plus and Enterprise plans.

The Cloud SaaS version of LangGraph Platform is hosted as part of LangSmith.

The Cloud SaaS version of LangGraph Platform provides a simple way to deploy and manage your LangGraph applications.

This deployment option provides access to the LangGraph Platform UI (within LangSmith) and an integration with GitHub, allowing you to deploy code from any of your repositories on GitHub.

For more information, please see:

Cloud SaaS Conceptual Guide
How to deploy to Cloud SaaS
Bring Your Own Cloud¶

Important

The Bring Your Own Cloud version of LangGraph Platform is only available for Enterprise plans.

This combines the best of both worlds for Cloud and Self-Hosted. Create your deployments through the LangGraph Platform UI (within LangSmith) and we manage the infrastructure so you don't have to. The infrastructure all runs within your cloud. This is currently only available on AWS.

For more information please see:

Bring Your Own Cloud Conceptual Guide
Related¶

For more information, please see:

LangGraph Platform plans
LangGraph Platform pricing
Deployment how-to guides
Comments
