# Tutorials

## Metadata
- **url**: https://langchain-ai.github.io/langgraph/tutorials/?q=

## Overview
LangGraph offers a variety of tutorials for users new to the framework or LLM application development. These tutorials cover everything from basic concepts to advanced use cases, helping users get started with building their own applications and understanding common patterns.

## Key Concepts
- **Getting Started**: Quick introductions to LangGraph's core concepts and capabilities
- **Use Cases**: Practical implementations for specific scenarios and problems
- **Agent Architectures**: Different approaches to building LLM-powered agents
- **LangGraph Platform**: Resources for deploying and managing LangGraph applications

## Getting Started
- **LangGraph Quickstart**: Build a chatbot with tool use and conversation memory, including human-in-the-loop capabilities and time-travel functionality
- **Common Workflows**: Overview of standard LLM-based workflows implemented with LangGraph
- **LangGraph Server**: Launch a local server and interact via REST API and Studio Web UI
- **Template Applications**: Start building with pre-configured templates
- **Cloud Deployment**: Deploy applications using LangGraph Cloud

## Use Cases

### Chatbots
- **Customer Support**: Multi-functional support bot for travel services (flights, hotels, car rentals)
- **Prompt Generation**: Information gathering chatbot based on user requirements
- **Code Assistant**: Code analysis and generation assistant

### Retrieval-Augmented Generation (RAG)
- **Agentic RAG**: Agent-driven retrieval to find the most relevant information
- **Adaptive RAG**: Strategy that combines query analysis with self-corrective capabilities
- **Corrective RAG**: LLM-based grading of retrieved information quality with alternative source retrieval
- **Self-RAG**: Incorporates self-reflection and self-grading on retrieved documents
- **SQL Agent**: Question-answering over SQL databases

## Agent Architectures

### Multi-Agent Systems
- **Network**: Collaboration between two or more agents
- **Supervisor**: LLM orchestration and delegation to individual agents
- **Hierarchical Teams**: Nested teams of agents for complex problem-solving

### Planning Agents
- **Plan-and-Execute**: Basic planning and execution agent
- **Reasoning without Observation**: Reduced re-planning through observation variable storage
- **LLMCompiler**: Streaming and eager execution of planner-created task DAGs

### Reflection & Critique
- **Basic Reflection**: Output reflection and revision
- **Reflexion**: Detail critique to guide subsequent steps
- **Tree of Thoughts**: Scored tree search over candidate solutions
- **Language Agent Tree Search**: Monte-carlo tree search driven by reflection and rewards
- **Self-Discover Agent**: Self-learning agent capability analysis

## Evaluation
- **Agent-based**: Chatbot evaluation via simulated user interactions
- **LangSmith Integration**: Dialog dataset evaluation in LangSmith

## Experimental Projects
- **Web Research (STORM)**: Wikipedia-like article generation through research and multi-perspective QA
- **TNT-LLM**: Rich taxonomies of user intent using Microsoft's Bing Copilot classification system
- **Web Navigation**: Website interaction and navigation agents
- **Competitive Programming**: Agents with episodic memory and human collaboration for solving complex programming problems
- **Complex Data Extraction**: Function calling for intricate extraction tasks

## LangGraph Platform

### Authentication & Access Control
- **Custom Authentication**: OAuth2 implementation for user authorization
- **Resource Authorization**: Private conversation management
- **Authentication Provider Integration**: User account validation with OAuth2
