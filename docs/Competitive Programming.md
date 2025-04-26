# Competitive Programming



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/tutorials/usaco/usaco/
- **html**: Home
Guides
Tutorials
Experimental
Competitive Programming¶

In this tutorial, you will build a computing olympiad agent that leverages three complementary techniques to boost performance: reflection, retrieval, and human-in-the-loop collaboration. These techniques and data are all adapted from the paper "Can Language Models Solve Olympiad Programming?" by Quan Shi, Michael Tang, Karthik Narasimhan, and Shunyu Yao. You can check out their paper at the following link:

You will construct an agentic graph capable of answering programming questions of increasing difficulty.

Reflection: In part 1, you will create a zero-shot tool calling agent and prompt it to reflect on the test case results to correct its initial errors. This is similar to the agent the paper reported as having a pass rate of 12.38 on the USACO benchmark.
Retrieval: In Part 2, you will implement an initial retrieval step as "episodic memory" for the agent that retrieves high-quality few-shot examples from our corpora of programming problems to help solve the bronze level question. This agent is similar to the one the paper benchmarked at 20.2.
Human-in-the-loop: In part 3, you will use interrupt_after to let the user copilot the agent to a better answer. The benchmark performance then is constrained only by the competitiveness of the human it is paired with.

Your final agent graph will be structured like the diagram below:

Parts 1 and 2 are analogous to the systems benchmarked in the paper as having a pass rate of 12.38 and 20.2 respectively.

While LLMs are not yet capable of autonomously solving all these problems, we can design the system that far surpasses the capabilities of a basic ReAct agent at answering these questions.

Before diving in, let's set up our machine. This will involve installing dependencies, fetching the dataset, and defining a utility function.

Setup¶

For this tutorial, we will need to install some dependencies, fetch the Olympiad dataset, and define a utility function to help run the candidate solutions to see if they pass the test cases.

First, let's install the required packages and set our API keys

%%capture --no-stderr

%pip install -U langgraph langsmith langchain_anthropic datasets langchain langchainhub

import getpass

import os





def _get_env(var: str):

    if not os.environ.get(var):

        os.environ[var] = getpass.getpass(f"{var}: ")





_get_env("ANTHROPIC_API_KEY")


Set up LangSmith for LangGraph development

Sign up for LangSmith to quickly spot issues and improve the performance of your LangGraph projects. LangSmith lets you use trace data to debug, test, and monitor your LLM apps built with LangGraph — read more about how to get started here.

Data¶

Fetch the USACO benchmark data using the util below:

import os

import zipfile



import datasets

import requests



usaco_url = "https://storage.googleapis.com/benchmarks-artifacts/usaco/usaco_sampled_with_tests.zip"

zip_path = "usaco.zip"

extract_path = "usaco_datasets"



response = requests.get(usaco_url)

with open(zip_path, "wb") as file:

    file.write(response.content)



with zipfile.ZipFile(zip_path, "r") as zip_ref:

    zip_ref.extractall(extract_path)



os.remove(zip_path)



ds = datasets.load_from_disk(os.path.join(extract_path, "usaco_v3_sampled_with_tests"))

Test Evaluation Utils¶

We also need a way to evaluate our generated code. We will use this unsafe code execution program to run the generated code against our test cases. Note: The code below runs arbitrary code on your local machine! Proceed with caution.

import multiprocessing

import queue

import subprocess

import sys

import time

import traceback



multiprocessing.set_start_method("fork", force=True)

# WARNING

# This program exists to execute untrusted model-generated code. Although

# it is highly unlikely that model-generated code will do something overtly

# malicious in response to this test suite, model-generated code may act

# destructively due to a lack of model capability or alignment.

# Users are strongly encouraged to sandbox this evaluation suite so that it

# does not perform destructive actions on their host or network.

# Proceed at your own risk:





def exec_program(q, program, input_data, expected_output, timeout):

    try:

        start_time = time.time()

        process = subprocess.Popen(

            [sys.executable, "-c", program],

            stdin=subprocess.PIPE,

            stdout=subprocess.PIPE,

            stderr=subprocess.PIPE,

            text=True,

        )

        stdout, stderr = process.communicate(input=input_data, timeout=timeout)

        if time.time() - start_time > timeout:

            raise TimeoutError("Execution timed out.")

        if process.returncode != 0:

            q.put(f"failed: {stderr}")

        else:

            if stdout.strip() == expected_output.strip():

                q.put("passed")

            else:

                q.put(f"wrong answer. Expected '{expected_output}', got '{stdout}'")

    except subprocess.TimeoutExpired:

        process.kill()

        q.put("timed out")

    except Exception:

        q.put(f"failed: {traceback.format_exc()}")





def check_correctness(

    program: str, input_data: str, expected_output: str, timeout: float

) -> str:

    q = multiprocessing.Queue()

    process = multiprocessing.Process(

        target=exec_program, args=(q, program, input_data, expected_output, timeout)

    )

    process.start()

    process.join(timeout=timeout + 1)

    if process.is_alive():

        process.terminate()

        process.join()

        result = "timed out"

    else:

        try:

            result = q.get_nowait()

        except queue.Empty:

            result = "no result returned"

    return result


Let's check an example program and output to see how it works:

program_code = "print('hello, world!')"

input_data = ""

expected_output = "hello, world!"

timeout = 2



test_result = check_correctness(program_code, input_data, expected_output, timeout)

print("Example 1: ", test_result)

test_result = check_correctness("print('goodbye')", input_data, "hi there", timeout)

print("Example 2: ", test_result)

Example 1:  passed

Example 2:  wrong answer. Expected 'hi there', got 'goodbye

'


Part 1: Zero-Shot with Reflection¶

In our first section, we will build a simple zero-shot tool-calling agent to try to solve these problems. We will incorporate a simple form of reflection directly in the agent's tool calling schema by adding a "reasoning" field. Furthermore, Claude was trained to "reason" with freeform text prior to invoking any tools. Together, this should induce reflective "chain-of-thought" prompting.

Note: this diverges somewhat from the paper's implementation, which uses an explicit reflection step with a variation of the Reflexion prompt.

By the end of this section, we will have built a reflective zero-shot programming agent that looks like the section marked "Part 1" in the system diagram below:

State¶

LangGraph's main primitive is the StateGraph, which you use to define an agent as a controllable state machine. The graph has node's (python functions) that perform the work, and edges that define how to route between the nodes. The State defines the interface between each node and carries all the information your agent needs.

Below, define a State for our programming olympiad agent. The messages will track the sequence of submissions (and test case feedback) as chat history. The status field will flip from in_progress to success if the submission passes all test cases. The other fields (test_cases, runtime_limit) are used by the evaluation node to test the agent's submissions. These values are not seen by the agent itself.

from typing import Annotated



from typing_extensions import TypedDict



from langgraph.graph.message import AnyMessage, add_messages





class TestCase(TypedDict):

    inputs: str

    outputs: str





class State(TypedDict):

    # Append-only chat memory so the agent can try to recover from initial mistakes.

    messages: Annotated[list[AnyMessage], add_messages]

    # From the dataset. These are used for testing.

    test_cases: list[TestCase]

    runtime_limit: int

    status: str


API Reference: add_messages

Now, convert the dataset into inputs our graph will accept.

input_states = [

    {

        "messages": [("user", row["description"])],

        "test_cases": row["test_cases"],

        "runtime_limit": row["runtime_limit"],

        "status": "in_progress",

        "problem_level": row["problem_level"],

    }

    for row in ds

]

Node 1: Solver¶

Create a solver node that prompts an LLM "agent" to use a writePython tool to generate the submitted code.

Using Pydantic with LangChain

This notebook uses Pydantic v2 BaseModel, which requires langchain-core >= 0.3. Using langchain-core < 0.3 will result in errors due to mixing of Pydantic v1 and v2 BaseModels.

from langchain_core.language_models import BaseChatModel

from langchain_core.prompts import ChatPromptTemplate



from pydantic import BaseModel, Field





class writePython(BaseModel):

    """Write python code that resolves the problem."""



    reasoning: str = Field(..., description="Conceptual solution.")

    pseudocode: str = Field(..., description="Detailed English pseudocode.")

    code: str = Field(..., description="Valid Python 3 solution to the problem")





class Solver:

    def __init__(self, llm: BaseChatModel, prompt: ChatPromptTemplate):

        self.runnable = prompt | llm.bind_tools([writePython])



    def __call__(self, state: State) -> dict:

        # Our agent only can see the "messages" and will ignore the test info

        return {"messages": [self.runnable.invoke({"messages": state["messages"]})]}


API Reference: BaseChatModel | ChatPromptTemplate

Now, create the solver below. We'll use Claude Opus

from langchain import hub

from langchain_anthropic import ChatAnthropic



# For this section, we are testing zero-shot performance and won't have

# any examples. Partial them out to pre-fill the template.

prompt = hub.pull("wfh/usaco-draft-solver").partial(examples="")

print("*" * 35 + "Prompt" + "*" * 35)

prompt.pretty_print()



# Use Haiku if you want to save $$ while (almost) never correctly answering the question

# llm = ChatAnthropic(model="claude-3-haiku-20240307")

llm = ChatAnthropic(model="claude-3-opus-20240229")



solver = Solver(llm, prompt)


API Reference: ChatAnthropic

***********************************Prompt***********************************

================================[1m System Message [0m================================



You are a world-class competitive programmer.

Please reply with a Python 3 solution to the problem below. 

First, reason through the problem and conceptualize a solution.

Then write detailed pseudocode to uncover any potential logical errors or omissions.

Finally output the working Python code for your solution, ensuring to fix any errors uncovered while writing pseudocode.



No outside libraries are allowed.[33;1m[1;3m{examples}[0m



=============================[1m Messages Placeholder [0m=============================



[33;1m[1;3m{messages}[0m


print("*" * 34 + " Example " + "*" * 34)

result = solver(

    {

        "messages": [

            (

                "user",

                "How do I get a perfectly random sample from an infinite stream",

            )

        ]

    }

)

result["messages"][0].pretty_print()

# Could expand to include (1)

# 1. Restate the problem in plain English

# 2. Closely following the explanation, restate and explain the solution in plain English

# 3. Write a pseudocode solution

# 4. Output the final Python solution with your solution steps in comments.

********************************** Example **********************************

==================================[1m Ai Message [0m==================================



[{'text': "<thinking>\nTo address this problem, we need to use the writePython function, which requires the following parameters:\n- reasoning: a conceptual solution to the problem\n- pseudocode: detailed pseudocode for the solution\n- code: working Python code implementing the solution\n\nThe key aspects to address in the solution are:\n1. We have an infinite stream, so we can't store all elements. Need an online algorithm.\n2. Need to ensure each element has an equal probability of being in the final sample.\n\nI believe I have enough information to provide values for all the required parameters.\n</thinking>", 'type': 'text'}, {'id': 'toolu_01UqpLYyueky5GtYMidS9oLF', 'input': {'reasoning': 'To get a perfectly random sample of size k from an infinite stream:\n\n1. Store the first k elements in an array (reservoir). \n2. For each ith element after the kth element (i > k):\n   - Generate a random integer j between 0 and i (inclusive)\n   - If j < k, replace the jth element of the reservoir with the ith element\n3. At the end, the reservoir contains the random sample.\n\nThis works because for any element, when we process the nth element, the probability that it is in the reservoir is:\n- k/n when n <= k (first k elements always selected)\n- k/n * k/(n-1) * k/(n-2) * ... * k/(k+1) = k/n when n > k\n\nSo any element has k/n probability of being in final reservoir, giving a perfectly random sample.', 'pseudocode': '\`\`\`\nfunction selectKItems(stream, k):\n    reservoir = [0..k-1]  # store first k elements\n\n    i = k\n    while stream has next item:\n        item = stream.next()\n        j = random(0, i)  # generate random index between 0 and i\n        if j < k:\n            reservoir[j] = item  # replace element at random index with new item\n        i += 1\n\n    return reservoir\n\`\`\`', 'code': 'import random\n\ndef reservoir_sampling(stream, k):\n    reservoir = []\n    \n    # Store first k elements in reservoir\n    for i in range(k):\n        reservoir.append(next(stream))\n\n    i = k\n    for item in stream:\n        # Generate random index between 0 and i\n        j = random.randint(0, i) \n        \n        # Replace element at random index with new item\n        if j < k:\n            reservoir[j] = item\n        i += 1\n\n    return reservoir'}, 'name': 'writePython', 'type': 'tool_use'}]


Node 2: Evaluate¶

Now define the "evaluate" node. This node takes the solver's submitted code and executes it against the test_cases in our State. This uses the unsafe check_correctness utility we defined in the setup above.

from langchain_core.messages import AIMessage, HumanMessage, ToolMessage





# This is the node we will add to the graph.

# Most tool-calling APIs require that the `ToolMessage` contain the ID

# of the

def format_tool_message(response: str, ai_message: AIMessage):

    return ToolMessage(

        content=response + "\nMake all fixes using the writePython tool.",

        tool_call_id=ai_message.tool_calls[0]["id"],

    )





def evaluate(state: State):

    test_cases = state["test_cases"]

    ai_message: AIMessage = state["messages"][-1]

    if not ai_message.tool_calls:

        return {

            "messages": [

                HumanMessage(

                    content="No code submitted. Please try again using the correct python code."

                )

            ]

        }

    try:

        code = ai_message.tool_calls[0]["args"]["code"]

    except Exception as e:

        return {"messages": [format_tool_message(repr(e), ai_message)]}

    num_test_cases = len(test_cases)

    succeeded = 0

    test_results = []

    # TODO: Multiprocess

    for test_case in test_cases:

        input_data = test_case["inputs"]

        expected_output = test_case["outputs"]

        test_result = check_correctness(code, input_data, expected_output, timeout)

        test_results.append(test_result)

        if test_result == "passed":

            succeeded += 1

    pass_rate = succeeded / num_test_cases if num_test_cases else "N/A"

    if pass_rate == 1:

        return {"status": "success"}



    responses = "\n".join(

        [f"<test id={i}>\n{r}\n</test>" for i, r in enumerate(test_results)]

    )

    response = f"Incorrect submission. Please respond with updated code.\nPass rate: {succeeded}/{num_test_cases}\nResults:\n{responses}"

    formatted_message = format_tool_message(response, ai_message)

    return {"messages": [formatted_message]}


API Reference: AIMessage | HumanMessage | ToolMessage

Create Graph¶

Now, put it all together! Once you've defined each node, defining the connectivity / state transitions is fairly easy.

Our Zero-shot graph defines a loop. If we visualize the data flow, we want the logic to: 1. First go to the solver, which attempts a first solution. 2. Next go to the evaluate node, which tests the solution. 3. If the solution passes, end, otherwise, return to the solver to try again.

In LangGraph, we use conditional_edges to define state transitions that contain conditional logic. Below, define the graph, adding a control_edge to handle step (3) above.

from langgraph.graph import END, StateGraph, START



builder = StateGraph(State)

builder.add_node("solver", solver)

builder.add_edge(START, "solver")

builder.add_node("evaluate", evaluate)

builder.add_edge("solver", "evaluate")





def control_edge(state: State):

    if state.get("status") == "success":

        return END

    return "solver"





builder.add_conditional_edges("evaluate", control_edge, {END: END, "solver": "solver"})

graph = builder.compile()


API Reference: END | StateGraph | START

from IPython.display import Image, display



try:

    display(Image(graph.get_graph().draw_mermaid_png()))

except Exception:

    # This requires some extra dependencies and is optional

    pass


Now that we've created our graph, let's see the type of question it will have to solve.

input_state = input_states[0].copy()

# We will reduce the test cases to speed this notebook up

input_state["test_cases"] = input_state["test_cases"][:3]

print(input_state["messages"][0][1])

Farmer John has $N$ ($1 \leq N \leq 2 \cdot 10^5$) farms, numbered from $1$ to

$N$. It is known that FJ closes farm $i$ at time $c_i$. Bessie wakes up at time

$S$, and wants to maximize the productivity of her day by visiting as many farms

as possible before they close. She plans to visit farm $i$ on time $t_i + S$.

Bessie must arrive at a farm strictly before Farmer John closes it to actually visit it.



Bessie has $Q$ $(1 \leq Q \leq 2 \cdot 10^5)$ queries. For each query, she gives

you two integers $S$ and $V$. For each query, output whether Bessie can visit at

least $V$ farms if she wakes up at time $S$.



INPUT FORMAT (input arrives from the terminal / stdin):

The first line consists of $N$ and $Q$.



The second line consists of $c_1, c_2, c_3 \dots c_N$ ($1 \leq c_i \leq 10^6$).



The third line consists of $t_1, t_2, t_3 \dots t_N$ ($1 \leq t_i \leq 10^6$).



The next $Q$ lines each consist of two integers $V$ ($1 \leq V \leq N$) and $S$

($1 \leq S \leq 10^6$).



OUTPUT FORMAT (print output to the terminal / stdout):

For each of the $Q$ queries, output YES or NO on a new line.



SAMPLE INPUT:

5 5

3 5 7 9 12

4 2 3 3 8

1 5

1 6

3 3

4 2

5 1

SAMPLE OUTPUT: 

YES

NO

YES

YES

NO



For the first query, Bessie will visit the farms at time $t = [9, 7, 8, 8, 13]$,

so she will only get to visit farm $4$ on time before FJ closes the farm.



For the second query, Bessie will not be able to visit any of the farms on time.



For the third query, Bessie will visit farms $3, 4, 5$ on time.



For the fourth and fifth queries, Bessie will be able to visit all but the first

farm on time.



SCORING:

Inputs 2-4: $N,Q\le 10^3$Inputs 5-9: $c_i, t_i \le 20$Inputs 10-17: No additional constraints.





Problem credits: Chongtian Ma

Pretty difficult! Let's run our simple "zero-shot" agent below to see how it fares. It most likely will not be able to solve this question (unless you are using a more powerful model than what I had available at the time of writing this tutorial (2024/04/20). We will trace the trajectory to LangSmith to review the series of submissions. To reduce the packet size, we will use "hide_inputs" and filter out the test_cases. All this is optional but useful for development.

Note: We expect a GraphRecursionError here from it not being able to answer it correctly in the allocated number of steps.

from langchain_core.tracers.context import tracing_v2_enabled

from langsmith import Client





# We don't need to include all the test cases in our traces.

def _hide_test_cases(inputs):

    copied = inputs.copy()

    # These are tens of MB in size. No need to send them up

    copied["test_cases"] = "..."

    return copied





client = Client(hide_inputs=_hide_test_cases, hide_outputs=_hide_test_cases)

with tracing_v2_enabled(client=client):

    events = graph.stream(input_state)

    for event in events:

        for value in event.values():

            messages = value.get("messages")

            if messages:

                if isinstance(messages, list):

                    messages = value["messages"][-1]

                print(

                    "Assistant:",

                    str(messages.content).replace("\n", "\\n")[:50],

                )


API Reference: tracing_v2_enabled

Assistant: [{'text': '<thinking>\nThe key steps to solve this

Assistant: KeyError('code')\nMake all fixes using the writePy

Assistant: [{'id': 'toolu_01KimhKt8aqQjGZJmrHVnAtE', 'input':

Assistant: Incorrect submission. Please respond with updated 

Assistant: [{'id': 'toolu_01CMZTqAd7BZQ2nSgtk9djRW', 'input':

Assistant: Incorrect submission. Please respond with updated 

Assistant: [{'id': 'toolu_01Kbaq9gX4BnHvps6TMfVGHL', 'input':

Assistant: Incorrect submission. Please respond with updated 

Assistant: [{'id': 'toolu_01MiSnpiGK5Yy4Cpp6GGbjmT', 'input':

Assistant: Incorrect submission. Please respond with updated 

Assistant: [{'id': 'toolu_01GWuvJezXLMVurUBG84odDP', 'input':

Assistant: Incorrect submission. Please respond with updated 

Assistant: [{'id': 'toolu_01W8DGmhcpFVctySmx58scf9', 'input':

Assistant: Incorrect submission. Please respond with updated 

Assistant: [{'id': 'toolu_018bhYtCKDK6S4MHiAxUZCrb', 'input':

Assistant: KeyError('code')\nMake all fixes using the writePy

Assistant: [{'id': 'toolu_01LCwaCjX9uZBV3jt9eAkmAa', 'input':

Assistant: Incorrect submission. Please respond with updated 

Assistant: [{'id': 'toolu_01WqJvdE2WDeTZXoKp2V7PWb', 'input':

Assistant: Incorrect submission. Please respond with updated 

Assistant: [{'id': 'toolu_01DGevkunt9zWx7SVDCHdBuv', 'input':

Assistant: Incorrect submission. Please respond with updated 

Assistant: [{'id': 'toolu_013comYKVxNSzTM4ZbH3L3FP', 'input':

Assistant: Incorrect submission. Please respond with updated

---------------------------------------------------------------------------

``````output

GraphRecursionError                       Traceback (most recent call last)

``````output

Cell In[25], line 17

     15 with tracing_v2_enabled(client=client):

     16     events = graph.stream(input_state)

---> 17     for event in events:

     18         for value in event.values():

     19             messages = value.get("messages")

``````output

File ~/.pyenv/versions/3.11.2/lib/python3.11/site-packages/langgraph/pregel/__init__.py:645, in Pregel.stream(self, input, config, stream_mode, output_keys, input_keys, interrupt_before_nodes, interrupt_after_nodes, debug)

    643         break

    644 elif step == config["recursion_limit"]:

--> 645     raise GraphRecursionError(

    646         f"Recursion limit of {config['recursion_limit']} reached"

    647         "without hitting a stop condition. You can increase the "

    648         "limit by setting the `recursion_limit` config key."

    649     )

    651 # before execution, check if we should interrupt

    652 if _should_interrupt(

    653     checkpoint,

    654     interrupt_before_nodes,

    655     self.stream_channels_list,

    656     next_tasks,

    657 ):

``````output

GraphRecursionError: Recursion limit of 25 reachedwithout hitting a stop condition. You can increase the limit by setting the `recursion_limit` config key.


It wasn't able to solve it in time but that's OK! If it were easy, this paper would be a lot shorter :)

You can view the agent's full LangSmith trace at the provided link.

In the next section we will add an improvement the paper terms "episodic memory", which in this case is really few-shot retrieval.

Part 2: Few-shot Retrieval¶

Even with reflective tool calling, our baseline agent from part 1 struggled with this difficult task. One way to "teach" an LLM how to better perform a task is through demonstrations, also known as "few-shot examples."

What the authors of the USACO paper call "episodic memory" is really just few-shot prompting over similar examples.

Each examples in this case is a different problems + solution within the dataset. The term "episodic memory" makes sense if you pretend your agent has already "solved" these problems and is recalling its solutions to them.

This section adds the "Episodic Memory" components from "Part 2" in the diagram below.

Note that this memory step is performed one time, before the logic of our zero-shot loop from part 1. The steps are as follows:

Prompt the LLM to generate a candidate solution.
Use the text of the candidate solution to retrieve the N most similar (problem, solution) pairs.
Format this result in the Zero-shot agent's prompt.

Below, let's implement our episodic memory as a retriever. We will follow the paper's retriever selection and use BM25.

%%capture --no-stderr

%pip install --upgrade --quiet  rank_bm25

State¶

The state is mostly recycled from part 1. Add additional "candidate" and "examples" fields to store the information for the memory steps.

from typing import Annotated



from typing_extensions import TypedDict



from langgraph.graph.message import AnyMessage, add_messages





class TestCase(TypedDict):

    inputs: str

    outputs: str





class State(TypedDict):

    # NEW! Candidate for retrieval + formatted fetched examples as "memory"

    candidate: AIMessage

    examples: str

    # Repeated from Part 1

    messages: Annotated[list[AnyMessage], add_messages]

    test_cases: list[TestCase]

    runtime_limit: int

    status: str


API Reference: add_messages

Nodes 1 and 3: Draft & Solver¶

Let's create our "agent". We will modify the Solver from Part 1 to reuse it for for the agent node and for the candidate program generation node ("draft").

from langchain import hub

from langchain_anthropic import ChatAnthropic





class Solver:

    def __init__(self, llm: BaseChatModel, prompt: ChatPromptTemplate):

        self.runnable = prompt | llm.bind_tools([writePython])



    def __call__(self, state: State) -> dict:

        # Our agent only can see the "messages" and will ignore the test info

        inputs = {"messages": state["messages"]}

        has_examples = bool(state.get("examples"))

        output_key = "candidate"  # Used in the draft node

        if has_examples:

            output_key = "messages"

            # Used in the solve node

            inputs["examples"] = state["examples"]

        response = self.runnable.invoke(inputs)

        if not response.content:

            return {

                output_key: AIMessage(

                    content="I'll need to think about this step by step."

                )

            }

        return {output_key: response}





prompt = hub.pull("wfh/usaco-draft-solver")

llm = ChatAnthropic(model="claude-3-opus-20240229")



draft_solver = Solver(llm, prompt.partial(examples=""))

solver = Solver(llm, prompt)


API Reference: ChatAnthropic

Node 2: Retrieve¶

The retrieve node takes a candidate solution (made by the 'solver' node), uses this to search for similar examples, then formats those in the message.

# We will test our agent on index 0 (the same as above).

# Later, we will test on index 2 (the first 'silver difficulty' question)

test_indices = [0, 2]

train_ds = [row for i, row in enumerate(ds) if i not in test_indices]

test_ds = [row for i, row in enumerate(ds) if i in test_indices]

from langchain_community.retrievers import BM25Retriever





def format_example(row):

    question = row["description"]

    answer = row["solution"]

    return f"""<problem>

{question}

</problem>

<solution>

{answer}

</solution>"""





# Skip our 'test examples' to avoid cheating

# This is "simulating" having seen other in-context examples

retriever = BM25Retriever.from_texts([format_example(row) for row in train_ds])


API Reference: BM25Retriever

Now define the node. Any node can optionally accept a second config positional argument. This contains configurable params you can adjust when invoking the graph. For instance, we can adjust the top k examples to retrieve for our agent.

from langchain_core.runnables import RunnableConfig





def retrieve_examples(state: State, config: RunnableConfig):

    top_k = config["configurable"].get("k") or 2

    ai_message: AIMessage = state["candidate"]

    if not ai_message.tool_calls:

        # We err here. To make more robust, you could loop back

        raise ValueError("Draft agent did not produce a valid code block")

    code = ai_message.tool_calls[0]["args"]["code"]

    examples_str = "\n".join(

        [doc.page_content for doc in retriever.invoke(code)[:top_k]]

    )

    examples_str = f"""

You previously solved the following problems in this competition:

<Examples>

{examples_str}

<Examples>

Approach this new question with similar sophistication."""

    return {"examples": examples_str}


API Reference: RunnableConfig

Graph¶

Now let's put it all together. The graph is slightly more complicated than in part 1, since we have to add the initial "draft" and "retrieve" nodes to our agent loop.

from langgraph.checkpoint.memory import MemorySaver

from langgraph.graph import END, StateGraph, START



builder = StateGraph(State)

builder.add_node("draft", draft_solver)

builder.add_edge(START, "draft")

builder.add_node("retrieve", retrieve_examples)

builder.add_node("solve", solver)

builder.add_node("evaluate", evaluate)

# Add connectivity

builder.add_edge("draft", "retrieve")

builder.add_edge("retrieve", "solve")

builder.add_edge("solve", "evaluate")





def control_edge(state: State):

    if state.get("status") == "success":

        return END

    return "solve"





builder.add_conditional_edges("evaluate", control_edge, {END: END, "solve": "solve"})





checkpointer = MemorySaver()

graph = builder.compile(checkpointer=checkpointer)


API Reference: MemorySaver | END | StateGraph | START

from IPython.display import Image, display



try:

    display(Image(graph.get_graph().draw_mermaid_png()))

except Exception:

    # This requires some extra dependencies and is optional

    pass


Let's try again on this problem:

config = {"configurable": {"thread_id": "question-recall", "k": 3}}

with tracing_v2_enabled(client=client):

    events = graph.stream(input_state, config)

    for event in events:

        for value in event.values():

            messages = value.get("messages")

            if messages:

                if isinstance(messages, list):

                    messages = value["messages"][-1]

                print(

                    "Assistant:",

                    str(messages.content).replace("\n", "\\n")[:50],

                )

            elif value.get("examples"):

                print("Retrieved examples:\n\n", value["examples"][:100] + "...")

            elif value.get("candidate"):

                print(str(value["candidate"].content)[:200])

[{'text': "<thinking>\nThis problem essentially asks to find the number of farms Bessie can visit before they close at each query. The key insights are:\n\n1. Bessie's arrival time at each farm is S +

Retrieved examples:





You previously solved the following problems in this competition:

<Examples>

<problem>



Farmer John...

Assistant: [{'text': "<thinking>\nThe key information given i

No recursion error! You can view the full LangSmith trace of the graph's execution at the provided link to confirm the results. You can also check the graph state to confirm that it passed all test cases successfully:

checkpoint = graph.get_state(config)

checkpoint.values["status"]

'success'


Congrats! You added "episodic memory" to your agent to fetch few-shot examples and solve this bronze level programming olympiad question!

Our agent is still limited, however. Let's test it out on a more challenging 🪙🏆silver✨ level question:

silver_row = test_ds[1]

silver_row["problem_level"]

'silver'


silver_input = {

    "messages": [("user", silver_row["description"])],

    "test_cases": silver_row["test_cases"],

    "runtime_limit": silver_row["runtime_limit"],

    "status": "in_progress",

}





config = {"configurable": {"thread_id": "silver-question-1", "k": 2}}

with tracing_v2_enabled(client=client):

    events = graph.stream(silver_input, config)

    for event in events:

        for value in event.values():

            messages = value.get("messages")

            if messages:

                if isinstance(messages, list):

                    messages = value["messages"][-1]

                print(

                    "Assistant:",

                    str(messages.content).replace("\n", "\\n")[:50],

                )

            elif value.get("examples"):

                print("Retrieved examples:\n\n", value["examples"][:100] + "...")

            elif value.get("candidate"):

                print(str(value["candidate"].content)[:200])

[{'text': "<thinking>\nThe relevant tool for this problem is writePython. It requires the following parameters:\n- reasoning: To solve this problem, we need to simulate the cruise by following the seq

Retrieved examples:





You previously solved the following problems in this competition:

<Examples>

<problem>



Farmer John...

Assistant: [{'text': "<thinking>\nTo solve this problem, we n

Assistant: Incorrect submission. Please respond with updated 

Assistant: [{'text': "<thinking>\nAfter reviewing the failed 

Assistant: Incorrect submission. Please respond with updated 

Assistant: [{'text': "<thinking>\nAfter reviewing the latest 

Assistant: Incorrect submission. Please respond with updated 

Assistant: [{'text': "<thinking>\nOops, looks like I made a s

Assistant: Incorrect submission. Please respond with updated 

Assistant: [{'text': "<thinking>\nHmm, some of the test cases

Assistant: Incorrect submission. Please respond with updated 

Assistant: [{'text': '<thinking>\nOops, looks like I accident

Assistant: Incorrect submission. Please respond with updated 

Assistant: [{'text': "<thinking>\nLooks like the code is now 

Assistant: Incorrect submission. Please respond with updated 

Assistant: [{'text': '<thinking>\nOops, looks like I accident

Assistant: Incorrect submission. Please respond with updated 

Assistant: [{'text': "<thinking>\nHmm, the optimization to si

Assistant: Incorrect submission. Please respond with updated 

Assistant: [{'text': "<thinking>\nOops, I did it again - acci

Assistant: Incorrect submission. Please respond with updated 

Assistant: [{'text': "<thinking>\nHmm, the latest code is sti

Assistant: Incorrect submission. Please respond with updated

---------------------------------------------------------------------------

``````output

GraphRecursionError                       Traceback (most recent call last)

``````output

Cell In[37], line 12

     10 with tracing_v2_enabled(client=client):

     11     events = graph.stream(silver_input, config)

---> 12     for event in events:

     13         for value in event.values():

     14             messages = value.get("messages")

``````output

File ~/.pyenv/versions/3.11.2/lib/python3.11/site-packages/langgraph/pregel/__init__.py:645, in Pregel.stream(self, input, config, stream_mode, output_keys, input_keys, interrupt_before_nodes, interrupt_after_nodes, debug)

    643         break

    644 elif step == config["recursion_limit"]:

--> 645     raise GraphRecursionError(

    646         f"Recursion limit of {config['recursion_limit']} reached"

    647         "without hitting a stop condition. You can increase the "

    648         "limit by setting the `recursion_limit` config key."

    649     )

    651 # before execution, check if we should interrupt

    652 if _should_interrupt(

    653     checkpoint,

    654     interrupt_before_nodes,

    655     self.stream_channels_list,

    656     next_tasks,

    657 ):

``````output

GraphRecursionError: Recursion limit of 25 reachedwithout hitting a stop condition. You can increase the limit by setting the `recursion_limit` config key.


Still too hard! AGI not achieved yet. To investigate our agent's trajectory in detail, check out the full LangSmith trace.

Our agent isn't good enough to be autonomous. The great thing about LangGraph is you don't have to decide between "autonomous agent" and "simple DAG": you can inject control and user-interfaces wherever it can usefully benefit your application.

Part 3: Human-in-the-loop¶

Our retrieval-enhanced agent was able to solve the bronze-level question but still failed for those with the more challenging silver difficulty.

Recall that the paper presented 3 complementary techniques that improved performance:

Reflection: explicitly prompting the LLM to "reflect" on its mistakes can help it
Few-shot prompting: retrieving relevant, high-quality examples as "memory"
Human-in-the-loop collaboration: without giving the correct answer, the human is allowed to help the agent reflect on its approach and point it in a better direction.

In this section, we will add the "human" node (marked as "part 3" in the diagram below), completing our agent graph:

From an ML perspective, this is a bit of a clever hans, but from the application designer's perspective, where the primary goal is to achieve a higher combined success rate, letting the human interject with thoughts and insights is only natural.

In either case, adding a human check to a LangGraph instance requires no extra lines of code. Let's do so by instructing the graph to interrupt_after the "evaluate" node to give the user a chance to modify the trajectory.

Start assembling your graph below. The following section is identical to our application in part 2:

# This is all the same as before

from langgraph.checkpoint.memory import MemorySaver

from langgraph.graph import END, StateGraph, START



builder = StateGraph(State)

prompt = hub.pull("wfh/usaco-draft-solver")

llm = ChatAnthropic(model="claude-3-opus-20240229", max_tokens_to_sample=4000)



draft_solver = Solver(llm, prompt.partial(examples=""))

builder.add_node("draft", draft_solver)

builder.add_edge(START, "draft")

builder.add_node("retrieve", retrieve_examples)

solver = Solver(llm, prompt)

builder.add_node("solve", solver)

builder.add_node("evaluate", evaluate)

builder.add_edge("draft", "retrieve")

builder.add_edge("retrieve", "solve")

builder.add_edge("solve", "evaluate")





def control_edge(state: State):

    if state.get("status") == "success":

        return END

    return "solve"





builder.add_conditional_edges("evaluate", control_edge, {END: END, "solve": "solve"})

checkpointer = MemorySaver()


API Reference: MemorySaver | END | StateGraph | START

Now finish by compiling the graph. Setinterrupt_after=["evaluate"] to instruct the agent to wait for human input before continuing execution.

graph = builder.compile(

    checkpointer=checkpointer,

    # New: this tells the graph to break any time it goes to the "human" node

    interrupt_after=["evaluate"],

)

from IPython.display import Image, display



try:

    display(Image(graph.get_graph().draw_mermaid_png()))

except Exception:

    # This requires some extra dependencies and is optional

    pass


As you can see in the graph above, the structure is the same as Part 2, except that we've inserted a "human" breakpoint between the "evaluate" and "solve" nodes.

Let's try this question again!

config = {"configurable": {"thread_id": "silver-hl-1", "k": 2}}

with tracing_v2_enabled(client=client):

    events = graph.stream(silver_input, config)

    for event in events:

        for value in event.values():

            messages = value.get("messages")

            if messages:

                if isinstance(messages, list):

                    messages = value["messages"][-1]

                print(

                    "Assistant:",

                    str(messages.content).replace("\n", "\\n")[:50],

                )

            elif value.get("examples"):

                print("Retrieved examples:\n\n", value["examples"][:100] + "...")

            elif value.get("candidate"):

                print(str(value["candidate"].content)[:200])

[{'text': "<thinking>\nTo solve this problem, we need to:\n1. Read in the input data - number of ports N, length of direction sequence M, number of repetitions K, the port connections, and the directi

Retrieved examples:





You previously solved the following problems in this competition:

<Examples>

<problem>

Farmer John ...

Assistant: [{'text': '<thinking>\nTo determine where Bessie e

Assistant: Incorrect submission. Please respond with updated

⏰Time to weigh in⏰: our model failed in its first attempt, so we have the opportunity to give it some advice.

Recall the original question:

snapshot = graph.get_state(config)

print(snapshot.values["messages"][0].content)

Problem 3: Luxury River Cruise [Josh Alman and Nathan Pinsker, 2013]



Farmer John is taking Bessie and the cows on a cruise! They are sailing on a 

network of rivers with N ports (1 <= N <= 1,000) labeled 1..N, and Bessie 

starts at port 1. Each port has exactly two rivers leading out of it which 

lead directly to other ports, and rivers can only be sailed one way.



At each port, the tour guides choose either the "left" river or the "right" 

river to sail down next, but they keep repeating the same choices over and 

over. More specifically, the tour guides have chosen a short sequence of M 

directions (1 <= M <= 500), each either "left" or "right", and have

repeated it K times (1 <= K <= 1,000,000,000). Bessie thinks she is going

in circles -- help her figure out where she ends up!



PROBLEM NAME: cruise



INPUT FORMAT:



* Line 1: Three space-separated integers N, M, and K.



* Lines 2..N+1: Line i+1 has two space-separated integers,

        representing the number of the ports that port i's left and

        right rivers lead to, respectively.



* Line N+2: M space-separated characters, either 'L' or 'R'. 'L'

        represents a choice of  'left' and 'R' represents a choice of

        'right'.



SAMPLE INPUT:



4 3 3

2 4

3 1

4 2

1 3

L L R



INPUT DETAILS:



The port numbers are arranged clockwise in a circle, with 'L' being a 

clockwise rotation and 'R' being a counterclockwise rotation. The sequence 

taken is LLRLLRLLR.



OUTPUT FORMAT:



* Line 1: A single integer giving the number of the port where

        Bessie's cruise ends.



SAMPLE OUTPUT:



4



OUTPUT DETAILS:



After the first iteration of the sequence of directions, Bessie is at port

2 (1 -> 2 -> 3 -> 2); after the second, she is at port 3 (2 -> 3 -> 4 ->

3), and at the end she is at port 4 (3 -> 4 -> 1 -> 4).

And then review the agent's current submission:

snapshot = graph.get_state(config)

print(snapshot.values["messages"][-2].content[0]["text"])

print("\n\nCode:\n\n")

print(snapshot.values["messages"][-2].tool_calls[0]["args"]["code"])

<thinking>

To determine where Bessie ends up, we need to:

1. Simulate the cruise by following the sequence of left/right directions

2. Repeat this sequence K times to find the final destination port



The problem provides:

- The number of ports N

- The connections between ports (left and right rivers for each port)

- The sequence of M directions (L or R) to follow

- The number of times K to repeat the sequence



With this information, we have everything needed to simulate the cruise and find the ending port. The key steps will be:

1. Read in the input data to initialize the river connections and direction sequence 

2. Iterate K times:

   - For each direction in the M-length sequence:

     - Move to the next port based on the current port and direction 

3. Output the final port number after K iterations



The solution will require loops to repeat the sequence K times and follow the M directions. Since K can be up to 1 billion, simulating all K iterations directly would be too slow. Instead, we can find a pattern in how the port changes after each M-length sequence, and then "fast-forward" by calculating which port we reach after K repetitions of the pattern.

</thinking>





Code:





N, M, K = map(int, input().split())



ports = []

for _ in range(N):

  left, right = map(int, input().split())

  ports.append((left, right))



directions = input().split()



cur = 1

pattern = []

seen = set() 

steps = 0



while cur not in seen:

  seen.add(cur)

  for d in directions:

    steps += 1

    if d == 'L': 

      cur = ports[cur-1][0]

    else:

      cur = ports[cur-1][1]

  pattern.append((cur, steps))



K %= steps

for port, step in pattern:

  if step > K:

    cur = port

    break

  K -= step



print(cur)


print(snapshot.values["messages"][-1].content[:200])

Incorrect submission. Please respond with updated code.

Pass rate: 4/10

Results:

<test id=0>

wrong answer. Expected '4

', got '3

'

</test>

<test id=1>

wrong answer. Expected '50

', got '2

'

</test>

<t

The agent failed. It's on the right track but clearly doesn't handle all the edge cases.

The agent needs to remember that simulation should include the cycle + whatever steps led up to the example. It could use the "tortoise and hare" algo for cycle detection, use the simulated path and break if and when a repeat is detected, and then

Let's let the agent know this by updating the graph state.

updated_config = graph.update_state(

    config,

    values={

        "messages": [

            (

                "user",

                """Consider breaking down the algorithm into separate parts: reading inputs, detecting cycles using the tortoise and hare algorithm, and determining Bessie's final position by skipping ahead K steps.



Read the inputs into three arrays:

- Two arrays L and R for the ports (adjust for 0-based indexing)

- A third array S for the direction sequence



Optimize by multiplying K by M before the main loop to convert the number of repetitions into the total number of steps.



Use the tortoise and hare algorithm to detect the cycle:

- Define a helper function get_next(v) that returns the next position and direction index

- Initialize two pointers s0 and s1 to (0, 0)

- In each iteration:

  - Move s0 by 1 step and s1 by 2 steps using get_next()

  - If s0 equals s1, decrement K by 1 and break out of the loop

  - Otherwise, decrement K by 1

- After the loop, if K is not 0, there is a cycle



To find the cycle length:

- Initialize a counter variable rho to 1

- Move s0 by 1 step using get_next()

- Enter a loop:

  - Move s0 by 1 step using get_next()

  - Increment rho

  - If s0 equals s1, break out of the loop



Skip ahead by reducing K modulo rho.



Simulate the remaining steps:

- While K > 0, move s0 to the next position using get_next() and decrement K



Print the final position (converted to 1-based indexing).



Pay close attention to the initialization and movement of pointers during cycle detection and length calculation. Ensure that the logic is correct and handles all cases accurately.""",

            )

        ]

    },

)


Now the graph's state contains our new message.

graph.get_state(config).values["messages"][-1]

HumanMessage(content="Consider breaking down the algorithm into separate parts: reading inputs, detecting cycles using the tortoise and hare algorithm, and determining Bessie's final position by skipping ahead K steps.\n\nRead the inputs into three arrays:\n- Two arrays L and R for the ports (adjust for 0-based indexing)\n- A third array S for the direction sequence\n\nOptimize by multiplying K by M before the main loop to convert the number of repetitions into the total number of steps.\n\nUse the tortoise and hare algorithm to detect the cycle:\n- Define a helper function get_next(v) that returns the next position and direction index\n- Initialize two pointers s0 and s1 to (0, 0)\n- In each iteration:\n  - Move s0 by 1 step and s1 by 2 steps using get_next()\n  - If s0 equals s1, decrement K by 1 and break out of the loop\n  - Otherwise, decrement K by 1\n- After the loop, if K is not 0, there is a cycle\n\nTo find the cycle length:\n- Initialize a counter variable rho to 1\n- Move s0 by 1 step using get_next()\n- Enter a loop:\n  - Move s0 by 1 step using get_next()\n  - Increment rho\n  - If s0 equals s1, break out of the loop\n\nSkip ahead by reducing K modulo rho.\n\nSimulate the remaining steps:\n- While K > 0, move s0 to the next position using get_next() and decrement K\n\nPrint the final position (converted to 1-based indexing).\n\nPay close attention to the initialization and movement of pointers during cycle detection and length calculation. Ensure that the logic is correct and handles all cases accurately.", id='98888982-a469-4c5a-ab65-743d2f2608dc')


Let's let the agent try again. Call stream with None to just use the inputs loaded from the memory. We will skip our human review for the next few attempats to see if it can correct itself.

num_trials = 1

with tracing_v2_enabled(client=client):

    for _ in range(num_trials):

        events = graph.stream(None, updated_config)

        for event in events:

            for value in event.values():

                messages = value.get("messages")

                if messages:

                    if isinstance(messages, list):

                        messages = value["messages"][-1]

                    print(

                        "Assistant:",

                        str(messages.content).replace("\n", "\\n")[:50],

                    )

                elif value.get("examples"):

                    print("Retrieved examples:\n\n", value["examples"][:100] + "...")

                elif value.get("candidate"):

                    print(str(value["candidate"].content)[:200])

        if graph.get_state(config).values["status"] == "success":

            break

        print("Continuing...")

Assistant: [{'text': '<thinking>\nThank you for the detailed 

Assistant: Incorrect submission. Please respond with updated 

Continuing...


most_recent_state = list(graph.get_state_history(config))[0]


OK so the agent tried again. Check out the LangSmith trace from this step to see its update.

snapshot = graph.get_state(most_recent_state.config)

ai_message = snapshot.values["messages"][-2]

if ai_message.content:

    print(ai_message.content)

print("\n\nCode:\n\n")

print(ai_message.tool_calls[0]["args"]["code"] if ai_message.tool_calls else "N/A")

[{'text': '<thinking>\nThank you for the detailed algorithm breakdown! Let me go through each step to make sure I understand and have the necessary information to implement the solution.\n\nReading inputs:\n- Read N, M, K and store in separate variables\n- Create arrays L and R to store the left and right port connections (adjust for 0-based indexing)\n- Create array S to store the M-length direction sequence \n- Multiply K by M upfront to get the total number of steps\n\nDetecting cycles with tortoise and hare:\n- Define get_next(v) to return the next position and direction index\n  - It will use the current position and direction to look up the next port in L/R\n- Initialize two pointers s0 and s1 to (0, 0) \n- Loop until s0 equals s1 or all K steps are taken:\n  - Move s0 by 1 step and s1 by 2 steps using get_next()\n  - Decrement K\n- After the loop, check if K is 0 to determine if a cycle was found\n\nFinding cycle length:\n- If a cycle was found, initialize rho to 1\n- Move s0 by 1 step \n- Loop until s0 equals s1 again:\n  - Move s0 by 1 step and increment rho\n- rho will equal the cycle length\n\nSkipping ahead:\n- Reduce K by taking it modulo rho\n\nSimulating remaining steps:\n- While K is greater than 0:\n  - Move s0 using get_next()\n  - Decrement K\n- s0 will hold the final position\n\nPrinting result:\n- Add 1 to the final position to convert back to 1-based indexing before printing\n\nThe key aspects are:\n- Handling the input format and 0-based indexing \n- Defining get_next() to handle moving to the next port based on direction\n- Correctly implementing the tortoise and hare cycle detection\n- Finding the cycle length after detection\n- Skipping ahead with modulo and simulating any remaining steps\n- Adjusting the output back to 1-based indexing\n\nI believe I have all the necessary pieces to implement this solution now. Let me code it up using the writePython tool.\n</thinking>', 'type': 'text'}, {'id': 'toolu_01EDrYeHJU7GxApRb1QfMA1b', 'input': {'reasoning': "Here's the problem-solving approach:\n\n1. Read in the input data:\n   - N ports, M-length direction sequence, K repetitions\n   - L and R arrays for left/right port connections\n   - S array for direction sequence\n   - Multiply K by M to get total steps\n\n2. Define get_next(v) helper function:\n   - Takes current position and direction index\n   - Returns next position and incremented direction index\n   - Looks up next port in L/R arrays based on current direction\n\n3. Detect cycle using tortoise and hare algorithm:\n   - Initialize s0 and s1 pointers to (0, 0)\n   - Loop until match or all steps taken:\n     - Move s0 by 1 step, s1 by 2 steps\n     - Decrement K\n   - Check if K is 0 after loop\n\n4. If cycle found, find cycle length:\n   - Initialize rho to 1\n   - Move s0 by 1 step\n   - Loop until s0 equals s1 again:\n     - Move s0 and increment rho\n   - rho is the cycle length\n\n5. Skip ahead by K % rho steps\n\n6. Simulate remaining steps:\n   - While K > 0:\n     - Move s0 with get_next()\n     - Decrement K\n   \n7. Print final position (+1 for 1-based indexing)\n\nKey points:\n- Multiplying K*M avoids nested loop\n- get_next() handles port transitions \n- Tortoise and hare finds cycles\n- Modulo skips ahead in cycle\n- Adjust 0-based indexing for input/output", 'pseudocode': "1. Read input:\n   N, M, K = read_ints()\n   L = [0] * N\n   R = [0] * N\n   for i in 0..N-1:\n     L[i], R[i] = read_ints()\n   S = read_direction_sequence()\n   K *= M\n\n2. Define get_next(v):\n   def get_next(pos, dir_idx):\n     if S[dir_idx] == 'L':\n       next_pos = L[pos]\n     else:\n       next_pos = R[pos]\n     next_dir_idx = (dir_idx + 1) % M\n     return (next_pos, next_dir_idx)\n\n3. Find cycle:\n   s0 = (0, 0)\n   s1 = (0, 0)  \n   while K:\n     s0 = get_next(s0[0], s0[1])\n     s1 = get_next(s1[0], get_next(s1[0], s1[1])[1])\n     K -= 1\n     if s0 == s1: break\n   if K != 0: no cycle, print s0[0] + 1\n\n4. Find cycle length:\n   rho = 1\n   s0 = get_next(s0[0], s0[1])\n   while s0 != s1:\n     s0 = get_next(s0[0], s0[1]) \n     rho += 1\n\n5. Skip steps:\n   K %= rho\n\n6. Remaining steps:  \n   while K:\n     s0 = get_next(s0[0], s0[1])\n     K -= 1\n     \n7. Print result:\n   print(s0[0] + 1)", 'code': "def read_ints():\n  return map(int, input().split())\n\nN, M, K = read_ints()\n\nL = [0] * N\nR = [0] * N\nfor i in range(N):\n  L[i], R[i] = read_ints()\n  L[i] -= 1\n  R[i] -= 1\n\nS = input().split()\n\nK *= M\n\ndef get_next(pos, dir_idx):\n  if S[dir_idx] == 'L':\n    next_pos = L[pos] \n  else:\n    next_pos = R[pos]\n  next_dir_idx = (dir_idx + 1) % M\n  return (next_pos, next_dir_idx)\n\ns0 = (0, 0)  \ns1 = (0, 0)\n\nwhile K:\n  if s0 == s1: break\n  \n  s0 = get_next(s0[0], s0[1])\n  s1 = get_next(s1[0], get_next(s1[0], s1[1])[1])\n  \n  K -= 1\n  \nif K:\n  rho = 1\n  s0 = get_next(s0[0], s0[1])\n  while s0 != s1:\n    s0 = get_next(s0[0], s0[1])\n    rho += 1\n  \n  K %= rho\n  \nwhile K:  \n  s0 = get_next(s0[0], s0[1])\n  K -= 1\n  \nprint(s0[0] + 1)"}, 'name': 'writePython', 'type': 'tool_use'}]





Code:





def read_ints():

  return map(int, input().split())



N, M, K = read_ints()



L = [0] * N

R = [0] * N

for i in range(N):

  L[i], R[i] = read_ints()

  L[i] -= 1

  R[i] -= 1



S = input().split()



K *= M



def get_next(pos, dir_idx):

  if S[dir_idx] == 'L':

    next_pos = L[pos] 

  else:

    next_pos = R[pos]

  next_dir_idx = (dir_idx + 1) % M

  return (next_pos, next_dir_idx)



s0 = (0, 0)  

s1 = (0, 0)



while K:

  if s0 == s1: break



  s0 = get_next(s0[0], s0[1])

  s1 = get_next(s1[0], get_next(s1[0], s1[1])[1])



  K -= 1



if K:

  rho = 1

  s0 = get_next(s0[0], s0[1])

  while s0 != s1:

    s0 = get_next(s0[0], s0[1])

    rho += 1



  K %= rho



while K:  

  s0 = get_next(s0[0], s0[1])

  K -= 1



print(s0[0] + 1)


print(snapshot.values["messages"][-1].content[:200])

Incorrect submission. Please respond with updated code.

Pass rate: 3/10

Results:

<test id=0>

passed

</test>

<test id=1>

timed out

</test>

<test id=2>

timed out

</test>

<test id=3>

timed out

</test>

<t

Still getting most test cases wrong.

Let's provide more feedback.

updated_config = graph.update_state(

    updated_config,

    values={

        "messages": [

            (

                "user",

                """That's better, but you're still getting some errors. Let's double check some things:



1. When calculating the cycle length, make sure the initialization and movement of the pointers is correct. Double-check the logic there and see if you can spot any discrepancies.

2. Check the condition for whether there's a cycle after the main loop to ensure it covers all cases, like if  K becomes 0 in the last iteration.



Think step by step through youur implementation and update using the writePython tool.""",

            )

        ]

    },

)


Now that we've provided this feedback, let's give the agent a few attempts at solving it before we weigh in again.

num_trials = 2

with tracing_v2_enabled(client=client):

    for _ in range(num_trials):

        events = graph.stream(None, updated_config)

        for event in events:

            for value in event.values():

                messages = value.get("messages")

                if messages:

                    if isinstance(messages, list):

                        messages = value["messages"][-1]

                    print(

                        "Assistant:",

                        str(messages.content).replace("\n", "\\n")[:50],

                    )

                elif value.get("examples"):

                    print("Retrieved examples:\n\n", value["examples"][:100] + "...")

                elif value.get("candidate"):

                    print(str(value["candidate"].content)[:200])

        if graph.get_state(config).values["status"] == "success":

            break

        print("Continuing...")

Assistant: [{'text': "<thinking>\nThe algorithm looks mostly

You can review a LangSmith trace (link) of the agent's response to your feedback at the provided link.

snapshot = graph.get_state(config)

print(snapshot.values["status"])

success

Success! - the LLM really wouldn't have been able to come to the correct answer without detailed human involvement.

Conclusion¶

Congrats on making it to the end! In this tutorial, you implemented an agent in LangGraph capable of solving challenging programming problems. You did so by leveraging a few common techniques to improve performance, including:

Reflection: while we didn't implement an explicit reflection step, our prompt and tool invocation was designed to encourage critique of previous outputs. You added this in Part 1.
Retrieval: the "episodic memory" of the agent retrieves high-quality few-shot examples from our corpora of programming problems to help solve the bronze level question. In Part 2, you implemented a retrieval memory as an initial step.
Human-in-the-loop: LLM-powered agents are still too weak to answer all these questions autonomously, but at times, they can get most of the way there and land on the right answer with human feedback. In Part 3, you used interrupt_after on the evaluate node and then included your feedback by using update_state on the graph.

LLMs are not capable of solving all these problems autonomously, but through better prompting and clever engineering, you can create a system that is able to more reliably arrive at the proper solution.

Comments
