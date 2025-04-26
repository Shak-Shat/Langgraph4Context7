# Partial Prompt Templates

## Overview
Partial prompt templates allow creating new templates with a subset of required values pre-filled, enabling more flexible prompt composition and eliminating the need to wait for all variables to be available simultaneously.

## Key Concepts
- Create templates with some variables pre-filled, requiring only remaining values later
- Two approaches: partial formatting with static string values or with dynamic functions
- Reduces the need to pass common or early-available values multiple times

## Prerequisites
```python
from langchain.prompts import PromptTemplate
from datetime import datetime
```

## Implementation

### Partial with String Values
```python
# Method 1: Using partial() method
prompt = PromptTemplate(template="{foo}{bar}", input_variables=["foo", "bar"])
partial_prompt = prompt.partial(foo="foo")
print(partial_prompt.format(bar="baz"))
```

Output:
```
foobaz
```

```python
# Method 2: Using partial_variables parameter during initialization
prompt = PromptTemplate(
    template="{foo}{bar}", 
    input_variables=["bar"], 
    partial_variables={"foo": "foo"}
)
print(prompt.format(bar="baz"))
```

Output:
```
foobaz
```

### Partial with Functions
```python
def _get_datetime():
    now = datetime.now()
    return now.strftime("%m/%d/%Y, %H:%M:%S")

# Method 1: Using partial() method
prompt = PromptTemplate(
    template="Tell me a {adjective} joke about the day {date}",
    input_variables=["adjective", "date"]
)
partial_prompt = prompt.partial(date=_get_datetime)
print(partial_prompt.format(adjective="funny"))
```

Output:
```
Tell me a funny joke about the day 02/27/2023, 22:15:16
```

```python
# Method 2: Using partial_variables parameter during initialization
prompt = PromptTemplate(
    template="Tell me a {adjective} joke about the day {date}",
    input_variables=["adjective"],
    partial_variables={"date": _get_datetime}
)
print(prompt.format(adjective="funny"))
```

Output:
```
Tell me a funny joke about the day 02/27/2023, 22:15:16
```

## Benefits
- Simplifies workflows where variables are received at different times
- Enables dynamic content generation without modifying template structure
- Reduces redundancy by automating common variable insertions
- Improves readability by separating static and dynamic template components

## Considerations
- Function partials are executed at format time, not at template creation time
- Performance implications when using complex functions in partial variables
- Be mindful of memory usage when partial functions access large datasets
