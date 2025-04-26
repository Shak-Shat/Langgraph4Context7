# Custom Prompt Template

## Overview
Custom prompt templates allow creating specialized templates with specific dynamic instructions for language models when default templates don't meet your needs.

## Key Concepts
- Custom prompt templates can dynamically generate prompts based on function analysis
- Two types: string prompt templates and chat prompt templates
- Custom templates must implement required interfaces

## Prerequisites
```python
import inspect
from langchain.prompts import StringPromptTemplate
from pydantic import BaseModel, validator
```

## Implementation

### Helper Function
```python
def get_source_code(function_name):
    # Get the source code of the function
    return inspect.getsource(function_name)
```

### Prompt Definition
```python
PROMPT = """\
Given the function name and source code, generate an English language explanation of the function.
Function Name: {function_name}
Source Code:
{source_code}
Explanation:
"""
```

### Custom Prompt Template Class
```python
class FunctionExplainerPromptTemplate(StringPromptTemplate, BaseModel):
    """A custom prompt template that takes in the function name as input, and formats the prompt template to provide the source code of the function."""

    @validator("input_variables")
    def validate_input_variables(cls, v):
        """Validate that the input variables are correct."""
        if len(v) != 1 or "function_name" not in v:
            raise ValueError("function_name must be the only input_variable.")
        return v

    def format(self, **kwargs) -> str:
        # Get the source code of the function
        source_code = get_source_code(kwargs["function_name"])

        # Generate the prompt to be sent to the language model
        prompt = PROMPT.format(
            function_name=kwargs["function_name"].__name__, source_code=source_code
        )
        return prompt

    def _prompt_type(self):
        return "function-explainer"
```

## Usage Example
```python
fn_explainer = FunctionExplainerPromptTemplate(input_variables=["function_name"])

# Generate a prompt for the function "get_source_code"
prompt = fn_explainer.format(function_name=get_source_code)
print(prompt)
```

Output:
```
Given the function name and source code, generate an English language explanation of the function.
Function Name: get_source_code
Source Code:
def get_source_code(function_name):
    # Get the source code of the function
    return inspect.getsource(function_name)

Explanation:
```

## Benefits
- Allows creating specialized prompt templates tailored to specific use cases
- Enables dynamic content generation based on analysis of provided objects
- Supports validation of inputs for robust template creation

## Considerations
- Custom templates require implementing specific interfaces (input_variables attribute and format method)
- Must handle potential errors when processing dynamic content
- Need to ensure proper memory management for large source code snippets
