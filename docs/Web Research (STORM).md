# Web Research with STORM

## Overview
STORM is a research assistant framework that generates Wikipedia-style articles on user-provided topics by implementing outline-driven RAG with multi-perspective conversational simulation to improve comprehensiveness and information density.

## Key Concepts
- Creates outlines via similar topic search to improve knowledge coverage
- Employs multi-perspective, grounded search conversation simulation 
- Uses subject matter expert role-playing to increase reference count
- Follows a sequential workflow: outline → survey → perspectives → interviews → refinement → writing

## Prerequisites
```python
# Install required packages
!pip install -U langchain_community langchain_openai langchain_fireworks langgraph wikipedia duckduckgo-search tavily-python

# Optional for visualization
# !brew install graphviz
# !CFLAGS="-I $(brew --prefix graphviz)/include" LDFLAGS="-L $(brew --prefix graphviz)/lib" pip install -U pygraphviz

# Set environment variables
import os
import getpass

def _set_env(var: str):
    if os.environ.get(var):
        return
    os.environ[var] = getpass.getpass(var + ":")

_set_env("OPENAI_API_KEY")
_set_env("TAVILY_API_KEY")

# LLM Setup
from langchain_openai import ChatOpenAI

fast_llm = ChatOpenAI(model="gpt-4o-mini")
long_context_llm = ChatOpenAI(model="gpt-4o")
```

## Implementation

### 1. Generate Initial Outline
```python
from typing import List, Optional
from langchain_core.prompts import ChatPromptTemplate
from pydantic import BaseModel, Field

direct_gen_outline_prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            "You are a Wikipedia writer. Write an outline for a Wikipedia page about a user-provided topic. Be comprehensive and specific.",
        ),
        ("user", "{topic}"),
    ]
)

class Subsection(BaseModel):
    subsection_title: str = Field(..., title="Title of the subsection")
    description: str = Field(..., title="Content of the subsection")

    @property
    def as_str(self) -> str:
        return f"### {self.subsection_title}\n\n{self.description}".strip()

class Section(BaseModel):
    section_title: str = Field(..., title="Title of the section")
    description: str = Field(..., title="Content of the section")
    subsections: Optional[List[Subsection]] = Field(
        default=None,
        title="Titles and descriptions for each subsection of the Wikipedia page.",
    )

    @property
    def as_str(self) -> str:
        subsections = "\n\n".join(
            f"### {subsection.subsection_title}\n\n{subsection.description}"
            for subsection in self.subsections or []
        )
        return f"## {self.section_title}\n\n{self.description}\n\n{subsections}".strip()

class Outline(BaseModel):
    page_title: str = Field(..., title="Title of the Wikipedia page")
    sections: List[Section] = Field(
        default_factory=list,
        title="Titles and descriptions for each section of the Wikipedia page.",
    )

    @property
    def as_str(self) -> str:
        sections = "\n\n".join(section.as_str for section in self.sections)
        return f"# {self.page_title}\n\n{sections}".strip()

generate_outline_direct = direct_gen_outline_prompt | fast_llm.with_structured_output(Outline)

# Generate initial outline
example_topic = "Impact of million-plus token context window language models on RAG"
initial_outline = generate_outline_direct.invoke({"topic": example_topic})
```

### 2. Expand Topics
```python
gen_related_topics_prompt = ChatPromptTemplate.from_template(
    """I'm writing a Wikipedia page for a topic mentioned below. Please identify and recommend some Wikipedia pages on closely related subjects. I'm looking for examples that provide insights into interesting aspects commonly associated with this topic, or examples that help me understand the typical content and structure included in Wikipedia pages for similar topics.

Please list the as many subjects and urls as you can.

Topic of interest: {topic}
"""
)

class RelatedSubjects(BaseModel):
    topics: List[str] = Field(
        description="Comprehensive list of related subjects as background research.",
    )

expand_chain = gen_related_topics_prompt | fast_llm.with_structured_output(RelatedSubjects)
related_subjects = await expand_chain.ainvoke({"topic": example_topic})
```

### 3. Generate Perspectives
```python
class Editor(BaseModel):
    affiliation: str = Field(
        description="Primary affiliation of the editor.",
    )
    name: str = Field(
        description="Name of the editor.", pattern=r"^[a-zA-Z0-9_-]{1,64}$"
    )
    role: str = Field(
        description="Role of the editor in the context of the topic.",
    )
    description: str = Field(
        description="Description of the editor's focus, concerns, and motives.",
    )

    @property
    def persona(self) -> str:
        return f"Name: {self.name}\nRole: {self.role}\nAffiliation: {self.affiliation}\nDescription: {self.description}\n"

class Perspectives(BaseModel):
    editors: List[Editor] = Field(
        description="Comprehensive list of editors with their roles and affiliations.",
    )

gen_perspectives_prompt = ChatPromptTemplate.from_messages([
    (
        "system",
        """You need to select a diverse (and distinct) group of Wikipedia editors who will work together to create a comprehensive article on the topic. Each of them represents a different perspective, role, or affiliation related to this topic.
You can use other Wikipedia pages of related topics for inspiration. For each editor, add a description of what they will focus on.

Wiki page outlines of related topics for inspiration:
{examples}""",
    ),
    ("user", "Create a diverse group of editors for a Wikipedia article on {topic}"),
])

generate_perspectives = gen_perspectives_prompt | fast_llm.with_structured_output(Perspectives)
perspectives = generate_perspectives.invoke(
    {"topic": example_topic, "examples": initial_outline.as_str}
)
```

### 4. Conduct Expert Interviews
```python
from typing import Dict, List, TypedDict
from langchain_core.messages import AIMessage, BaseMessage, HumanMessage
from langgraph.graph import END, StateGraph, START
from langgraph.prebuilt import ToolExecutor
from langgraph.graph.message import add_messages
from langchain.tools import tool
from langchain.schema.runnable import RunnableMap, RunnableLambda
from langgraph.graph.retry import RetryPolicy

class InterviewState(TypedDict):
    editor: Editor
    messages: List[BaseMessage]

max_num_turns = 3

gen_answer = RunnableMap(
    {
        "llm_answer": answer_question_chain,
        "message": lambda x: AIMessage(
            content=x["llm_answer"]["answer"], name="Subject_Matter_Expert"
        ),
    }
) | RunnableLambda(lambda x: {"messages": [x["message"]]})

def route_messages(state: InterviewState, name: str = "Subject_Matter_Expert"):
    messages = state["messages"]
    num_responses = len(
        [m for m in messages if isinstance(m, AIMessage) and m.name == name]
    )
    if num_responses >= max_num_turns:
        return END
    last_question = messages[-2]
    if last_question.content.endswith("Thank you so much for your help!"):
        return END
    return "ask_question"

builder = StateGraph(InterviewState)
builder.add_node("ask_question", generate_question, retry=RetryPolicy(max_attempts=5))
builder.add_node("answer_question", gen_answer, retry=RetryPolicy(max_attempts=5))
builder.add_conditional_edges("answer_question", route_messages)
builder.add_edge("ask_question", "answer_question")
builder.add_edge(START, "ask_question")
interview_graph = builder.compile(checkpointer=False).with_config(
    run_name="Conduct Interviews"
)
```

### 5. Refine Outline
```python
refine_outline_prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            """You are a Wikipedia writer. You have gathered information from experts and search engines. Now, you are refining the outline of the Wikipedia page. 
You need to make sure that the outline is comprehensive and specific. 
Topic you are writing about: {topic} 

Old outline:

{old_outline}""",
        ),
        (
            "user",
            "Refine the outline based on your conversations with subject-matter experts:\n\nConversations:\n\n{conversations}\n\nWrite the refined Wikipedia outline:",
        ),
    ]
)

refine_outline_chain = refine_outline_prompt | long_context_llm.with_structured_output(Outline)
refined_outline = refine_outline_chain.invoke(
    {
        "topic": example_topic,
        "old_outline": initial_outline.as_str,
        "conversations": "\n\n".join(
            f"### {m.name}\n\n{m.content}" for m in final_state["messages"]
        ),
    }
)
```

### 6. Building the Complete Workflow
```python
from langgraph.checkpoint.memory import MemorySaver

builder_of_storm = StateGraph(ResearchState)

nodes = [
    ("init_research", initialize_research),
    ("conduct_interviews", conduct_interviews),
    ("refine_outline", refine_outline),
    ("index_references", index_references),
    ("write_sections", write_sections),
    ("write_article", write_article),
]

for i in range(len(nodes)):
    name, node = nodes[i]
    builder_of_storm.add_node(name, node, retry=RetryPolicy(max_attempts=3))
    if i > 0:
        builder_of_storm.add_edge(nodes[i - 1][0], name)

builder_of_storm.add_edge(START, nodes[0][0])
builder_of_storm.add_edge(nodes[-1][0], END)
storm = builder_of_storm.compile(checkpointer=MemorySaver())
```

## Usage Example
```python
# Run the workflow with a specific topic
config = {"configurable": {"thread_id": "my-thread"}}
async for step in storm.astream(
    {
        "topic": "Groq, NVIDIA, Llamma.cpp and the future of LLM Inference",
    },
    config,
):
    name = next(iter(step))
    print(name)
    print("-- ", str(step[name])[:300])

# Retrieve the final article
checkpoint = storm.get_state(config)
article = checkpoint.values["article"]

# Display the output
from IPython.display import Markdown
Markdown(article.replace("\n#", "\n##"))
```

## Benefits
- Produces more comprehensive research articles with richer references
- Improves information density through multi-perspective research
- Creates more organized content by building and refining outlines
- Implements a systematic approach to gathering knowledge
- Uses conversation simulation for deeper exploration of topics

## Considerations
- Performance depends on the quality of search results
- Requires multiple API calls, which can increase costs
- May generate redundant information across different expert interviews
- The number of perspectives (N) and conversation turns (M) are hyperparameters that need tuning
- Processing large amounts of content requires models with sufficient context windows

# Web Research with STORM

## Overview
STORM is a research assistant framework that generates Wikipedia-style articles on user-provided topics by implementing outline-driven RAG with multi-perspective conversational simulation to improve comprehensiveness and information density.

## Key Concepts
- Creates outlines via similar topic search to improve knowledge coverage
- Employs multi-perspective, grounded search conversation simulation 
- Uses subject matter expert role-playing to increase reference count
- Follows a sequential workflow: outline → survey → perspectives → interviews → refinement → writing

## Prerequisites
```python
# Install required packages
!pip install -U langchain_community langchain_openai langchain_fireworks langgraph wikipedia duckduckgo-search tavily-python

# Optional for visualization
# !brew install graphviz
# !CFLAGS="-I $(brew --prefix graphviz)/include" LDFLAGS="-L $(brew --prefix graphviz)/lib" pip install -U pygraphviz

# Set environment variables
import os
import getpass

def _set_env(var: str):
    if os.environ.get(var):
        return
    os.environ[var] = getpass.getpass(var + ":")

_set_env("OPENAI_API_KEY")
_set_env("TAVILY_API_KEY")

# LLM Setup
from langchain_openai import ChatOpenAI

fast_llm = ChatOpenAI(model="gpt-4o-mini")
long_context_llm = ChatOpenAI(model="gpt-4o")
```

## Implementation

### 1. Generate Initial Outline
```python
from typing import List, Optional
from langchain_core.prompts import ChatPromptTemplate
from pydantic import BaseModel, Field

direct_gen_outline_prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            "You are a Wikipedia writer. Write an outline for a Wikipedia page about a user-provided topic. Be comprehensive and specific.",
        ),
        ("user", "{topic}"),
    ]
)

class Subsection(BaseModel):
    subsection_title: str = Field(..., title="Title of the subsection")
    description: str = Field(..., title="Content of the subsection")

    @property
    def as_str(self) -> str:
        return f"### {self.subsection_title}\n\n{self.description}".strip()

class Section(BaseModel):
    section_title: str = Field(..., title="Title of the section")
    description: str = Field(..., title="Content of the section")
    subsections: Optional[List[Subsection]] = Field(
        default=None,
        title="Titles and descriptions for each subsection of the Wikipedia page.",
    )

    @property
    def as_str(self) -> str:
        subsections = "\n\n".join(
            f"### {subsection.subsection_title}\n\n{subsection.description}"
            for subsection in self.subsections or []
        )
        return f"## {self.section_title}\n\n{self.description}\n\n{subsections}".strip()

class Outline(BaseModel):
    page_title: str = Field(..., title="Title of the Wikipedia page")
    sections: List[Section] = Field(
        default_factory=list,
        title="Titles and descriptions for each section of the Wikipedia page.",
    )

    @property
    def as_str(self) -> str:
        sections = "\n\n".join(section.as_str for section in self.sections)
        return f"# {self.page_title}\n\n{sections}".strip()

generate_outline_direct = direct_gen_outline_prompt | fast_llm.with_structured_output(Outline)

# Generate initial outline
example_topic = "Impact of million-plus token context window language models on RAG"
initial_outline = generate_outline_direct.invoke({"topic": example_topic})
```

### 2. Expand Topics
```python
gen_related_topics_prompt = ChatPromptTemplate.from_template(
    """I'm writing a Wikipedia page for a topic mentioned below. Please identify and recommend some Wikipedia pages on closely related subjects. I'm looking for examples that provide insights into interesting aspects commonly associated with this topic, or examples that help me understand the typical content and structure included in Wikipedia pages for similar topics.

Please list the as many subjects and urls as you can.

Topic of interest: {topic}
"""
)

class RelatedSubjects(BaseModel):
    topics: List[str] = Field(
        description="Comprehensive list of related subjects as background research.",
    )

expand_chain = gen_related_topics_prompt | fast_llm.with_structured_output(RelatedSubjects)
related_subjects = await expand_chain.ainvoke({"topic": example_topic})
```

### 3. Generate Perspectives
```python
class Editor(BaseModel):
    affiliation: str = Field(
        description="Primary affiliation of the editor.",
    )
    name: str = Field(
        description="Name of the editor.", pattern=r"^[a-zA-Z0-9_-]{1,64}$"
    )
    role: str = Field(
        description="Role of the editor in the context of the topic.",
    )
    description: str = Field(
        description="Description of the editor's focus, concerns, and motives.",
    )

    @property
    def persona(self) -> str:
        return f"Name: {self.name}\nRole: {self.role}\nAffiliation: {self.affiliation}\nDescription: {self.description}\n"

class Perspectives(BaseModel):
    editors: List[Editor] = Field(
        description="Comprehensive list of editors with their roles and affiliations.",
    )

gen_perspectives_prompt = ChatPromptTemplate.from_messages([
    (
        "system",
        """You need to select a diverse (and distinct) group of Wikipedia editors who will work together to create a comprehensive article on the topic. Each of them represents a different perspective, role, or affiliation related to this topic.
You can use other Wikipedia pages of related topics for inspiration. For each editor, add a description of what they will focus on.

Wiki page outlines of related topics for inspiration:
{examples}""",
    ),
    ("user", "Create a diverse group of editors for a Wikipedia article on {topic}"),
])

generate_perspectives = gen_perspectives_prompt | fast_llm.with_structured_output(Perspectives)
perspectives = generate_perspectives.invoke(
    {"topic": example_topic, "examples": initial_outline.as_str}
)
```

### 4. Conduct Expert Interviews
```python
from typing import Dict, List, TypedDict
from langchain_core.messages import AIMessage, BaseMessage, HumanMessage
from langgraph.graph import END, StateGraph, START
from langgraph.prebuilt import ToolExecutor
from langgraph.graph.message import add_messages
from langchain.tools import tool
from langchain.schema.runnable import RunnableMap, RunnableLambda
from langgraph.graph.retry import RetryPolicy

class InterviewState(TypedDict):
    editor: Editor
    messages: List[BaseMessage]

max_num_turns = 3

gen_answer = RunnableMap(
    {
        "llm_answer": answer_question_chain,
        "message": lambda x: AIMessage(
            content=x["llm_answer"]["answer"], name="Subject_Matter_Expert"
        ),
    }
) | RunnableLambda(lambda x: {"messages": [x["message"]]})

def route_messages(state: InterviewState, name: str = "Subject_Matter_Expert"):
    messages = state["messages"]
    num_responses = len(
        [m for m in messages if isinstance(m, AIMessage) and m.name == name]
    )
    if num_responses >= max_num_turns:
        return END
    last_question = messages[-2]
    if last_question.content.endswith("Thank you so much for your help!"):
        return END
    return "ask_question"

builder = StateGraph(InterviewState)
builder.add_node("ask_question", generate_question, retry=RetryPolicy(max_attempts=5))
builder.add_node("answer_question", gen_answer, retry=RetryPolicy(max_attempts=5))
builder.add_conditional_edges("answer_question", route_messages)
builder.add_edge("ask_question", "answer_question")
builder.add_edge(START, "ask_question")
interview_graph = builder.compile(checkpointer=False).with_config(
    run_name="Conduct Interviews"
)
```

### 5. Refine Outline
```python
refine_outline_prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            """You are a Wikipedia writer. You have gathered information from experts and search engines. Now, you are refining the outline of the Wikipedia page. 
You need to make sure that the outline is comprehensive and specific. 
Topic you are writing about: {topic} 

Old outline:

{old_outline}""",
        ),
        (
            "user",
            "Refine the outline based on your conversations with subject-matter experts:\n\nConversations:\n\n{conversations}\n\nWrite the refined Wikipedia outline:",
        ),
    ]
)

refine_outline_chain = refine_outline_prompt | long_context_llm.with_structured_output(Outline)
refined_outline = refine_outline_chain.invoke(
    {
        "topic": example_topic,
        "old_outline": initial_outline.as_str,
        "conversations": "\n\n".join(
            f"### {m.name}\n\n{m.content}" for m in final_state["messages"]
        ),
    }
)
```

### 6. Building the Complete Workflow
```python
from langgraph.checkpoint.memory import MemorySaver

builder_of_storm = StateGraph(ResearchState)

nodes = [
    ("init_research", initialize_research),
    ("conduct_interviews", conduct_interviews),
    ("refine_outline", refine_outline),
    ("index_references", index_references),
    ("write_sections", write_sections),
    ("write_article", write_article),
]

for i in range(len(nodes)):
    name, node = nodes[i]
    builder_of_storm.add_node(name, node, retry=RetryPolicy(max_attempts=3))
    if i > 0:
        builder_of_storm.add_edge(nodes[i - 1][0], name)

builder_of_storm.add_edge(START, nodes[0][0])
builder_of_storm.add_edge(nodes[-1][0], END)
storm = builder_of_storm.compile(checkpointer=MemorySaver())
```

## Usage Example
```python
# Run the workflow with a specific topic
config = {"configurable": {"thread_id": "my-thread"}}
async for step in storm.astream(
    {
        "topic": "Groq, NVIDIA, Llamma.cpp and the future of LLM Inference",
    },
    config,
):
    name = next(iter(step))
    print(name)
    print("-- ", str(step[name])[:300])

# Retrieve the final article
checkpoint = storm.get_state(config)
article = checkpoint.values["article"]

# Display the output
from IPython.display import Markdown
Markdown(article.replace("\n#", "\n##"))
```

## Benefits
- Produces more comprehensive research articles with richer references
- Improves information density through multi-perspective research
- Creates more organized content by building and refining outlines
- Implements a systematic approach to gathering knowledge
- Uses conversation simulation for deeper exploration of topics

## Considerations
- Performance depends on the quality of search results
- Requires multiple API calls, which can increase costs
- May generate redundant information across different expert interviews
- The number of perspectives (N) and conversation turns (M) are hyperparameters that need tuning
- Processing large amounts of content requires models with sufficient context windows

6. [Future Directions](#Future-Directions)

7. [Conclusion](#Conclusion)

8. [References](#References)



## Background



### Million-Plus Token Context Window Language Models



Million-plus token context window language models, exemplified by systems like Gemini 1.5, have revolutionized language modeling by their ability to process and interpret extensive texts, potentially exceeding a million tokens in a single analysis[1]. The capacity to manage such large volumes of data enables these models to grasp context and subtlety to a degree previously unattainable, enhancing their effectiveness in generating text that is coherent, relevant, and nuanced. The development of these models has been characterized by innovative architecture and the utilization of vast training datasets, pushing the envelope of natural language processing capabilities[2].



### Retrieval-Augmented Generation (RAG)



RAG systems represent an innovative paradigm in AI, merging the strengths of retrieval-based and generative models to improve the quality and relevance of text generation[3]. By initially retrieving related documents or data in response to a query, and subsequently using this information to guide the generation process, RAG overcomes the limitations inherent in fixed context windows. This methodology allows for dynamic access to a broad range of information, significantly enhancing the model's ability to generate accurate, informative, and contextually appropriate responses[4].



## Integration of Million-Plus Token Context Window Models and RAG



The integration of million-plus token context window models with RAG systems has been a natural progression in the quest for more sophisticated NLP solutions. By combining the extensive contextual understanding afforded by large context window models with the dynamic, information-rich capabilities of RAG, researchers and developers have been able to create AI systems that exhibit unprecedented levels of understanding, coherence, and relevance in text generation[5].



## Impact on Natural Language Processing



The fusion of these technologies has had a significant impact on the field of NLP, leading to advancements in several key areas:

- **Enhanced Understanding**: The combined system exhibits a deeper comprehension of both the immediate context and broader subject matter[6].

- **Improved Coherence**: Generated text is more coherent over longer passages, maintaining consistency and relevance[7].

- **Increased Relevance**: Outputs are more contextually relevant, drawing accurately from a wider range of sources[8].



## Applications



This technological convergence has broadened the applicability of NLP systems in numerous fields, including but not limited to:

- **Automated Content Creation**: Generating written content that is both informative and contextually appropriate for various platforms[9].

- **Customer Support**: Providing answers that are not only accurate but also tailored to the specific context of user inquiries[10].

- **Research Assistance**: Assisting in literature review and data analysis by retrieving and synthesizing relevant information from vast databases[11].



## Challenges and Limitations



Despite their advancements, the integration of these technologies faces several challenges:

- **Computational Resources**: The processing of million-plus tokens and the dynamic retrieval of relevant information require significant computational power[12].

- **Data Privacy and Security**: Ensuring the confidentiality and integrity of the data accessed by these systems poses ongoing concerns[13].

- **Bias and Fairness**: The potential for inheriting and amplifying biases from training data remains a critical issue to address[14].



## Future Directions



Future research is likely to focus on optimizing computational efficiency, enhancing the models' ability to understand and generate more diverse and nuanced text, and addressing ethical considerations associated with AI and NLP technologies[15].



## Conclusion



The integration of million-plus token context window language models with RAG systems represents a milestone in the evolution of natural language processing, offering enhanced capabilities that have significant implications across various applications. As these technologies continue to evolve, they promise to further transform the landscape of AI-driven language models.



## References



1. Gemini 1.5 Documentation. (n.d.).

2. The Evolution of Language Models. (2022).

3. Introduction to Retrieval-Augmented Generation. (2021).

4. Leveraging Large Context Windows for NLP. (2023).

5. Integrating Context Window Models with RAG. (2023).

6. Deep Learning in NLP. (2020).

7. Coherence in Text Generation. (2019).

8. Contextual Relevance in AI. (2021).

9. Applications of NLP in Content Creation. (2022).

10. AI in Customer Support. (2023).

11. NLP for Research Assistance. (2021).

12. Computational Challenges in NLP. (2022).

13. Data Privacy in AI Systems. (2020).

14. Addressing Bias in AI. (2021).

15. Future of NLP Technologies. (2023).


Final Flow¶

Now it's time to string everything together. We will have 6 main stages in sequence: . 1. Generate the initial outline + perspectives 2. Batch converse with each perspective to expand the content for the article 3. Refine the outline based on the conversations 4. Index the reference docs from the conversations 5. Write the individual sections of the article 6. Write the final wiki

The state tracks the outputs of each stage.

class ResearchState(TypedDict):

    topic: str

    outline: Outline

    editors: List[Editor]

    interview_results: List[InterviewState]

    # The final sections output

    sections: List[WikiSection]

    article: str

import asyncio





async def initialize_research(state: ResearchState):

    topic = state["topic"]

    coros = (

        generate_outline_direct.ainvoke({"topic": topic}),

        survey_subjects.ainvoke(topic),

    )

    results = await asyncio.gather(*coros)

    return {

        **state,

        "outline": results[0],

        "editors": results[1].editors,

    }





async def conduct_interviews(state: ResearchState):

    topic = state["topic"]

    initial_states = [

        {

            "editor": editor,

            "messages": [

                AIMessage(

                    content=f"So you said you were writing an article on {topic}?",

                    name="Subject_Matter_Expert",

                )

            ],

        }

        for editor in state["editors"]

    ]

    # We call in to the sub-graph here to parallelize the interviews

    interview_results = await interview_graph.abatch(initial_states)



    return {

        **state,

        "interview_results": interview_results,

    }





def format_conversation(interview_state):

    messages = interview_state["messages"]

    convo = "\n".join(f"{m.name}: {m.content}" for m in messages)

    return f'Conversation with {interview_state["editor"].name}\n\n' + convo





async def refine_outline(state: ResearchState):

    convos = "\n\n".join(

        [

            format_conversation(interview_state)

            for interview_state in state["interview_results"]

        ]

    )



    updated_outline = await refine_outline_chain.ainvoke(

        {

            "topic": state["topic"],

            "old_outline": state["outline"].as_str,

            "conversations": convos,

        }

    )

    return {**state, "outline": updated_outline}





async def index_references(state: ResearchState):

    all_docs = []

    for interview_state in state["interview_results"]:

        reference_docs = [

            Document(page_content=v, metadata={"source": k})

            for k, v in interview_state["references"].items()

        ]

        all_docs.extend(reference_docs)

    await vectorstore.aadd_documents(all_docs)

    return state





async def write_sections(state: ResearchState):

    outline = state["outline"]

    sections = await section_writer.abatch(

        [

            {

                "outline": refined_outline.as_str,

                "section": section.section_title,

                "topic": state["topic"],

            }

            for section in outline.sections

        ]

    )

    return {

        **state,

        "sections": sections,

    }





async def write_article(state: ResearchState):

    topic = state["topic"]

    sections = state["sections"]

    draft = "\n\n".join([section.as_str for section in sections])

    article = await writer.ainvoke({"topic": topic, "draft": draft})

    return {

        **state,

        "article": article,

    }

Create the graph¶
from langgraph.checkpoint.memory import MemorySaver



builder_of_storm = StateGraph(ResearchState)



nodes = [

    ("init_research", initialize_research),

    ("conduct_interviews", conduct_interviews),

    ("refine_outline", refine_outline),

    ("index_references", index_references),

    ("write_sections", write_sections),

    ("write_article", write_article),

]

for i in range(len(nodes)):

    name, node = nodes[i]

    builder_of_storm.add_node(name, node, retry=RetryPolicy(max_attempts=3))

    if i > 0:

        builder_of_storm.add_edge(nodes[i - 1][0], name)



builder_of_storm.add_edge(START, nodes[0][0])

builder_of_storm.add_edge(nodes[-1][0], END)

storm = builder_of_storm.compile(checkpointer=MemorySaver())


API Reference: MemorySaver

from IPython.display import Image, display



try:

    display(Image(storm.get_graph().draw_mermaid_png()))

except Exception:

    # This requires some extra dependencies and is optional

    pass


config = {"configurable": {"thread_id": "my-thread"}}

async for step in storm.astream(

    {

        "topic": "Groq, NVIDIA, Llamma.cpp and the future of LLM Inference",

    },

    config,

):

    name = next(iter(step))

    print(name)

    print("-- ", str(step[name])[:300])

init_research

--  {'topic': 'Groq, NVIDIA, Llamma.cpp and the future of LLM Inference', 'outline': Outline(page_title='Groq, NVIDIA, Llamma.cpp and the future of LLM Inference', sections=[Section(section_title='Introduction', description='Overview of Groq, NVIDIA, Llamma.cpp, and their significance in the field of La

conduct_interviews

--  {'topic': 'Groq, NVIDIA, Llamma.cpp and the future of LLM Inference', 'outline': Outline(page_title='Groq, NVIDIA, Llamma.cpp and the future of LLM Inference', sections=[Section(section_title='Introduction', description='Overview of Groq, NVIDIA, Llamma.cpp, and their significance in the field of La

refine_outline

--  {'topic': 'Groq, NVIDIA, Llamma.cpp and the future of LLM Inference', 'outline': Outline(page_title='Groq, NVIDIA, Llamma.cpp and the Future of LLM Inference', sections=[Section(section_title='Introduction', description='An overview of the significance and roles of Groq, NVIDIA, and Llamma.cpp in th

index_references

--  {'topic': 'Groq, NVIDIA, Llamma.cpp and the future of LLM Inference', 'outline': Outline(page_title='Groq, NVIDIA, Llamma.cpp and the Future of LLM Inference', sections=[Section(section_title='Introduction', description='An overview of the significance and roles of Groq, NVIDIA, and Llamma.cpp in th

write_sections

--  {'topic': 'Groq, NVIDIA, Llamma.cpp and the future of LLM Inference', 'outline': Outline(page_title='Groq, NVIDIA, Llamma.cpp and the Future of LLM Inference', sections=[Section(section_title='Introduction', description='An overview of the significance and roles of Groq, NVIDIA, and Llamma.cpp in th

write_article

--  {'topic': 'Groq, NVIDIA, Llamma.cpp and the future of LLM Inference', 'outline': Outline(page_title='Groq, NVIDIA, Llamma.cpp and the Future of LLM Inference', sections=[Section(section_title='Introduction', description='An overview of the significance and roles of Groq, NVIDIA, and Llamma.cpp in th

__end__

--  {'topic': 'Groq, NVIDIA, Llamma.cpp and the future of LLM Inference', 'outline': Outline(page_title='Groq, NVIDIA, Llamma.cpp and the Future of LLM Inference', sections=[Section(section_title='Introduction', description='An overview of the significance and roles of Groq, NVIDIA, and Llamma.cpp in th


checkpoint = storm.get_state(config)

article = checkpoint.values["article"]

Render the Wiki¶

Now we can render the final wiki page!

from IPython.display import Markdown



# We will down-header the sections to create less confusion in this notebook

Markdown(article.replace("\n#", "\n##"))

Large Language Model (LLM) Inference Technologies¶
Contents¶
Introduction
Groq's Advancements in LLM Inference
NVIDIA's Contributions to LLM Inference
Hardware Innovations
Software Solutions
Research and Development
Llamma.cpp: Accelerating LLM Inference
The Future of LLM Inference
References
Introduction¶

The advent of million-plus token context window language models, such as Gemini 1.5, has significantly advanced the field of artificial intelligence, particularly in natural language processing (NLP). These models have expanded the capabilities of machine learning in understanding and generating text over vastly larger contexts than previously possible. This leap in technology has paved the way for transformative applications across various domains, including the integration into Retrieval-Augmented Generation (RAG) systems to produce more accurate and contextually rich responses.

Groq's Advancements in LLM Inference¶

Groq has introduced the Groq Linear Processor Unit (LPU), a purpose-built hardware architecture for LLM inference. This innovation positions Groq as a leader in efficient and high-performance LLM processing by optimizing the hardware specifically for LLM tasks. The Groq LPU dramatically reduces latency and increases the throughput of LLM inferences, facilitating advancements in a wide range of applications, from natural language processing to broader artificial intelligence technologies[1].

NVIDIA's Contributions to LLM Inference¶

NVIDIA has played a pivotal role in advancing LLM inference through its GPUs, optimized for AI and machine learning workloads, and specialized software frameworks. The company's GPU architecture and software solutions, such as the CUDA Deep Neural Network library (cuDNN) and the TensorRT inference optimizer, are designed to accelerate computational processes and improve LLM performance. NVIDIA's active participation in research and development further underscores its commitment to enhancing the capabilities of LLMs[1].

Hardware Innovations¶

NVIDIA's GPU architecture facilitates high throughput and parallel processing for LLM inference tasks, significantly reducing inference time and enabling complex models to be used in real-time applications.

Software Solutions¶

NVIDIA's suite of software tools, including cuDNN and TensorRT, optimizes LLM performance on its hardware, streamlining the deployment of LLMs by improving their efficiency and reducing latency.

Research and Development¶

NVIDIA collaborates with academic and industry partners to develop new techniques and models that push the boundaries of LLM technology, aiming to make LLMs more powerful and applicable across a broader range of tasks.

Llamma.cpp: Accelerating LLM Inference¶

Llamma.cpp is a framework developed to enhance the speed and efficiency of LLM inference. By integrating specialized hardware, such as Groq's LPU, and optimizing for parallel processing, Llamma.cpp significantly accelerates computation times and reduces energy consumption. The framework supports million-plus token context window models, enabling applications requiring deep contextual understanding and extensive knowledge retrieval[1][2].

The Future of LLM Inference¶

The future of LLM inference is poised for transformative changes with advances in purpose-built hardware architectures like Groq's LPU. These innovations promise to enhance the speed and efficiency of LLM processing, leading to more interactive, capable, and integrated AI applications. The potential for advanced hardware and sophisticated LLMs to enable near-instantaneous processing of complex queries and interactions opens new avenues for research and application in various fields, suggesting a future where AI is seamlessly integrated into society[1][2].

References¶

[1] "Groq's LPU: Advancing LLM Inference Efficiency," Prompt Engineering. https://promptengineering.org/groqs-lpu-advancing-llm-inference-efficiency/

[2] "The Speed of Thought: Harnessing the Fastest LLM with Groq's LPU," Medium. https://medium.com/@anasdavoodtk1/the-speed-of-thought-harnessing-the-fastest-llm-with-groqs-lpu-11bb00864e9c

Comments
