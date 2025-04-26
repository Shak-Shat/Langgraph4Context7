# Web-Enabled Navigation Agent (Web Voyager)

## Overview
Web Voyager is a vision-enabled web browsing agent that controls mouse and keyboard actions by viewing annotated browser screenshots and choosing appropriate next steps. It implements a reasoning and action (ReAct) loop with unique features for browser interaction.

## Key Concepts
- Uses set-of-marks image annotations as UI affordances for the agent
- Controls both mouse and keyboard through browser automation
- Processes annotated screenshots for visual understanding
- Makes decisions through a multi-modal LLM
- Works through a ReAct loop of observation, reasoning, and action

## Prerequisites
```python
# Install required packages
%pip install -U --quiet langgraph langsmith langchain_openai
%pip install --upgrade --quiet playwright > /dev/null

# Install browser automation
!playwright install

# Required for running async playwright in Jupyter
import nest_asyncio
nest_asyncio.apply()

# Set up API keys
import os
from getpass import getpass

def _getpass(env_var: str):
    if not os.environ.get(env_var):
        os.environ[env_var] = getpass(f"{env_var}=")

_getpass("OPENAI_API_KEY")
```

## Implementation

### 1. Define Agent State
```python
from typing import List, Optional
from typing_extensions import TypedDict
from langchain_core.messages import BaseMessage, SystemMessage
from playwright.async_api import Page

class BBox(TypedDict):
    x: float
    y: float
    text: str
    type: str
    ariaLabel: str

class Prediction(TypedDict):
    action: str
    args: Optional[List[str]]

# State representing agent's execution context
class AgentState(TypedDict):
    page: Page  # The Playwright web page
    input: str  # User request
    img: str  # b64 encoded screenshot
    bboxes: List[BBox]  # Bounding boxes from browser annotation
    prediction: Prediction  # Agent's output
    scratchpad: List[BaseMessage]  # System messages with intermediate steps
    observation: str  # Most recent tool response
```

### 2. Define Web Interaction Tools
```python
import asyncio
import platform

async def click(state: AgentState):
    # Click on labeled element
    page = state["page"]
    click_args = state["prediction"]["args"]
    if click_args is None or len(click_args) != 1:
        return f"Failed to click bounding box labeled as number {click_args}"
    
    bbox_id = int(click_args[0])
    try:
        bbox = state["bboxes"][bbox_id]
    except Exception:
        return f"Error: no bbox for : {bbox_id}"
    
    x, y = bbox["x"], bbox["y"]
    await page.mouse.click(x, y)
    return f"Clicked {bbox_id}"

async def type_text(state: AgentState):
    page = state["page"]
    type_args = state["prediction"]["args"]
    if type_args is None or len(type_args) != 2:
        return f"Failed to type in element from bounding box labeled as number {type_args}"
    
    bbox_id = int(type_args[0])
    bbox = state["bboxes"][bbox_id]
    x, y = bbox["x"], bbox["y"]
    text_content = type_args[1]
    
    await page.mouse.click(x, y)
    # Clear existing text
    select_all = "Meta+A" if platform.system() == "Darwin" else "Control+A"
    await page.keyboard.press(select_all)
    await page.keyboard.press("Backspace")
    await page.keyboard.type(text_content)
    await page.keyboard.press("Enter")
    
    return f"Typed {text_content} and submitted"

async def scroll(state: AgentState):
    page = state["page"]
    scroll_args = state["prediction"]["args"]
    if scroll_args is None or len(scroll_args) != 2:
        return "Failed to scroll due to incorrect arguments."

    target, direction = scroll_args
    
    if target.upper() == "WINDOW":
        scroll_amount = 500
        scroll_direction = -scroll_amount if direction.lower() == "up" else scroll_amount
        await page.evaluate(f"window.scrollBy(0, {scroll_direction})")
    else:
        scroll_amount = 200
        target_id = int(target)
        bbox = state["bboxes"][target_id]
        x, y = bbox["x"], bbox["y"]
        scroll_direction = -scroll_amount if direction.lower() == "up" else scroll_amount
        await page.mouse.move(x, y)
        await page.mouse.wheel(0, scroll_direction)
    
    return f"Scrolled {direction} in {'window' if target.upper() == 'WINDOW' else 'element'}"

async def wait(state: AgentState):
    sleep_time = 5
    await asyncio.sleep(sleep_time)
    return f"Waited for {sleep_time}s."

async def go_back(state: AgentState):
    page = state["page"]
    await page.go_back()
    return f"Navigated back a page to {page.url}."

async def to_google(state: AgentState):
    page = state["page"]
    await page.goto("https://www.google.com/")
    return "Navigated to google.com."
```

### 3. Implement Browser Page Annotation
```python
import base64
from langchain_core.runnables import chain as chain_decorator

# Load JavaScript annotation script
with open("mark_page.js") as f:
    mark_page_script = f.read()

@chain_decorator
async def mark_page(page):
    await page.evaluate(mark_page_script)
    for _ in range(10):
        try:
            bboxes = await page.evaluate("markPage()")
            break
        except Exception:
            # May be loading...
            asyncio.sleep(3)
    
    screenshot = await page.screenshot()
    # Ensure the bboxes don't follow us around
    await page.evaluate("unmarkPage()")
    
    return {
        "img": base64.b64encode(screenshot).decode(),
        "bboxes": bboxes,
    }
```

### 4. Create the Agent's Decision-Making Logic
```python
from langchain import hub
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough
from langchain_openai import ChatOpenAI

async def annotate(state):
    marked_page = await mark_page.with_retry().ainvoke(state["page"])
    return {**state, **marked_page}

def format_descriptions(state):
    labels = []
    for i, bbox in enumerate(state["bboxes"]):
        text = bbox.get("ariaLabel") or ""
        if not text.strip():
            text = bbox["text"]
        el_type = bbox.get("type")
        labels.append(f'{i} (<{el_type}/>): "{text}"')
    bbox_descriptions = "\nValid Bounding Boxes:\n" + "\n".join(labels)
    return {**state, "bbox_descriptions": bbox_descriptions}

def parse(text: str) -> dict:
    action_prefix = "Action: "
    if not text.strip().split("\n")[-1].startswith(action_prefix):
        return {"action": "retry", "args": f"Could not parse LLM Output: {text}"}
    
    action_block = text.strip().split("\n")[-1]
    action_str = action_block[len(action_prefix):]
    split_output = action_str.split(" ", 1)
    
    if len(split_output) == 1:
        action, action_input = split_output[0], None
    else:
        action, action_input = split_output
    
    action = action.strip()
    if action_input is not None:
        action_input = [inp.strip().strip("[]") for inp in action_input.strip().split(";")]
    
    return {"action": action, "args": action_input}

# Pull prompt template from hub
prompt = hub.pull("wfh/web-voyager")

# Set up multimodal LLM
llm = ChatOpenAI(model="gpt-4-vision-preview", max_tokens=4096)

# Assemble agent components
agent = annotate | RunnablePassthrough.assign(
    prediction=format_descriptions | prompt | llm | StrOutputParser() | parse
)
```

### 5. Build the Agent Graph
```python
import re
from langchain_core.runnables import RunnableLambda
from langgraph.graph import END, START, StateGraph

# Function to update agent's scratchpad with previous actions
def update_scratchpad(state: AgentState):
    old = state.get("scratchpad")
    if old:
        txt = old[0].content
        last_line = txt.rsplit("\n", 1)[-1]
        step = int(re.match(r"\d+", last_line).group()) + 1
    else:
        txt = "Previous action observations:\n"
        step = 1
    
    txt += f"\n{step}. {state['observation']}"
    return {**state, "scratchpad": [SystemMessage(content=txt)]}

# Create the state graph
graph_builder = StateGraph(AgentState)

# Add agent node
graph_builder.add_node("agent", agent)
graph_builder.add_edge(START, "agent")

# Add scratchpad update node
graph_builder.add_node("update_scratchpad", update_scratchpad)
graph_builder.add_edge("update_scratchpad", "agent")

# Define available tools
tools = {
    "Click": click,
    "Type": type_text,
    "Scroll": scroll,
    "Wait": wait,
    "GoBack": go_back,
    "Google": to_google,
}

# Add tool nodes
for node_name, tool in tools.items():
    graph_builder.add_node(
        node_name,
        # Map tool output to "observation" key in state
        RunnableLambda(tool) | (lambda observation: {"observation": observation}),
    )
    # All tools return to update_scratchpad
    graph_builder.add_edge(node_name, "update_scratchpad")

# Router function to select appropriate tool
def select_tool(state: AgentState):
    action = state["prediction"]["action"]
    if action == "ANSWER":
        return END
    if action == "retry":
        return "agent"
    return action

# Add conditional routing
graph_builder.add_conditional_edges("agent", select_tool)

# Compile the graph
graph = graph_builder.compile()
```

## Usage Example
```python
from IPython import display
from playwright.async_api import async_playwright

# Initialize browser
browser = await async_playwright().start()
browser = await browser.chromium.launch(headless=False, args=None)
page = await browser.new_page()
_ = await page.goto("https://www.google.com")

# Function to execute agent
async def call_agent(question: str, page, max_steps: int = 150):
    event_stream = graph.astream(
        {
            "page": page,
            "input": question,
            "scratchpad": [],
        },
        {
            "recursion_limit": max_steps,
        },
    )
    
    final_answer = None
    steps = []
    
    async for event in event_stream:
        if "agent" not in event:
            continue
        
        pred = event["agent"].get("prediction") or {}
        action = pred.get("action")
        action_input = pred.get("args")
        
        display.clear_output(wait=False)
        steps.append(f"{len(steps) + 1}. {action}: {action_input}")
        print("\n".join(steps))
        display.display(display.Image(base64.b64decode(event["agent"]["img"])))
        
        if "ANSWER" in action:
            final_answer = action_input[0]
            break
    
    return final_answer

# Example query
result = await call_agent("Could you explain the WebVoyager paper (on arxiv)?", page)
print(f"Final response: {result}")
```

## Benefits
- Enables autonomous web navigation without specialized APIs
- Bridges the gap between LLMs and web interfaces
- Works with any website, including those with complex layouts
- Provides a visual context for decision making
- Creates reproducible sequences of web interactions

## Considerations
- Requires proper annotation of web elements for reliable operation
- Performance depends on visual recognition capabilities of the vision model
- Browser automation may be affected by website changes or CAPTCHA systems
- Actions are sequential and can be time-consuming for complex workflows
- JavaScript annotation script must be maintained separately
- Sensitive to element positioning changes on websites
