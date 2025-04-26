# Chat Prompts

## Overview
Chat prompts are built around structured messages rather than plain text, enabling the creation of complex conversation flows for chat models. They provide a flexible way to create message-based interactions.

## Key Concepts
- Chat prompts use message templates instead of plain text
- Multiple message types: SystemMessagePromptTemplate, HumanMessagePromptTemplate, AIMessagePromptTemplate
- ChatPromptTemplate combines multiple message templates into a conversation flow

## Prerequisites
```python
from langchain.prompts import PromptTemplate
from langchain.prompts.chat import (
    ChatPromptTemplate,
    SystemMessagePromptTemplate,
    AIMessagePromptTemplate,
    HumanMessagePromptTemplate,
)
```

## Implementation

### Using from_template Method
```python
# Create a system message template
template="You are a helpful assistant that translates {input_language} to {output_language}."
system_message_prompt = SystemMessagePromptTemplate.from_template(template)

# Create a human message template
human_template="{text}"
human_message_prompt = HumanMessagePromptTemplate.from_template(human_template)

# Combine them into a chat prompt template
chat_prompt = ChatPromptTemplate.from_messages([system_message_prompt, human_message_prompt])

# Generate formatted messages and get a completion
chat(chat_prompt.format_prompt(
    input_language="English", 
    output_language="French", 
    text="I love programming."
).to_messages())
```

Output:
```
AIMessage(content="J'adore la programmation.", additional_kwargs={})
```

### Direct Construction Method
```python
# Create PromptTemplate first
prompt=PromptTemplate(
    template="You are a helpful assistant that translates {input_language} to {output_language}.",
    input_variables=["input_language", "output_language"],
)
# Use it to create a SystemMessagePromptTemplate
system_message_prompt = SystemMessagePromptTemplate(prompt=prompt)
```

## Benefits
- Provides a structured way to create complex conversational prompts
- Supports multiple message types for different conversational roles
- Enables templating within message structures
- Allows converting between message and string formats based on model requirements

## Considerations
- Chat prompts work with chat models, while string prompts work with LLMs
- Multiple message formats may require appropriate conversion for different models
- Format must match the expected input format of the target model
