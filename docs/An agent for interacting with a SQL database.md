# Building a SQL Database Agent

## Overview
This guide demonstrates how to create a LangGraph agent that interacts with SQL databases. You'll learn how to build a workflow that understands database schemas, generates correct SQL queries, executes them safely, and returns well-formatted responses to user questions about data.

## Key Concepts
- **Database Connection**: Methods for securely connecting to SQL databases
- **Schema Exploration**: Automated discovery and understanding of database structure
- **Query Generation**: LLM-based SQL query creation from natural language
- **Query Validation**: Verification steps to ensure generated SQL is safe and correct
- **Result Formatting**: Converting raw SQL results into user-friendly responses

## Prerequisites
- LangGraph and LangChain packages installed
- Access to an OpenAI API key
- A SQL database (we'll use SQLite in this example)
- Basic understanding of SQL concepts

```python
# Install required packages
pip install -U langgraph langchain_openai langchain_community

# Set up environment variables
import os
from getpass import getpass

if "OPENAI_API_KEY" not in os.environ:
    os.environ["OPENAI_API_KEY"] = getpass("Enter your OpenAI API key: ")
```

## Implementation

### Setting Up the Database
First, let's download and connect to a sample SQLite database (Chinook):

```python
import requests
import sqlite3
from langchain_community.utilities import SQLDatabase

# Download sample database
def setup_chinook_db():
    """Download and set up the Chinook sample database."""
    url = "https://storage.googleapis.com/benchmarks-artifacts/chinook/Chinook.db"
    
    # Check if file already exists
    if os.path.exists("Chinook.db"):
        print("Database already exists.")
        return
    
    # Download the file
    response = requests.get(url)
    if response.status_code == 200:
        with open("Chinook.db", "wb") as file:
            file.write(response.content)
        print("Chinook database downloaded successfully.")
    else:
        raise Exception(f"Failed to download database: {response.status_code}")

# Set up the database
setup_chinook_db()

# Connect to the database using LangChain's utility
db = SQLDatabase.from_uri("sqlite:///Chinook.db")

# View available tables
tables = db.get_usable_table_names()
print(f"Available tables: {tables}")
```

### Creating Database Tools
Now, let's define tools for interacting with the database:

```python
from langchain_community.agent_toolkits import SQLDatabaseToolkit
from langchain_core.tools import Tool, tool
from langchain_openai import ChatOpenAI

# Initialize language model
llm = ChatOpenAI(model="gpt-4o", temperature=0)

# Create SQL toolkit with standard tools
toolkit = SQLDatabaseToolkit(db=db, llm=llm)
tools = toolkit.get_tools()

# Extract specific tools we'll need
list_tables_tool = next(tool for tool in tools if tool.name == "sql_db_list_tables")
get_schema_tool = next(tool for tool in tools if tool.name == "sql_db_schema")
run_query_tool = next(tool for tool in tools if tool.name == "sql_db_query")

# Define a custom query execution tool with better error handling
@tool
def execute_sql_query(query: str) -> str:
    """
    Execute an SQL query against the database and return the results.
    
    Args:
        query: SQL query string to execute
        
    Returns:
        Query results or error message
    """
    try:
        result = db.run_no_throw(query)
        if "Error" in result:
            return f"Query error: {result}"
        return result
    except Exception as e:
        return f"Error executing query: {str(e)}"
```

### Building a Query Validator
Let's create a validation step to check queries before execution:

```python
from langchain_core.prompts import ChatPromptTemplate

# Define a system prompt for the query validator
validator_system_prompt = """You are an expert SQL reviewer. Your job is to check SQL queries before they are executed to ensure they are:
1. Syntactically correct
2. Safe to execute (no destructive operations)
3. Optimized for performance
4. Free of common SQL errors

If you find any issues:
- Correct minor syntax errors
- Flag potentially destructive operations (DROP, DELETE, UPDATE without WHERE)
- Suggest query optimizations

Return the corrected query or an explanation of why the query is unsafe.
"""

# Create a query validation chain
query_validator = (
    ChatPromptTemplate.from_messages([
        ("system", validator_system_prompt),
        ("human", "Please review this SQL query:\n\n{query}")
    ])
    | llm
)

# Function to validate and potentially fix a query
def validate_query(query: str) -> str:
    """
    Validate and potentially correct an SQL query.
    
    Args:
        query: SQL query to validate
        
    Returns:
        Validated or corrected query
    """
    validator_response = query_validator.invoke({"query": query})
    
    # Extract the corrected query from the response
    response_text = validator_response.content
    
    # A simple heuristic: if the response contains a SQL query (which typically has SELECT, etc.)
    # then extract that as the corrected query
    if "SELECT" in response_text and "FROM" in response_text:
        # Extract what appears to be the SQL query
        import re
        potential_query = re.search(r"```sql\n(.*?)\n```", response_text, re.DOTALL)
        
        if potential_query:
            return potential_query.group(1).strip()
        
        # If no code block, try to extract the query some other way
        lines = response_text.split("\n")
        for line in lines:
            if "SELECT" in line:
                return line.strip()
    
    # If no clear query was found, return the original
    return query
```

### Creating the SQL Agent Workflow
Now, let's define our LangGraph workflow:

```python
from typing_extensions import TypedDict
from typing import List, Optional
from langgraph.graph import StateGraph, START, END
from langchain_core.messages import HumanMessage, AIMessage, SystemMessage

# Define our state structure
class AgentState(TypedDict):
    messages: List  # Conversation history
    schema: Optional[str]  # Database schema information
    query: Optional[str]  # Generated SQL query
    result: Optional[str]  # Query execution results

# Step 1: Understand the user's request
def analyze_question(state: AgentState):
    """Extract relevant information from the user question."""
    # Get the last user message
    for message in reversed(state["messages"]):
        if isinstance(message, HumanMessage):
            question = message.content
            break
    else:
        return {"messages": state["messages"] + [AIMessage(content="No question found.")]}
    
    # Retrieve table names
    tables_info = list_tables_tool.invoke("")
    
    # Retrieve schema for potentially relevant tables
    schema_info = get_schema_tool.invoke(tables_info)
    
    # Store schema information
    return {
        "schema": schema_info,
        "messages": state["messages"] + [
            AIMessage(content=f"I'll help you query the database. I found these tables: {tables_info}")
        ]
    }

# Step 2: Generate SQL query
def generate_query(state: AgentState):
    """Generate an SQL query based on the user's question and database schema."""
    # Get the last user message
    for message in reversed(state["messages"]):
        if isinstance(message, HumanMessage):
            question = message.content
            break
    
    # Create a prompt for query generation
    query_generation_prompt = ChatPromptTemplate.from_messages([
        ("system", "You are an expert SQL query generator. Use the database schema to create an accurate SQL query."),
        ("human", "Database schema:\n{schema}\n\nGenerate an SQL query to answer this question: {question}")
    ])
    
    # Generate the query
    query_chain = query_generation_prompt | llm
    query_response = query_chain.invoke({
        "schema": state["schema"],
        "question": question
    })
    
    # Extract query from the response
    import re
    query_match = re.search(r"```sql\n(.*?)\n```", query_response.content, re.DOTALL)
    
    if query_match:
        query = query_match.group(1).strip()
    else:
        # If no code block, use the full response
        query = query_response.content
    
    # Validate the query
    validated_query = validate_query(query)
    
    return {
        "query": validated_query,
        "messages": state["messages"] + [
            AIMessage(content=f"I've generated this SQL query to answer your question:\n```sql\n{validated_query}\n```")
        ]
    }

# Step 3: Execute the query
def execute_query(state: AgentState):
    """Execute the generated SQL query against the database."""
    query = state["query"]
    
    # Execute the query
    result = execute_sql_query.invoke(query)
    
    return {
        "result": result,
        "messages": state["messages"] + [
            AIMessage(content=f"Here are the query results:\n```\n{result}\n```")
        ]
    }

# Step 4: Format results into a human-friendly response
def format_response(state: AgentState):
    """Format the query results into a clear, human-friendly response."""
    # Get the last user message
    for message in reversed(state["messages"]):
        if isinstance(message, HumanMessage):
            question = message.content
            break
    
    # Create a formatting prompt
    formatting_prompt = ChatPromptTemplate.from_messages([
        ("system", "You are a helpful assistant. Explain database query results in a clear, concise manner."),
        ("human", "Question: {question}\n\nSQL Query: {query}\n\nQuery Results: {result}\n\nPlease provide a clear explanation of these results.")
    ])
    
    # Generate formatted response
    format_chain = formatting_prompt | llm
    formatted_response = format_chain.invoke({
        "question": question,
        "query": state["query"],
        "result": state["result"]
    })
    
    return {
        "messages": state["messages"] + [AIMessage(content=formatted_response.content)]
    }

# Build the workflow graph
workflow = StateGraph(AgentState)

# Add nodes
workflow.add_node("analyze_question", analyze_question)
workflow.add_node("generate_query", generate_query)
workflow.add_node("execute_query", execute_query)
workflow.add_node("format_response", format_response)

# Define edges
workflow.add_edge(START, "analyze_question")
workflow.add_edge("analyze_question", "generate_query")
workflow.add_edge("generate_query", "execute_query")
workflow.add_edge("execute_query", "format_response")
workflow.add_edge("format_response", END)

# Compile the workflow
sql_agent = workflow.compile()
```

## Usage Example
Let's see our SQL agent in action:

```python
# Initialize the conversation
initial_state = {
    "messages": [
        SystemMessage(content="I am a helpful AI assistant that can answer questions about the Chinook music database."),
        HumanMessage(content="Who is the top-selling artist by total sales?")
    ],
    "schema": None,
    "query": None,
    "result": None
}

# Run the workflow
result = sql_agent.invoke(initial_state)

# Display the conversation
for message in result["messages"]:
    if isinstance(message, SystemMessage):
        print(f"SYSTEM: {message.content}")
    elif isinstance(message, HumanMessage):
        print(f"HUMAN: {message.content}")
    elif isinstance(message, AIMessage):
        print(f"AI: {message.content}")
    print("---")

# Try another question
follow_up_question = {
    "messages": result["messages"] + [
        HumanMessage(content="What are the top 3 best-selling genres?")
    ],
    "schema": None,
    "query": None,
    "result": None
}

result = sql_agent.invoke(follow_up_question)

# Display only the new messages
for message in result["messages"][-5:]:
    if isinstance(message, HumanMessage):
        print(f"HUMAN: {message.content}")
    elif isinstance(message, AIMessage):
        print(f"AI: {message.content}")
    print("---")
```

## Benefits
- **Accurate Query Generation**: The agent understands database structure to create precise SQL
- **Safety Validation**: Query checking prevents common errors and security issues
- **Natural Language Interface**: Users can ask questions without knowing SQL
- **Extensible Architecture**: The workflow can be enhanced with additional processing steps
- **Explanation Capabilities**: Results are explained clearly rather than just returning raw data

## Considerations
- **Database Access Control**: Ensure the agent only has read access for sensitive databases
- **Schema Complexity**: Very large databases may need additional processing for schema understanding
- **Query Optimization**: Consider adding specific optimization for complex queries
- **Error Handling**: Add retry logic for failed queries with different approaches
- **Result Size Management**: Implement pagination for large result sets
