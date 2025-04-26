# Semantic Search for Agent Memory

## Overview
A guide for implementing semantic search capabilities in your agent's memory store, allowing it to search for memories based on meaning rather than exact text matches.

## Key Concepts
- Vector embeddings enable similarity-based memory retrieval
- Multi-vector indexing for different aspects of memories
- Field-specific indexing for targeted searches
- Selective indexing to control what's searchable

## Prerequisites
```python
%pip install -U langgraph langchain-openai langchain
import os
```

## Implementation

### Creating a Store with Semantic Search
```python
from langchain.embeddings import init_embeddings
from langgraph.store.memory import InMemoryStore

# Create store with semantic search enabled
embeddings = init_embeddings("openai:text-embedding-3-small")
store = InMemoryStore(
    index={
        "embed": embeddings,
        "dims": 1536,
    }
)
```

### Storing and Retrieving Memories
```python
# Store some memories
store.put(("user_123", "memories"), "1", {"text": "I love pizza"})
store.put(("user_123", "memories"), "2", {"text": "I prefer Italian food"})
store.put(("user_123", "memories"), "3", {"text": "I don't like spicy food"})
store.put(("user_123", "memories"), "3", {"text": "I am studying econometrics"})
store.put(("user_123", "memories"), "3", {"text": "I am a plumber"})

# Find memories about food preferences
memories = store.search(("user_123", "memories"), query="I like food?", limit=5)

for memory in memories:
    print(f'Memory: {memory.value["text"]} (similarity: {memory.score})')
```

Example output:
```
Memory: I prefer Italian food (similarity: 0.46482669521168163)
Memory: I love pizza (similarity: 0.35514845174380766)
Memory: I am a plumber (similarity: 0.155698702336571)
```

### Using in a Simple Agent
```python
from typing import Optional
from langchain.chat_models import init_chat_model
from langgraph.store.base import BaseStore
from langgraph.graph import START, MessagesState, StateGraph

llm = init_chat_model("openai:gpt-4o-mini")

def chat(state, *, store: BaseStore):
    # Search based on user's last message
    items = store.search(
        ("user_123", "memories"), query=state["messages"][-1].content, limit=2
    )
    memories = "\n".join(item.value["text"] for item in items)
    memories = f"## Memories of user\n{memories}" if memories else ""
    response = llm.invoke(
        [
            {"role": "system", "content": f"You are a helpful assistant.\n{memories}"},
            *state["messages"],
        ]
    )
    return {"messages": [response]}

builder = StateGraph(MessagesState)
builder.add_node(chat)
builder.add_edge(START, "chat")
graph = builder.compile(store=store)

for message, metadata in graph.stream(
    input={"messages": [{"role": "user", "content": "I'm hungry"}]},
    stream_mode="messages",
):
    print(message.content, end="")
```

Example output:
```
What are you in the mood for? Since you love Italian food and pizza, would you like to order a pizza or try making one at home?
```

### Using in a ReAct Agent
```python
import uuid
from typing import Optional
from langchain.chat_models import init_chat_model
from langgraph.prebuilt import InjectedStore
from langgraph.store.base import BaseStore
from typing_extensions import Annotated
from langgraph.prebuilt import create_react_agent

def prepare_messages(state, *, store: BaseStore):
    # Search based on user's last message
    items = store.search(
        ("user_123", "memories"), query=state["messages"][-1].content, limit=2
    )
    memories = "\n".join(item.value["text"] for item in items)
    memories = f"## Memories of user\n{memories}" if memories else ""
    return [
        {"role": "system", "content": f"You are a helpful assistant.\n{memories}"}
    ] + state["messages"]

# You can also use the store directly within a tool!
def upsert_memory(
    content: str,
    *,
    memory_id: Optional[uuid.UUID] = None,
    store: Annotated[BaseStore, InjectedStore],
):
    """Upsert a memory in the database."""
    # The LLM can use this tool to store a new memory
    mem_id = memory_id or uuid.uuid4()
    store.put(
        ("user_123", "memories"),
        key=str(mem_id),
        value={"text": content},
    )
    return f"Stored memory {mem_id}"

agent = create_react_agent(
    init_chat_model("openai:gpt-4o-mini"),
    tools=[upsert_memory],
    # The 'prompt' function is run to prepare the messages for the LLM. It is called
    # right before each LLM call
    prompt=prepare_messages,
    store=store,
)

for message, metadata in agent.stream(
    input={"messages": [{"role": "user", "content": "I'm hungry"}]},
    stream_mode="messages",
):
    print(message.content, end="")
```

Example output:
```
What are you in the mood for? Since you love Italian food and pizza, maybe something in that realm would be great! Would you like suggestions for a specific dish or restaurant?
```

## Advanced Usage

### Multi-vector Indexing
```python
# Configure store to embed both memory content and emotional context
store = InMemoryStore(
    index={"embed": embeddings, "dims": 1536, "fields": ["memory", "emotional_context"]}
)
# Store memories with different content/emotion pairs
store.put(
    ("user_123", "memories"),
    "mem1",
    {
        "memory": "Had pizza with friends at Mario's",
        "emotional_context": "felt happy and connected",
        "this_isnt_indexed": "I prefer ravioli though",
    },
)
store.put(
    ("user_123", "memories"),
    "mem2",
    {
        "memory": "Ate alone at home",
        "emotional_context": "felt a bit lonely",
        "this_isnt_indexed": "I like pie",
    },
)

# Search focusing on emotional state - matches mem2
results = store.search(
    ("user_123", "memories"), query="times they felt isolated", limit=1
)
print("Expect mem 2")
for r in results:
    print(f"Item: {r.key}; Score ({r.score})")
    print(f"Memory: {r.value['memory']}")
    print(f"Emotion: {r.value['emotional_context']}\n")

# Search focusing on social eating - matches mem1
print("Expect mem1")
results = store.search(("user_123", "memories"), query="fun pizza", limit=1)
for r in results:
    print(f"Item: {r.key}; Score ({r.score})")
    print(f"Memory: {r.value['memory']}")
    print(f"Emotion: {r.value['emotional_context']}\n")

print("Expect random lower score (ravioli not indexed)")
results = store.search(("user_123", "memories"), query="ravioli", limit=1)
for r in results:
    print(f"Item: {r.key}; Score ({r.score})")
    print(f"Memory: {r.value['memory']}")
    print(f"Emotion: {r.value['emotional_context']}\n")
```

Example output:
```
Expect mem 2
Item: mem2; Score (0.5895009051396596)
Memory: Ate alone at home
Emotion: felt a bit lonely

Expect mem1
Item: mem1; Score (0.6207546534134083)
Memory: Had pizza with friends at Mario's
Emotion: felt happy and connected

Expect random lower score (ravioli not indexed)
Item: mem1; Score (0.2686278787315685)
Memory: Had pizza with friends at Mario's
Emotion: felt happy and connected
```

### Override Fields at Storage Time
```python
store = InMemoryStore(
    index={
        "embed": embeddings,
        "dims": 1536,
        "fields": ["memory"],
    }  # Default to embed memory field
)

# Store one memory with default indexing
store.put(
    ("user_123", "memories"),
    "mem1",
    {"memory": "I love spicy food", "context": "At a Thai restaurant"},
)

# Store another overriding which fields to embed
store.put(
    ("user_123", "memories"),
    "mem2",
    {"memory": "The restaurant was too loud", "context": "Dinner at an Italian place"},
    index=["context"],  # Override: only embed the context
)

# Search about food - matches mem1 (using default field)
print("Expect mem1")
results = store.search(
    ("user_123", "memories"), query="what food do they like", limit=1
)
for r in results:
    print(f"Item: {r.key}; Score ({r.score})")
    print(f"Memory: {r.value['memory']}")
    print(f"Context: {r.value['context']}\n")

# Search about restaurant atmosphere - matches mem2 (using overridden field)
print("Expect mem2")
results = store.search(
    ("user_123", "memories"), query="restaurant environment", limit=1
)
for r in results:
    print(f"Item: {r.key}; Score ({r.score})")
    print(f"Memory: {r.value['memory']}")
    print(f"Context: {r.value['context']}\n")
```

Example output:
```
Expect mem1
Item: mem1; Score (0.3374968677940555)
Memory: I love spicy food
Context: At a Thai restaurant

Expect mem2
Item: mem2; Score (0.36784461593247436)
Memory: The restaurant was too loud
Context: Dinner at an Italian place
```

### Disable Indexing for Specific Memories
```python
store = InMemoryStore(index={"embed": embeddings, "dims": 1536, "fields": ["memory"]})

# Store a normal indexed memory
store.put(
    ("user_123", "memories"),
    "mem1",
    {"memory": "I love chocolate ice cream", "type": "preference"},
)

# Store a system memory without indexing
store.put(
    ("user_123", "memories"),
    "mem2",
    {"memory": "User completed onboarding", "type": "system"},
    index=False,  # Disable indexing entirely
)

# Search about food preferences - finds mem1
print("Expect mem1")
results = store.search(("user_123", "memories"), query="what food preferences", limit=1)
for r in results:
    print(f"Item: {r.key}; Score ({r.score})")
    print(f"Memory: {r.value['memory']}")
    print(f"Type: {r.value['type']}\n")

# Search about onboarding - won't find mem2 (not indexed)
print("Expect low score (mem2 not indexed)")
results = store.search(("user_123", "memories"), query="onboarding status", limit=1)
for r in results:
    print(f"Item: {r.key}; Score ({r.score})")
    print(f"Memory: {r.value['memory']}")
    print(f"Type: {r.value['type']}\n")
```

Example output:
```
Expect mem1
Item: mem1; Score (0.32269984224327286)
Memory: I love chocolate ice cream
Type: preference

Expect low score (mem2 not indexed)
Item: mem1; Score (0.010241633698527089)
Memory: I love chocolate ice cream
Type: preference
```

## Benefits
- More natural retrieval based on semantic meaning rather than keywords
- Contextually aware memory access that understands concepts and relationships
- Ability to prioritize memories with stronger semantic relevance
- Flexible control over which memory aspects are searchable

## Considerations
- Requires embedding models which may have associated costs
- Vector dimensions must match the embedding model used
- Memory retrieval quality depends on embedding quality
- Should configure index fields carefully to balance search performance and relevance
