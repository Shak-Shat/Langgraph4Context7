# Corrective RAG (CRAG)

## Overview
Corrective-RAG (CRAG) is a strategy for Retrieval Augmented Generation that incorporates self-reflection and self-grading on retrieved documents. CRAG aims to improve the quality of generated responses by evaluating the relevance of retrieved documents and dynamically supplementing with additional data sources when necessary.

## Key Concepts
- **Document Grading**: Evaluating each retrieved document for relevance to the user's question
- **Knowledge Refinement**: Partitioning documents into "knowledge strips" and grading each strip
- **Dynamic Retrieval Expansion**: Falling back to additional data sources (like web search) when retrieved documents are insufficient
- **Query Rewriting**: Optimizing the initial query to improve web search results
- **Conditional Processing Flow**: Graph-based decision-making that adapts the retrieval pipeline based on document grading

## Prerequisites
```python
# Install required packages
pip install langchain_community tiktoken langchain-openai langchainhub chromadb langchain langgraph tavily-python

# Set environment variables for API access
import os
os.environ["OPENAI_API_KEY"] = "your-openai-api-key"
os.environ["TAVILY_API_KEY"] = "your-tavily-api-key"
```

## Implementation

### 1. Create Vector Store and Retriever

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_community.document_loaders import WebBaseLoader
from langchain_community.vectorstores import Chroma
from langchain_openai import OpenAIEmbeddings

# Example URLs to index
urls = [
    "https://lilianweng.github.io/posts/2023-06-23-agent/",
    "https://lilianweng.github.io/posts/2023-03-15-prompt-engineering/",
    "https://lilianweng.github.io/posts/2023-10-25-adv-attack-llm/",
]

# Load and process documents
docs = [WebBaseLoader(url).load() for url in urls]
docs_list = [item for sublist in docs for item in sublist]

# Split into chunks
text_splitter = RecursiveCharacterTextSplitter.from_tiktoken_encoder(
    chunk_size=250, chunk_overlap=0
)
doc_splits = text_splitter.split_documents(docs_list)

# Create vector store and retriever
vectorstore = Chroma.from_documents(
    documents=doc_splits,
    collection_name="rag-chroma",
    embedding=OpenAIEmbeddings(),
)
retriever = vectorstore.as_retriever()
```

### 2. Create Document Grader

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field

# Define data model for document grading
class GradeDocuments(BaseModel):
    """Binary score for relevance check on retrieved documents."""
    binary_score: str = Field(
        description="Documents are relevant to the question, 'yes' or 'no'"
    )

# Set up LLM with structured output
llm = ChatOpenAI(model="gpt-3.5-turbo-0125", temperature=0)
structured_llm_grader = llm.with_structured_output(GradeDocuments)

# Create grading prompt
system = """You are a grader assessing relevance of a retrieved document to a user question. 
    If the document contains keyword(s) or semantic meaning related to the question, grade it as relevant.
    Give a binary score 'yes' or 'no' score to indicate whether the document is relevant to the question."""
    
grade_prompt = ChatPromptTemplate.from_messages(
    [
        ("system", system),
        ("human", "Retrieved document: \n\n {document} \n\n User question: {question}"),
    ]
)

# Create retrieval grader chain
retrieval_grader = grade_prompt | structured_llm_grader
```

### 3. Create RAG Generation Chain

```python
from langchain import hub
from langchain_core.output_parsers import StrOutputParser

# Load RAG prompt from hub
prompt = hub.pull("rlm/rag-prompt")

# Set up LLM
llm = ChatOpenAI(model_name="gpt-3.5-turbo", temperature=0)

# Helper function to format documents
def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

# Create RAG chain
rag_chain = prompt | llm | StrOutputParser()
```

### 4. Create Query Rewriter for Web Search

```python
# Set up LLM
llm = ChatOpenAI(model="gpt-3.5-turbo-0125", temperature=0)

# Create query rewriting prompt
system = """You a question re-writer that converts an input question to a better version that is optimized
     for web search. Look at the input and try to reason about the underlying semantic intent / meaning."""
     
re_write_prompt = ChatPromptTemplate.from_messages(
    [
        ("system", system),
        (
            "human",
            "Here is the initial question: \n\n {question} \n Formulate an improved question.",
        ),
    ]
)

# Create question rewriter chain
question_rewriter = re_write_prompt | llm | StrOutputParser()
```

### 5. Set Up Web Search Tool

```python
from langchain_community.tools.tavily_search import TavilySearchResults

# Create web search tool
web_search_tool = TavilySearchResults(k=3)
```

### 6. Define Graph State and Node Functions

```python
from typing import List
from typing_extensions import TypedDict
from langchain.schema import Document

# Define graph state
class GraphState(TypedDict):
    """
    Represents the state of our graph.

    Attributes:
        question: question
        generation: LLM generation
        web_search: whether to add search
        documents: list of documents
    """
    question: str
    generation: str
    web_search: str
    documents: List[str]

# Node: Document Retrieval
def retrieve(state):
    """Retrieve documents from vector store"""
    question = state["question"]
    documents = retriever.invoke(question)
    return {"documents": documents, "question": question}

# Node: Document Grading
def grade_documents(state):
    """Determines whether the retrieved documents are relevant to the question."""
    question = state["question"]
    documents = state["documents"]
    
    # Score each doc
    filtered_docs = []
    web_search = "No"
    for d in documents:
        score = retrieval_grader.invoke(
            {"question": question, "document": d.page_content}
        )
        grade = score.binary_score
        if grade == "yes":
            filtered_docs.append(d)
        else:
            web_search = "Yes"
            continue
    return {"documents": filtered_docs, "question": question, "web_search": web_search}

# Node: Query Transformation
def transform_query(state):
    """Transform the query to produce a better question."""
    question = state["question"]
    documents = state["documents"]
    
    # Re-write question
    better_question = question_rewriter.invoke({"question": question})
    return {"documents": documents, "question": better_question}

# Node: Web Search
def web_search(state):
    """Web search based on the re-phrased question."""
    question = state["question"]
    documents = state["documents"]
    
    # Web search
    docs = web_search_tool.invoke({"query": question})
    web_results = "\n".join([d["content"] for d in docs])
    web_results = Document(page_content=web_results)
    documents.append(web_results)
    
    return {"documents": documents, "question": question}

# Node: Answer Generation
def generate(state):
    """Generate answer"""
    question = state["question"]
    documents = state["documents"]
    
    # RAG generation
    generation = rag_chain.invoke({"context": documents, "question": question})
    return {"documents": documents, "question": question, "generation": generation}

# Edge: Decision function
def decide_to_generate(state):
    """Determines whether to generate an answer, or re-generate a question."""
    web_search = state["web_search"]
    
    if web_search == "Yes":
        # Need to transform query and use web search
        return "transform_query"
    else:
        # We have relevant documents, so generate answer
        return "generate"
```

### 7. Build and Compile the Graph

```python
from langgraph.graph import END, StateGraph, START

# Create the graph
workflow = StateGraph(GraphState)

# Add nodes
workflow.add_node("retrieve", retrieve)
workflow.add_node("grade_documents", grade_documents)
workflow.add_node("generate", generate)
workflow.add_node("transform_query", transform_query)
workflow.add_node("web_search_node", web_search)

# Build graph edges
workflow.add_edge(START, "retrieve")
workflow.add_edge("retrieve", "grade_documents")
workflow.add_conditional_edges(
    "grade_documents",
    decide_to_generate,
    {
        "transform_query": "transform_query",
        "generate": "generate",
    },
)
workflow.add_edge("transform_query", "web_search_node")
workflow.add_edge("web_search_node", "generate")
workflow.add_edge("generate", END)

# Compile
app = workflow.compile()
```

## Example Usage

### Example 1: Query About Agent Memory

```python
from pprint import pprint

# Run query
inputs = {"question": "What are the types of agent memory?"}

# Stream output at each node (for understanding the flow)
for output in app.stream(inputs):
    for key, value in output.items():
        pprint(f"Node '{key}':")
    pprint("\n---\n")

# Sample output:
# 'Agents possess short-term memory, which is utilized for in-context learning, 
# and long-term memory, allowing them to retain and recall vast amounts of 
# information over extended periods. Some experts also classify working memory 
# as a distinct type, although it can be considered a part of short-term 
# memory in many cases.'
```

### Example 2: Query About Research Paper

```python
# Run query
inputs = {"question": "How does the AlphaCodium paper work?"}

# Sample output:
# 'The AlphaCodium paper functions by proposing a code-oriented iterative flow 
# that involves repeatedly running and fixing generated code against 
# input-output tests. Its key mechanisms include generating additional data 
# like problem reflection and test reasoning to aid the iterative process, as 
# well as enriching the code generation process. AlphaCodium aims to improve 
# the performance of Large Language Models on code problems by following a 
# test-based, multi-stage approach.'
```

## Execution Flow

1. **Retrieve documents** from vector store based on query
2. **Grade each document** for relevance to the question
3. **Decision point**:
   - If all documents are relevant → proceed to generation
   - If any documents are irrelevant → transform query for web search
4. **Web search** with transformed query (if needed)
5. **Generate answer** using all relevant documents

## Benefits

- Improves retrieval quality by filtering irrelevant documents
- Expands knowledge sources dynamically when vector store is insufficient
- Optimizes queries for better web search results
- Creates a more robust RAG pipeline with self-assessment capabilities
