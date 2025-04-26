# Breakpoints



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/concepts/breakpoints/
- **html**: Breakpoints¶

Breakpoints pause graph execution at specific points and enable stepping through execution step by step. Breakpoints are powered by LangGraph's persistence layer, which saves the state after each graph step. Breakpoints can also be used to enable human-in-the-loop workflows, though we recommend using the interrupt function for this purpose.

Requirements¶

To use breakpoints, you will need to:

Specify a checkpointer to save the graph state after each step.
Set breakpoints to specify where execution should pause.
Run the graph with a thread ID to pause execution at the breakpoint.
Resume execution using invoke/ainvoke/stream/astream (see The Command primitive).
Setting breakpoints¶

There are two places where you can set breakpoints:

Before or after a node executes by setting breakpoints at compile time or run time. We call these static breakpoints.
Inside a node using the NodeInterrupt exception.
Static breakpoints¶

Static breakpoints are triggered either before or after a node executes. You can set static breakpoints by specifying interrupt_before and interrupt_after at "compile" time or run time.

Compile time
Run time
graph = graph_builder.compile(

    interrupt_before=["node_a"], 

    interrupt_after=["node_b", "node_c"],

    checkpointer=..., # Specify a checkpointer

)



thread_config = {

    "configurable": {

        "thread_id": "some_thread"

    }

}



# Run the graph until the breakpoint

graph.invoke(inputs, config=thread_config)



# Optionally update the graph state based on user input

graph.update_state(update, config=thread_config)



# Resume the graph

graph.invoke(None, config=thread_config)


Static breakpoints can be especially useful for debugging if you want to step through the graph execution one node at a time or if you want to pause the graph execution at specific nodes.

NodeInterrupt exception¶

We recommend that you use the interrupt function instead of the NodeInterrupt exception if you're trying to implement human-in-the-loop workflows. The interrupt function is easier to use and more flexible.

NodeInterrupt exception
Additional Resources 📚¶
Conceptual Guide: Persistence: Read the persistence guide for more context about persistence.
Conceptual Guide: Human-in-the-loop: Read the human-in-the-loop guide for more context on integrating human feedback into LangGraph applications using breakpoints.
How to View and Update Past Graph State: Step-by-step instructions for working with graph state that demonstrate the replay and fork actions.
Comments
