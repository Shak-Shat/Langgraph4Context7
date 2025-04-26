# Memory in LangGraph

## Overview
Memory is a crucial component for AI agents to store, retrieve, and use information effectively. This document covers how to implement both short-term and long-term memory in LangGraph applications.

## Key Concepts
- **Short-term memory**: Thread-scoped memory for single conversations, managed as part of the agent's state
- **Long-term memory**: Cross-thread memory saved within custom namespaces, accessible across different conversations
- **Memory types**: Semantic (facts), Episodic (experiences), and Procedural (instructions/rules)
- **Memory management**: Balancing precision, recall, latency, and cost

## Prerequisites
```python
from typing import Union, Annotated
from typing_extensions import TypedDict
from langchain_core.messages import RemoveMessage, AIMessage, HumanMessage
from langgraph.graph import add_messages, MessagesState
from langchain_core.messages import trim_messages
from langchain_openai import ChatOpenAI
from langgraph.store.memory import InMemoryStore
```

## Implementation

### Short-term Memory Management
Short-term memory allows your application to remember previous interactions within a single thread. LangGraph manages this as part of the agent's state.

#### Managing Long Conversation History
When conversations grow longer, you may need to trim or summarize them to fit within context windows.

##### Editing Message Lists
```python
def manage_list(existing: list, updates: Union[list, dict]):
    if isinstance(updates, list):
        # Normal case, add to the history
        return existing + updates
    elif isinstance(updates, dict) and updates["type"] == "keep":
        # You get to decide what this looks like.
        # For example, you could simplify and just accept a string "DELETE"
        # and clear the entire list.
        return existing[updates["from"]:updates["to"]]
    # etc. We define how to interpret updates

class State(TypedDict):
    my_list: Annotated[list, manage_list]

def my_node(state: State):
    return {
        # We return an update for the field "my_list" saying to
        # keep only values from index -5 to the end (deleting the rest)
        "my_list": {"type": "keep", "from": -5, "to": None}
    }
```

##### Using RemoveMessage
```python
class State(TypedDict):
    # add_messages will default to upserting messages by ID to the existing list
    # if a RemoveMessage is returned, it will delete the message in the list by ID
    messages: Annotated[list, add_messages]

def my_node_1(state: State):
    # Add an AI message to the `messages` list in the state
    return {"messages": [AIMessage(content="Hi")]}

def my_node_2(state: State):
    # Delete all but the last 2 messages from the `messages` list in the state
    delete_messages = [RemoveMessage(id=m.id) for m in state['messages'][:-2]]
    return {"messages": delete_messages}
```

##### Summarizing Conversations
```python
class State(MessagesState):
    summary: str

def summarize_conversation(state: State):
    # First, we get any existing summary
    summary = state.get("summary", "")

    # Create our summarization prompt
    if summary:
        # A summary already exists
        summary_message = (
            f"This is a summary of the conversation to date: {summary}\n\n"
            "Extend the summary by taking into account the new messages above:"
        )
    else:
        summary_message = "Create a summary of the conversation above:"

    # Add prompt to our history
    messages = state["messages"] + [HumanMessage(content=summary_message)]
    response = model.invoke(messages)

    # Delete all but the 2 most recent messages
    delete_messages = [RemoveMessage(id=m.id) for m in state["messages"][:-2]]
    return {"summary": response.content, "messages": delete_messages}
```

##### Trimming Messages by Token Count
```python
trim_messages(
    messages,
    # Keep the last <= n_count tokens of the messages.
    strategy="last",
    # Remember to adjust based on your model
    # or else pass a custom token_encoder
    token_counter=ChatOpenAI(model="gpt-4"),
    # Remember to adjust based on the desired conversation
    # length
    max_tokens=45,
    # Most chat models expect that chat history starts with either:
    # (1) a HumanMessage or
    # (2) a SystemMessage followed by a HumanMessage
    start_on="human",
    # Most chat models expect that chat history ends with either:
    # (1) a HumanMessage or
    # (2) a ToolMessage
    end_on=("human", "tool"),
    # Usually, we want to keep the SystemMessage
    # if it's present in the original history.
    # The SystemMessage has special instructions for the model.
    include_system=True,
)
```

### Long-term Memory

Long-term memory allows systems to retain information across different conversations or sessions, organized in custom namespaces.

#### Storing Memories
```python
def embed(texts: list[str]) -> list[list[float]]:
    # Replace with an actual embedding function or LangChain embeddings object
    return [[1.0, 2.0] * len(texts)]

# InMemoryStore saves data to an in-memory dictionary. Use a DB-backed store in production use.
store = InMemoryStore(index={"embed": embed, "dims": 2})
user_id = "my-user"
application_context = "chitchat"
namespace = (user_id, application_context)
store.put(
    namespace,
    "a-memory",
    {
        "rules": [
            "User likes short, direct language",
            "User only speaks English & python",
        ],
        "my-key": "my-value",
    },
)
# get the "memory" by ID
item = store.get(namespace, "a-memory")
# search for "memories" within this namespace, filtering on content equivalence, sorted by vector similarity
items = store.search(
    namespace, filter={"my-key": "my-value"}, query="language preferences"
)
```

#### Memory Types

##### Semantic Memory (Facts)
You can implement semantic memory as:
- **Profile**: A single, continuously updated document of user information
- **Collection**: Multiple documents continuously updated and extended over time

##### Episodic Memory (Experiences)
Often implemented through few-shot example prompting, where agents learn from past sequences.

##### Procedural Memory (Instructions)
```python
# Node that *uses* the instructions
def call_model(state: State, store: BaseStore):
    namespace = ("agent_instructions", )
    instructions = store.get(namespace, key="agent_a")[0]
    # Application logic
    prompt = prompt_template.format(instructions=instructions.value["instructions"])
    ...

# Node that updates instructions
def update_instructions(state: State, store: BaseStore):
    namespace = ("instructions",)
    current_instructions = store.search(namespace)[0]
    # Memory logic
    prompt = prompt_template.format(instructions=instructions.value["instructions"], conversation=state["messages"])
    output = llm.invoke(prompt)
    new_instructions = output['new_instructions']
    store.put(("agent_instructions",), "agent_a", {"instructions": new_instructions})
    ...
```

## Usage Example
To implement memory in your agent:
1. Define your state with appropriate memory structures
2. Use reducers like `add_messages` to manage message history
3. Create nodes that can update, trim, or summarize memories as needed
4. For long-term memory, use a Store implementation appropriate to your needs

## Benefits
- Improved contextual understanding in conversations
- Better personalization through semantic memories
- Increased efficiency by avoiding repetition
- Enhanced ability to learn from past interactions

## Considerations
- Memory management impacts token usage and costs
- Long contexts may degrade model performance
- Writing memories "on the hot path" may increase latency
- Background memory creation requires careful scheduling
