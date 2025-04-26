# Adding Breakpoints

## Overview
This guide demonstrates how to implement human-in-the-loop breakpoints in LangGraph workflows. Breakpoints allow you to pause execution at specific points in your graph, inspect the current state, and manually approve continuation or make adjustments before resuming execution.

## Key Concepts
- **Breakpoints**: Designated points in your graph where execution pauses for human intervention
- **Interrupt Conditions**: Rules that determine when execution should pause, such as before or after specific nodes
- **State Inspection**: Examination of graph state at the breakpoint to assess correctness
- **Resume Mechanism**: Method to continue execution after manual approval
- **Override Options**: Ability to modify state before resuming execution

## Prerequisites
- LangGraph SDK installed
- Access to a deployed LangGraph application
- Basic understanding of asynchronous programming

```python
# Install required packages
pip install langgraph-sdk

# Import necessary libraries
import asyncio
from langgraph_sdk import get_client
```

## Implementation

### Setting Up the Client
First, initialize the SDK client to communicate with your LangGraph deployment:

```python
async def setup_client():
    # Connect to your LangGraph deployment
    client = get_client(url="https://your-deployment-url.com")
    
    # Create a new thread for this conversation
    thread = await client.threads.create()
    
    return client, thread
```

### Adding Interruption Points
Specify breakpoints by designating nodes where execution should pause:

```python
async def run_with_breakpoints(client, thread, assistant_id):
    # Define the input for the graph
    input_data = {
        "messages": [
            {"role": "user", "content": "Please analyze this financial report and suggest investments"}
        ]
    }
    
    # Stream execution with interruption points
    run = None
    async for chunk in client.runs.stream(
        thread["thread_id"],
        assistant_id,
        input=input_data,
        stream_mode="updates",
        # Pause before critical analysis or action nodes
        interrupt_before=["analysis_node", "recommendation_node"]
    ):
        print(f"Event: {chunk.event}")
        
        # Store run information when interrupted
        if chunk.event == "interrupt" and not run:
            run = chunk.data
            print(f"Execution paused at: {run.get('status_info', {}).get('interrupted_before')}")
            
    return run
```

### Inspecting State at Breakpoints
When execution pauses, you can examine the current state:

```python
async def inspect_state(client, thread, run_id):
    # Retrieve the current state of the run
    run_state = await client.runs.get(thread["thread_id"], run_id)
    
    # Extract and display relevant information
    current_node = run_state.get("status_info", {}).get("interrupted_before")
    state = run_state.get("state", {})
    
    print(f"Currently at node: {current_node}")
    print(f"Current state: {state}")
    
    return state
```

### Resuming Execution
After reviewing, approve continuation or provide modified state:

```python
async def resume_execution(client, thread, run_id, override=None):
    # Resume execution with optional state modifications
    await client.runs.resume(
        thread["thread_id"],
        run_id,
        override=override  # Pass None to continue with unmodified state
    )
    
    print("Execution resumed")
    
    # Stream the remaining execution
    async for chunk in client.runs.stream(
        thread["thread_id"],
        run_id,
        stream_mode="updates"
    ):
        print(f"Event: {chunk.event}")
        if chunk.event == "interrupt":
            # Handle any additional interruptions
            print("Another breakpoint reached")
            return chunk.data
```

## Usage Example
Here's a complete example of implementing breakpoints in a financial advisor agent:

```python
async def financial_advisor_with_breakpoints():
    # Setup
    client, thread = await setup_client()
    assistant_id = "financial_advisor"
    
    # Initial run with breakpoints
    print("Starting financial analysis with human approval steps...")
    run = await run_with_breakpoints(client, thread, assistant_id)
    
    # Execution paused at first breakpoint (analysis_node)
    if run:
        state = await inspect_state(client, thread, run["id"])
        
        # Hypothetical: User reviews analysis and approves
        user_approval = input("Review the analysis. Type 'approve' to continue or 'modify' to change state: ")
        
        if user_approval == "approve":
            # Continue without modifications
            run = await resume_execution(client, thread, run["id"])
        elif user_approval == "modify":
            # Example: Override with corrected analysis
            override = {
                "analysis": {
                    "risk_assessment": "medium",
                    "market_outlook": "positive",
                    "corrected_by_human": True
                }
            }
            run = await resume_execution(client, thread, run["id"], override)
    
    # Execution paused at second breakpoint (recommendation_node)
    if run:
        state = await inspect_state(client, thread, run["id"])
        
        # Hypothetical: User reviews recommendations
        final_approval = input("Review investment recommendations. Type 'approve' to finalize: ")
        
        if final_approval == "approve":
            # Complete the execution
            await resume_execution(client, thread, run["id"])

# Run the example
asyncio.run(financial_advisor_with_breakpoints())
```

## Benefits
- **Human Oversight**: Critical decisions can be verified before proceeding
- **Quality Control**: Experts can validate AI outputs at key decision points
- **Error Prevention**: Catch and correct issues before they propagate
- **Explainability**: Better understanding of the workflow through step-by-step inspection
- **Safety**: Additional layer of protection for sensitive operations

## Considerations
- **Performance Impact**: Breakpoints introduce latency and require human availability
- **User Experience**: Design intuitive interfaces for state inspection and modification
- **Stateful Management**: Ensure proper handling of multiple paused executions
- **Strategic Placement**: Add breakpoints only at critical decision points
- **Timeout Handling**: Consider implementing timeouts for interrupted runs

## References
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/cloud/how-tos/human_in_the_loop_breakpoint/)
