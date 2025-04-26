# Prompt Engineering in LangGraph Studio



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/cloud/how-tos/iterate_graph_studio/
- **html**: Prompt Engineering in LangGraph Studio¶

In LangGraph Studio you can iterate on the prompts used within your graph by utilizing the LangSmith Playground. To do so:

Open an existing thread or create a new one.
Within the thread log, any nodes that have made an LLM call will have a "View LLM Runs" button. Clicking this will open a popover with the LLM runs for that node.
Select the LLM run you want to edit. This will open the LangSmith Playground with the selected LLM run.

From here you can edit the prompt, test different model configurations and re-run just this LLM call without having to re-run the entire graph. When you are happy with your changes, you can copy the updated prompt back into your graph.

For more information on how to use the LangSmith Playground, see the LangSmith Playground documentation.

Comments
