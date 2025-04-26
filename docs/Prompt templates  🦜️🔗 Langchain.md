# Prompt Templates

## Overview
Prompt templates provide pre-defined recipes for generating prompts for language models. They include instructions, examples, and context appropriate for specific tasks, allowing for model-agnostic reusability across different language models.

## Key Concepts
- Templates can include variables that are filled at runtime
- Two main types: PromptTemplate (string-based) and ChatPromptTemplate (message-based)
- Templates support validation to ensure all required variables are provided
- Can be used as part of LangChain Expression Language (LCEL)

## Prerequisites
```python
from langchain.prompts import PromptTemplate, ChatPromptTemplate
from langchain.chat_models import ChatOpenAI
from langchain.prompts import HumanMessagePromptTemplate
from langchain.schema.messages import SystemMessage
```

## Implementation

### String Prompt Template

```python
# Create a template with variables
prompt_template = PromptTemplate.from_template(
    "Tell me a {adjective} joke about {content}."
)
prompt_template.format(adjective="funny", content="chickens")
```

Output:
```
'Tell me a funny joke about chickens.'
```

```python
# Template with no variables
prompt_template = PromptTemplate.from_template("Tell me a joke")
prompt_template.format()
```

Output:
```
'Tell me a joke'
```

#### Input Validation
```python
# This will raise an error because 'content' is missing from input_variables
try:
    invalid_prompt = PromptTemplate(
        input_variables=["adjective"],
        template="Tell me a {adjective} joke about {content}.",
    )
except Exception as e:
    print(f"Validation Error: {e}")
```

### Chat Prompt Template

```python
# Creating a chat template with tuple representation
chat_template = ChatPromptTemplate.from_messages(
    [
        ("system", "You are a helpful AI bot. Your name is {name}."),
        ("human", "Hello, how are you doing?"),
        ("ai", "I'm doing well, thanks!"),
        ("human", "{user_input}"),
    ]
)

messages = chat_template.format_messages(name="Bob", user_input="What is your name?")
```

```python
# Alternative approach using message objects
chat_template = ChatPromptTemplate.from_messages(
    [
        SystemMessage(
            content=(
                "You are a helpful assistant that re-writes the user's text to "
                "sound more upbeat."
            )
        ),
        HumanMessagePromptTemplate.from_template("{text}"),
    ]
)

llm = ChatOpenAI()
llm(chat_template.format_messages(text="i dont like eating tasty things."))
```

Output:
```
AIMessage(content='I absolutely love indulging in delicious treats!')
```

### Using with LCEL

```python
# Using the Runnable interface
prompt_val = prompt_template.invoke({"adjective": "funny", "content": "chickens"})

# Convert to string
prompt_val.to_string()
```

Output:
```
'Tell me a joke'
```

```python
# Convert to messages
prompt_val.to_messages()
```

Output:
```
[HumanMessage(content='Tell me a joke')]
```

```python
# Working with chat templates
chat_val = chat_template.invoke({"text": "i dont like eating tasty things."})

# Get messages
chat_val.to_messages()
```

Output:
```
[SystemMessage(content="You are a helpful assistant that re-writes the user's text to sound more upbeat."),
 HumanMessage(content='i dont like eating tasty things.')]
```

```python
# Convert to string format
chat_val.to_string()
```

Output:
```
"System: You are a helpful assistant that re-writes the user's text to sound more upbeat.\nHuman: i dont like eating tasty things."
```

## Benefits
- Promotes reusability across different models and tasks
- Simplifies prompt engineering and management
- Provides validation to catch template errors early
- Enables dynamic content generation based on variables
- Supports both string-based and chat-based models

## Considerations
- Different models may require different template formats
- Templates with many variables can become complex to manage
- Need to ensure all required variables are provided at runtime
- Consider performance implications for very large templates
