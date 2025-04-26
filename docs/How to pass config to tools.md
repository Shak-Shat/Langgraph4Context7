# Passing Configuration to Tools

## Overview
This guide demonstrates how to securely pass runtime configuration values to tools in LangGraph applications. You'll learn how to let the language model control intended parameters while keeping sensitive values like user IDs secure by injecting them through application configuration.

## Key Concepts
- **Runtime Configuration**: Values passed to tools during execution rather than in the tool's schema
- **RunnableConfig**: A standard interface for passing configuration to LangChain and LangGraph components
- **User Context**: Securely associating tool operations with specific users without exposing IDs
- **Configurable Parameters**: Special parameters that are excluded from the LLM's tool schema

## Prerequisites
- LangGraph and LangChain packages installed
- A language model provider (like Anthropic or OpenAI)
- Understanding of basic LangGraph concepts and tool usage

```python
# Install required packages
pip install -U langgraph langchain-anthropic

# Set up environment variables
import os
os.environ["ANTHROPIC_API_KEY"] = "your-api-key"
```

## Implementation

### Creating Tools with Config Parameters
Define tools that require configuration values such as user IDs:

```python
from typing import List
from langchain_core.tools import tool
from langchain_core.runnables.config import RunnableConfig

# Simple in-memory database for this example
user_to_pets = {}

@tool
def update_favorite_pets(pets: List[str], config: RunnableConfig) -> str:
    """Add the list of favorite pets for a user.
    
    Args:
        pets: List of pet names to store
        config: Runtime configuration (will be automatically populated)
        
    Returns:
        Confirmation message
    """
    user_id = config.get("configurable", {}).get("user_id")
    if not user_id:
        return "Error: No user ID provided in configuration"
    
    user_to_pets[user_id] = pets
    return f"Successfully updated favorite pets for user"

@tool
def delete_favorite_pets(config: RunnableConfig) -> str:
    """Delete the favorite pets list for a user.
    
    Args:
        config: Runtime configuration (will be automatically populated)
        
    Returns:
        Confirmation message
    """
    user_id = config.get("configurable", {}).get("user_id")
    if not user_id:
        return "Error: No user ID provided in configuration"
    
    if user_id in user_to_pets:
        del user_to_pets[user_id]
        return "Successfully deleted your pets list"
    return "No pets list found to delete"

@tool
def list_favorite_pets(config: RunnableConfig) -> str:
    """List favorite pets for a user.
    
    Args:
        config: Runtime configuration (will be automatically populated)
        
    Returns:
        Comma-separated list of pets or message if none found
    """
    user_id = config.get("configurable", {}).get("user_id")
    if not user_id:
        return "Error: No user ID provided in configuration"
    
    pets = user_to_pets.get(user_id, [])
    if not pets:
        return "You haven't added any favorite pets yet"
    return f"Your favorite pets are: {', '.join(pets)}"
```

### Examining the Tool Schema
Verify that the `config` parameter is hidden from the LLM:

```python
# Check what parameters are visible to the LLM
schema = update_favorite_pets.tool_call_schema.schema()
print(schema)
# Note: Only 'pets' appears in the schema, 'config' is excluded
```

### Creating a Tool-Using Agent
Set up an agent that can use the configuration-based tools:

```python
from langchain_anthropic import ChatAnthropic
from langgraph.prebuilt import create_react_agent

# Initialize the model and tools
model = ChatAnthropic(model="claude-3-5-sonnet")
tools = [update_favorite_pets, delete_favorite_pets, list_favorite_pets]

# Create the agent graph
graph = create_react_agent(model, tools)
```

### Running the Agent with Configuration
Execute the agent while providing the necessary configuration:

```python
from langchain_core.messages import HumanMessage

# Clear any existing data
user_to_pets.clear()

# First conversation: Adding pets
inputs = {"messages": [HumanMessage(content="I want to add cats and dogs to my favorite pets list")]}
config = {"configurable": {"user_id": "user-123"}}

# Stream the execution
print("Adding pets:")
for chunk in graph.stream(inputs, config, stream_mode="values"):
    if "messages" in chunk and chunk["messages"]:
        last_message = chunk["messages"][-1]
        if hasattr(last_message, "content"):
            print(f"Agent: {last_message.content}")

# Show the current state
print(f"\nUser data after adding pets: {user_to_pets}")

# Second conversation: Retrieving pets
inputs = {"messages": [HumanMessage(content="What are my favorite pets?")]}
config = {"configurable": {"user_id": "user-123"}}

print("\nRetrieving pets:")
for chunk in graph.stream(inputs, config, stream_mode="values"):
    if "messages" in chunk and chunk["messages"]:
        last_message = chunk["messages"][-1]
        if hasattr(last_message, "content"):
            print(f"Agent: {last_message.content}")
```

### Using Different User IDs
Show how user data is isolated between different user IDs:

```python
# Add pets for a second user
inputs = {"messages": [HumanMessage(content="My favorite pets are fish and birds")]}
config = {"configurable": {"user_id": "user-456"}}

print("\nAdding pets for another user:")
for chunk in graph.stream(inputs, config, stream_mode="values"):
    if "messages" in chunk and chunk["messages"]:
        last_message = chunk["messages"][-1]
        if hasattr(last_message, "content"):
            print(f"Agent: {last_message.content}")

# Show the current state for both users
print(f"\nUser data for both users: {user_to_pets}")

# Retrieve pets for the second user
inputs = {"messages": [HumanMessage(content="List my favorite pets")]}
config = {"configurable": {"user_id": "user-456"}}

print("\nRetrieving pets for second user:")
for chunk in graph.stream(inputs, config, stream_mode="values"):
    if "messages" in chunk and chunk["messages"]:
        last_message = chunk["messages"][-1]
        if hasattr(last_message, "content"):
            print(f"Agent: {last_message.content}")
```

### Deleting User Data
Demonstrate how to securely remove user-specific data:

```python
# Delete pets for a user
inputs = {"messages": [HumanMessage(content="Please forget my favorite pets")]}
config = {"configurable": {"user_id": "user-123"}}

print("\nDeleting pets for first user:")
for chunk in graph.stream(inputs, config, stream_mode="values"):
    if "messages" in chunk and chunk["messages"]:
        last_message = chunk["messages"][-1]
        if hasattr(last_message, "content"):
            print(f"Agent: {last_message.content}")

# Show the updated state
print(f"\nUser data after deletion: {user_to_pets}")
```

## Usage Example
Here's a complete example of a multi-user preference management system:

```python
from typing import Dict, List, Any
from langchain_core.tools import tool
from langchain_core.runnables.config import RunnableConfig
from langchain_anthropic import ChatAnthropic
from langgraph.prebuilt import create_react_agent
from langchain_core.messages import HumanMessage

# Database for user preferences
user_preferences = {}

@tool
def set_preference(category: str, value: Any, config: RunnableConfig) -> str:
    """Set a user preference.
    
    Args:
        category: The preference category (e.g., 'theme', 'language')
        value: The preference value
        config: Runtime configuration
        
    Returns:
        Confirmation message
    """
    user_id = config.get("configurable", {}).get("user_id")
    if not user_id:
        return "Error: Missing user context"
    
    if user_id not in user_preferences:
        user_preferences[user_id] = {}
    
    user_preferences[user_id][category] = value
    return f"Successfully set {category} preference to {value}"

@tool
def get_preference(category: str, config: RunnableConfig) -> str:
    """Get a user preference.
    
    Args:
        category: The preference category to retrieve
        config: Runtime configuration
        
    Returns:
        The preference value or a message if not found
    """
    user_id = config.get("configurable", {}).get("user_id")
    if not user_id:
        return "Error: Missing user context"
    
    user_prefs = user_preferences.get(user_id, {})
    if category in user_prefs:
        return f"Your {category} preference is set to: {user_prefs[category]}"
    
    return f"You don't have a {category} preference set yet"

@tool
def list_preferences(config: RunnableConfig) -> str:
    """List all preferences for a user.
    
    Args:
        config: Runtime configuration
        
    Returns:
        List of preferences or a message if none found
    """
    user_id = config.get("configurable", {}).get("user_id")
    if not user_id:
        return "Error: Missing user context"
    
    user_prefs = user_preferences.get(user_id, {})
    if not user_prefs:
        return "You don't have any preferences set yet"
    
    prefs_list = [f"{k}: {v}" for k, v in user_prefs.items()]
    return "Your preferences:\n" + "\n".join(prefs_list)

# Create the agent
model = ChatAnthropic(model="claude-3-sonnet")
tools = [set_preference, get_preference, list_preferences]
agent = create_react_agent(model, tools)

# Example conversation
result = agent.invoke(
    {"messages": [HumanMessage(content="Please set my theme preference to dark mode")]},
    {"configurable": {"user_id": "user-789"}}
)

print(result["messages"][-1].content)
print(f"User preferences: {user_preferences}")
```

## Benefits
- **Security**: Keep sensitive user IDs and context out of LLM access
- **Clean Tool Definitions**: Tools remain focused on their core functionality without user management code
- **User Isolation**: Easily maintain separation between different users' data
- **Configuration Flexibility**: Change configuration values between runs without modifying tool code
- **Simplified Authentication**: Pass authentication context without exposing it to the model

## Considerations
- **Default Values**: Consider providing sensible defaults if configuration values are missing
- **Error Handling**: Add robust checks for missing or invalid configuration parameters
- **Documentation**: Clearly document which configuration values tools expect
- **Validation**: Validate user IDs and other sensitive configuration values 
- **Tool Design**: Tools using configuration should still be meaningful and clear to the LLM
