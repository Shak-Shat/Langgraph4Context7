# Prompt Pipelining

## Overview
Prompt pipelining provides a user-friendly interface for composing different parts of prompts together, enabling easy creation of complex prompts through component reuse. It works with both string prompts and chat prompts.

## Key Concepts
- Combine multiple prompt components through operator overloading
- String prompts join templates with strings
- Chat prompts build message sequences incrementally
- Support for variables across joined components

## Prerequisites
```python
from langchain.prompts import PromptTemplate
from langchain.chains import LLMChain
from langchain.chat_models import ChatOpenAI
from langchain.schema import AIMessage, HumanMessage, SystemMessage
```

## Implementation

### String Prompt Pipelining
```python
# Create a pipelined string prompt
prompt = (
    PromptTemplate.from_template("Tell me a joke about {topic}")
    + ", make it funny"
    + "\n\nand in {language}"
)

# Examine the resulting template
print(prompt)
```

Output:
```
PromptTemplate(input_variables=['language', 'topic'], output_parser=None, partial_variables={}, template='Tell me a joke about {topic}, make it funny\n\nand in {language}', template_format='f-string', validate_template=True)
```

```python
# Format the prompt with variables
formatted = prompt.format(topic="sports", language="spanish")
print(formatted)
```

Output:
```
'Tell me a joke about sports, make it funny\n\nand in spanish'
```

### Using in an LLMChain
```python
model = ChatOpenAI()
chain = LLMChain(llm=model, prompt=prompt)
result = chain.run(topic="sports", language="spanish")
print(result)
```

Output:
```
'¿Por qué el futbolista llevaba un paraguas al partido?\n\nPorque pronosticaban lluvia de goles.'
```

### Chat Prompt Pipelining
```python
# Initialize with a system message
prompt = SystemMessage(content="You are a nice pirate")

# Create a pipeline with multiple message types
new_prompt = (
    prompt + HumanMessage(content="hi") + AIMessage(content="what?") + "{input}"
)

# Format the messages
messages = new_prompt.format_messages(input="i said hi")
print(messages)
```

Output:
```
[SystemMessage(content='You are a nice pirate', additional_kwargs={}),
 HumanMessage(content='hi', additional_kwargs={}, example=False),
 AIMessage(content='what?', additional_kwargs={}, example=False),
 HumanMessage(content='i said hi', additional_kwargs={}, example=False)]
```

### Using Chat Pipeline in an LLMChain
```python
model = ChatOpenAI()
chain = LLMChain(llm=model, prompt=new_prompt)
result = chain.run("i said hi")
print(result)
```

Output:
```
'Oh, hello! How can I assist you today?'
```

## Benefits
- Simplifies creation of complex, multi-part prompts
- Enables reuse of common prompt components
- Provides intuitive syntax for prompt composition
- Works with both string-based and message-based prompts
- Maintains variable interpolation across joined components

## Considerations
- Variables must be consistently named across joined components
- Order matters, especially in chat prompts where message sequence is important
- String prompt pipelining concatenates text, which may affect formatting
- First element in string prompt pipelining must be a proper PromptTemplate
