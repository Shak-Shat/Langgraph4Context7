# Complex Data Extraction with Function Calling

## Overview

Function calling is a powerful mechanism for integrating LLMs into your software stack. Even the most capable models like GPT-4 and Claude can struggle with complex functions, especially when schemas involve nesting or advanced data validation rules. This implementation demonstrates how to increase the reliability of function calling through validation with re-prompting techniques.

## Key Concepts

- **Function Calling**: A core primitive for LLM integration that enables structured data extraction
- **Validation with Re-prompting**: A technique that verifies LLM outputs against expected schemas and requests corrections when needed
- **Regular Extraction with Retries**: A simple approach where the LLM regenerates function calls completely when errors occur
- **JSONPatch-Based Corrections**: An advanced technique that only fixes the specific parts of the response that contain errors

## Prerequisites

To implement this solution, you'll need to install the following packages:

```python
# Install required packages
pip install -U langchain-anthropic langgraph
pip install -U jsonpatch  # For JSONPatch-based retries
```

Set up your API keys:

```python
import os
import getpass

def _set_env(var: str):
    if not os.environ.get(var):
        os.environ[var] = getpass.getpass(f"{var}: ")

_set_env("OPENAI_API_KEY")
```

## Implementation

### Basic Validator + Retry Graph

The core implementation uses a state graph that follows this approach:
1. Prompt the LLM to respond
2. Validate any tool calls it makes
3. If valid, return the result; if invalid, format the error and prompt the LLM to fix it

The implementation provides two strategies:
- **Regular Extraction with Retries**: The LLM regenerates the entire function call
- **JSONPatch-Based Retries**: The LLM generates a patch to fix only the incorrect parts

#### 1. Regular Extraction with Retries

```python
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ValidationNode
from langchain_core.language_models import BaseChatModel
from langchain_core.messages import AIMessage, BaseMessage, HumanMessage

def bind_validator_with_retries(
    llm: BaseChatModel,
    *,
    tools: list,
    tool_choice: Optional[str] = None,
    max_attempts: int = 3,
) -> Runnable[Union[List[AnyMessage], PromptValue], AIMessage]:
    """Binds validators + retry logic ensure validity of generated tool calls."""
    bound_llm = llm.bind_tools(tools, tool_choice=tool_choice)
    retry_strategy = RetryStrategy(max_attempts=max_attempts)
    validator = ValidationNode(tools)
    return _bind_validator_with_retries(
        bound_llm,
        validator=validator,
        tool_choice=tool_choice,
        retry_strategy=retry_strategy,
    ).with_config(metadata={"retry_strategy": "default"})
```

#### Example: Simple Function Validation

Using Pydantic for data validation:

```python
from pydantic import BaseModel, Field, field_validator

class Respond(BaseModel):
    """Use to generate the response. Always use when responding to the user"""
    reason: str = Field(description="Step-by-step justification for the answer.")
    answer: str
    
    @field_validator("answer")
    def reason_contains_apology(cls, answer: str):
        if "llama" not in answer.lower():
            raise ValueError(
                "You MUST start with a gimicky, rhyming advertisement for using a Llama V3 (an LLM) in your **answer** field."
                " Must be an instant hit. Must be weaved into the answer."
            )

tools = [Respond]

# Create the LLM
from langchain_anthropic import ChatAnthropic
from langchain_core.prompts import ChatPromptTemplate

llm = ChatAnthropic(model="claude-3-haiku-20240307")
bound_llm = bind_validator_with_retries(llm, tools=tools)
prompt = ChatPromptTemplate.from_messages([
    ("system", "Respond directly by calling the Respond function."),
    ("placeholder", "{messages}"),
])

chain = prompt | bound_llm
```

#### 2. JSONPatch-Based Retries

For complex nested schemas, JSONPatch-based retries often work better:

```python
def bind_validator_with_jsonpatch_retries(
    llm: BaseChatModel,
    *,
    tools: list,
    tool_choice: Optional[str] = None,
    max_attempts: int = 3,
) -> Runnable[Union[List[AnyMessage], PromptValue], AIMessage]:
    """Uses JSONPatch to correct validation errors in tool calls."""
    # Implementation for patch-based corrections
    # This allows the LLM to fix only specific parts of a response rather than regenerating everything
    
    class JsonPatch(BaseModel):
        """A JSON Patch document represents an operation to be performed on a JSON document."""
        op: Literal["add", "remove", "replace"] = Field(
            ...,
            description="The operation to be performed."
        )
        path: str = Field(
            ...,
            description="A JSON Pointer path that references a location within the target document."
        )
        value: Any = Field(
            ...,
            description="The value to be used within the operation."
        )
    
    class PatchFunctionParameters(BaseModel):
        """Respond with JSONPatch operations to correct validation errors."""
        tool_call_id: str = Field(...)
        reasoning: str = Field(...)
        patches: list[JsonPatch] = Field(...)
    
    # Create validator and retry strategy
    # [Implementation details...]
    
    return _bind_validator_with_retries(
        bound_llm,
        validator=validator,
        retry_strategy=retry_strategy,
        tool_choice=tool_choice,
    )
```

### Usage with Complex Schemas

When dealing with deeply nested schemas:

```python
class TranscriptSummary(BaseModel):
    metadata: TranscriptMetadata
    participants: List[Member]
    key_moments: List[KeyMoments]
    insightful_quotes: List[InsightfulQuote]
    overall_summary: str
    next_steps: List[str]
    other_stuff: List[OutputFormat]

# Regular retries often fail on complex schemas
try:
    results = chain.invoke({...})
except ValueError as e:
    print(repr(e))  # ValueError('Could not extract a valid value in 3 attempts.')

# JSONPatch retries can handle complex schemas
bound_llm = bind_validator_with_jsonpatch_retries(llm, tools=tools)
chain = prompt | bound_llm
results = chain.invoke({...})  # Successfully extracts data
```

## Benefits

1. **Increased Reliability**: Even with complex data structures, the system can reliably extract structured data
2. **Progressive Refinement**: The system can build on partial successes rather than starting from scratch
3. **Transparency**: Error messages provide clear feedback about what's wrong and how to fix it
4. **Efficiency**: JSONPatch-based retries only modify the parts that need fixing, preserving valid data

Retries are an effective way to reduce function calling failures. While more powerful LLMs may reduce the need for retries, data validation remains important for controlling how LLMs interact with the rest of your software stack. Monitoring retry rates using tools like LangSmith can help identify patterns of failures and opportunities for improvement.
