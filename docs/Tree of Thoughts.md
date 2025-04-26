# Tree of Thoughts

## Metadata
- **url**: https://langchain-ai.github.io/langgraph/tutorials/tot/tot/

## Overview
Tree of Thoughts (ToT) is a general LLM agent search algorithm that combines reflection with structured search methods. Developed by Yao et al., it enables language models to explore multiple reasoning paths for complex problem-solving by maintaining a tree of thoughts and evaluating intermediate steps.

## Key Concepts
- **Expansion**: Generate multiple candidate solutions to a problem
- **Scoring**: Measure the quality of candidate solutions
- **Pruning**: Retain only the top K best candidates
- **Iterative Search**: Continue the expand-score-prune cycle until a solution is found or depth limit is reached
- **Beam Search**: Maintain multiple promising solution paths simultaneously

## Prerequisites
- LangGraph and LangChain OpenAI packages
- Access to an LLM provider (e.g., OpenAI)
- Basic understanding of graph-based algorithms

## Implementation

### 1. Setting Up the Task
The example implementation demonstrates ToT by solving the "Game of 24" - a mathematical puzzle where players must combine four numbers using basic operations to reach exactly 24.

```python
import operator
from typing import List, Literal, Union, NamedTuple, Optional
from pydantic import BaseModel, Field

# Define the data structures
class Equation(BaseModel):
    """Formula combining provided numbers to reach 24"""
    tokens: List[Union[float, str]] = Field(
        description="Stack of tokens and operators in reverse-polish notation"
    )
    
    def compute(self) -> float:
        # Computation logic for evaluating the equation
        op_funcs = {"+": operator.add, "-": operator.sub, 
                   "*": operator.mul, "/": operator.truediv}
        stack = []
        for token in self.tokens:
            if isinstance(token, float):
                stack.append(token)
            else:
                b, a = stack.pop(), stack.pop()
                stack.append(op_funcs[token](a, b))
        return stack[0]

# Structures for representing candidates in the search
class Candidate(NamedTuple):
    candidate: Equation
    score: Optional[float] = None
    feedback: Optional[str] = None
```

### 2. The Expander Component
The expander generates candidate solutions using an LLM with structured output.

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

# Create the prompt
prompt = ChatPromptTemplate.from_messages([
    (
        "system",
        "You are playing the Game of 24. Using the provided numbers, create an equation that evaluates to 24.\n"
        "Submit exactly {k} guesses for this round.",
    ),
    ("user", "Solve the 24 game for these numbers: {problem}.{candidate}"),
])

# Set up the LLM with structured output
llm = ChatOpenAI(model="gpt-4o-mini")
bound_llm = llm.with_structured_output(GuessEquations)
solver = prompt | bound_llm

# Expansion function
def expand(state, *, config):
    """Generate next candidate solutions"""
    configurable = _ensure_configurable(config)
    candidate_str = "\n\n" + str(state["seed"]) if state.get("seed") else ""
    
    try:
        equation_submission = solver.invoke({
            "problem": state["problem"],
            "candidate": candidate_str,
            "k": configurable["k"],
        }, config=config)
    except Exception:
        return {"candidates": []}
    
    new_candidates = [
        Candidate(candidate=equation) for equation in equation_submission.equations
    ]
    return {"candidates": new_candidates}
```

### 3. The Scorer Component
The scorer evaluates candidate solutions based on task-specific criteria.

```python
def compute_score(problem, candidate):
    """Score the candidate solution"""
    numbers = list(map(int, problem.split()))
    
    # Check if all numbers are used exactly once
    used_numbers = [token for token in candidate.candidate.tokens if isinstance(token, float)]
    if sorted(used_numbers) != sorted(numbers):
        return ScoredCandidate(
            candidate=candidate.candidate, 
            score=0, 
            feedback="The equation must use all 4 numbers exactly once."
        )
    
    # Check if the equation evaluates to 24
    try:
        result = candidate.candidate.compute()
        score = 1 / (1 + abs(24 - result))
        feedback = f"Result: {result}"
    except Exception as e:
        score = 0
        feedback = f"Invalid equation. Error: {repr(e)}"
    
    return ScoredCandidate(candidate=candidate.candidate, score=score, feedback=feedback)

def score(state):
    """Evaluate all candidate generations"""
    candidates = state["candidates"]
    scored = []
    for candidate in candidates:
        scored.append(compute_score(state["problem"], candidate))
    return {"scored_candidates": scored, "candidates": "clear"}
```

### 4. The Pruning Component
The pruner selects the most promising candidates to continue the search.

```python
def prune(state, *, config):
    """Select best candidates for next iteration"""
    scored_candidates = state["scored_candidates"]
    beam_size = _ensure_configurable(config)["beam_size"]
    
    # Sort by score in descending order
    organized = sorted(
        scored_candidates, key=lambda candidate: candidate[1], reverse=True
    )
    pruned = organized[:beam_size]
    
    return {
        "candidates": pruned,        # Update starting point for next iteration
        "scored_candidates": "clear", # Clear old memory
        "depth": 1,                  # Increment depth
    }
```

### 5. Constructing the Search Graph
The full algorithm is constructed as a graph with conditional edges.

```python
from langgraph.graph import StateGraph
from langgraph.constants import Send
from langgraph.checkpoint.memory import MemorySaver

# Termination condition
def should_terminate(state, config):
    configurable = _ensure_configurable(config)
    solved = state["candidates"][0].score >= configurable["threshold"]
    
    if solved or state["depth"] >= configurable["max_depth"]:
        return "__end__"
    
    return [
        Send("expand", {**state, "seed": candidate})
        for candidate in state["candidates"]
    ]

# Create the graph
builder = StateGraph(state_schema=ToTState, config_schema=Configuration)

# Add nodes
builder.add_node(expand)
builder.add_node(score)
builder.add_node(prune)

# Add edges
builder.add_edge("expand", "score")
builder.add_edge("score", "prune")
builder.add_conditional_edges("prune", should_terminate, path_map=["expand", "__end__"])
builder.add_edge("__start__", "expand")

# Compile with memory-based checkpointer
graph = builder.compile(checkpointer=MemorySaver())
```

### 6. Execution
Running the ToT algorithm on a problem.

```python
config = {
    "configurable": {
        "thread_id": "test_1",
        "max_depth": 10,
        "beam_size": 3,
        "threshold": 0.9,
        "k": 5
    }
}

# Run the graph with streaming to see intermediate steps
for step in graph.stream({"problem": "1 5 7 12"}, config):
    print(step)

# Retrieve the final state
final_state = graph.get_state(config)
winning_solution = final_state.values["candidates"][0]
search_depth = final_state.values["depth"]

if winning_solution[1] == 1:
    print(f"Found a winning solution in {search_depth} steps: {winning_solution}")
else:
    print(f"Failed to find a winning solution in {search_depth} steps. Best guess: {winning_solution}")
```

## Benefits

### 1. Systematic Exploration
- Explores multiple reasoning paths simultaneously rather than committing to a single line of thought
- Reduces the chances of getting stuck in local optima

### 2. Quality Assessment
- Provides mechanisms to evaluate intermediate solution quality
- Allows the algorithm to focus computational resources on promising directions

### 3. Customizable Search Strategy
- Configurable parameters for search depth, beam size, and quality thresholds
- Can be adapted to different problem domains by modifying the expander and scorer

### 4. Combinable with Various Search Algorithms
- While the example shows beam search, the framework can be adapted to use other algorithms (e.g., DFS, BFS, or A*)
- Can integrate task-specific heuristics through the scoring function

### 5. Improved Problem-Solving Capability
- Enables LLMs to solve complex problems requiring multiple steps of reasoning
- Provides a structured approach to breaking down problems too complex for single-pass solving
