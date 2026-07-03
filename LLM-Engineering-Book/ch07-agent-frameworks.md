# Chapter 7: Agent Frameworks

Up to now, our LLM applications have mostly followed a fixed path: a prompt goes in, an answer comes out, maybe after retrieving some documents. **Agents** break that mold. An agent can *decide* what to do next — which tool to call, whether to search again, when it's finished — looping until the goal is met. This is the leap from "a smart autocomplete" to "a digital worker that gets things done."

This is the longest chapter in the book because agents are where the whole LLM ecosystem comes together: prompts, tools (Chapter 6's MCP), retrieval (Chapters 4-5), and control logic. We'll define what an agent is, walk through the four leading frameworks (LangGraph, CrewAI, AutoGen, smolagents) with runnable code, go deep on tool calling and design patterns, and build a finance equity-research agent end to end.

Install what we use across the chapter:

```bash
pip install langgraph langchain-openai crewai autogen-agentchat autogen-ext smolagents
```

---

## 7.1 What Is an Agent? The ReAct Loop

**Simple definition:** An agent is an LLM-powered system that can reason about a goal, choose and use tools, observe the results, and repeat this loop until the task is done.

**Intuition:** A regular chain is like a printed recipe — fixed steps, same order every time. An agent is like a chef: given "make dinner," the chef decides to check the fridge (a tool), sees there's no pasta, decides to grill fish instead, tastes it (observes), adjusts the seasoning, and stops when the dish is right. The chef *chooses* the steps based on what they observe.

### 7.1.1 The ReAct loop

The dominant agent pattern is **ReAct**, short for **Reason + Act**. The agent alternates between *thinking* and *doing*:

```
        +-----------------------------------------------+
        |                                               |
        v                                               |
   [ THOUGHT ]  -> "I need the current stock price."    |
        |                                               |
        v                                               |
   [ ACTION ]   -> call tool: get_price("AAPL")         |
        |                                               |
        v                                               |
   [ OBSERVATION ] -> tool returns: $192.50             |
        |                                               |
        +-------- enough info? ---- NO -----------------+
                       |
                      YES
                       v
                 [ FINAL ANSWER ]
```

In words: the agent **Reasons** ("what should I do?"), takes an **Action** (calls a tool), gets an **Observation** (the tool's result), and loops. When it has enough information, it stops and gives the final answer.

**Finance use case:** Ask an agent "Is Apple trading above its 50-day average?" It reasons it needs two numbers, calls a price tool, then a moving-average tool, observes both, compares them, and answers — all decided at runtime, not hard-coded.

Here is the loop in a minimal, dependency-light form so you can see the mechanics:

```python
# A tiny illustration of the ReAct idea (pseudocode-style, real Python).
def run_agent(question, llm, tools, max_steps=5):
    scratchpad = ""                      # remembers thoughts/observations
    for step in range(max_steps):        # <- loop cap prevents infinite loops!
        # Ask the LLM what to do next, given progress so far.
        decision = llm(f"Question: {question}\nProgress:\n{scratchpad}\n"
                       "Reply with either ACTION: tool(args) or FINAL: answer")
        if decision.startswith("FINAL:"):
            return decision.replace("FINAL:", "").strip()
        # Otherwise parse and run the requested tool.
        tool_name, arg = parse(decision)          # e.g. get_price, "AAPL"
        observation = tools[tool_name](arg)        # ACT
        scratchpad += f"\nThought/Action: {decision}\nObservation: {observation}"
    return "Stopped: reached max steps."           # safety fallback
```

Note the `max_steps` cap — a first, essential guard against infinite loops, which we revisit in 7.10.

---

## 7.2 LangGraph: State Machines, Nodes, Edges, and Checkpoints

**Simple definition:** LangGraph models an agent as a **graph**: a set of **nodes** (steps that do work) connected by **edges** (which step runs next), all operating on a shared **state**.

**Intuition:** Think of a board game. The **state** is the game board (all current information). Each **node** is a square that does something and updates the board. **Edges** are the arrows telling you which square to move to next — and some edges are *conditional* ("if you rolled a 6, go here; otherwise go there"). This graph structure gives you precise, debuggable control over an agent's flow, which is why LangGraph is popular for production agents.

### 7.2.1 The four building blocks

- **State** — a shared data structure (often a `TypedDict`) that every node reads and writes.
- **Nodes** — Python functions that take the state and return an update to it.
- **Edges** — connections between nodes; **conditional edges** decide the path based on the state.
- **Checkpoints** — saved snapshots of the state, enabling memory, pausing, and resuming.

### 7.2.2 A full LangGraph agent

Let's build an agent that answers questions and can call a stock-price tool, looping until done.

```python
from typing import Annotated, TypedDict
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langchain_core.messages import HumanMessage
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition
from langgraph.checkpoint.memory import MemorySaver

# --- 1. Define a TOOL the agent can call ---
@tool
def get_stock_price(ticker: str) -> str:
    """Return the current price for a stock ticker like 'AAPL'."""
    fake = {"AAPL": 192.50, "TSLA": 245.10, "MSFT": 415.30}
    return f"{ticker} is trading at ${fake.get(ticker, 'unknown')}"

tools = [get_stock_price]

# --- 2. Define the STATE. add_messages appends new messages to history ---
class State(TypedDict):
    messages: Annotated[list, add_messages]

# --- 3. Bind the tools to the model so it knows it can call them ---
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0).bind_tools(tools)

# --- 4. Define NODES (functions that update state) ---
def chatbot(state: State):
    # The LLM decides: answer directly, or request a tool call.
    return {"messages": [llm.invoke(state["messages"])]}

# --- 5. Wire the GRAPH ---
graph = StateGraph(State)
graph.add_node("chatbot", chatbot)
graph.add_node("tools", ToolNode(tools))      # prebuilt node that runs tools

graph.add_edge(START, "chatbot")
# CONDITIONAL edge: if the model asked for a tool -> go to 'tools', else -> END
graph.add_conditional_edges("chatbot", tools_condition)
graph.add_edge("tools", "chatbot")            # after a tool runs, think again

# --- 6. Add CHECKPOINTING for memory across turns ---
app = graph.compile(checkpointer=MemorySaver())

# --- 7. Run it. thread_id ties turns together (like a session) ---
cfg = {"configurable": {"thread_id": "user-1"}}
out = app.invoke(
    {"messages": [HumanMessage("What is Apple's stock price?")]}, config=cfg)
print(out["messages"][-1].content)
# -> "Apple (AAPL) is currently trading at $192.50."
```

**Walk-through in plain English:** The graph starts at the `chatbot` node. The LLM either answers or requests the `get_stock_price` tool. The **conditional edge** routes to the `tools` node if a tool was requested; the tool runs, and control loops back to `chatbot` to reason on the result. When no more tools are needed, it ends. The `MemorySaver` checkpointer remembers the conversation under `thread_id`, so follow-ups work and you could even pause and resume the agent.

**Finance use case:** This exact structure underlies a portfolio assistant: nodes for fetching prices, computing returns, and checking risk limits, with conditional edges deciding whether more data is needed before answering.

---

## 7.3 CrewAI: Role-Based Multi-Agent Crews

**Simple definition:** CrewAI lets you assemble a **crew** of specialized agents, each with a **role**, who collaborate on **tasks** to reach a shared goal.

**Intuition:** Instead of one generalist doing everything, you build a small team like a real company department: a Researcher who gathers facts and a Writer who turns them into a report. Each member has a clear job description (role), a motivation (goal), and a backstory that shapes their behavior. CrewAI is loved for how naturally it maps to "org chart" thinking.

### 7.3.1 A researcher + writer crew

```python
from crewai import Agent, Task, Crew, Process

# --- Define two specialized agents with roles, goals, and backstories ---
researcher = Agent(
    role="Market Researcher",
    goal="Find accurate, up-to-date facts about the given company",
    backstory="A meticulous equity analyst who never states an unverified fact.",
    verbose=True,
)

writer = Agent(
    role="Financial Writer",
    goal="Turn research notes into a clear, concise investor summary",
    backstory="A financial journalist known for plain-English explanations.",
    verbose=True,
)

# --- Define tasks; note the writer's task depends on the researcher's output ---
research_task = Task(
    description="Gather key facts about {company}: business model, "
                "recent revenue trend, and main risks.",
    expected_output="A bullet list of verified facts.",
    agent=researcher,
)

write_task = Task(
    description="Using the research, write a 150-word investor summary of {company}.",
    expected_output="A polished 150-word summary.",
    agent=writer,
    context=[research_task],       # <- receives the researcher's output
)

# --- Assemble the crew. sequential = one task after another ---
crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, write_task],
    process=Process.sequential,
)

result = crew.kickoff(inputs={"company": "Nvidia"})
print(result)
# -> A 150-word investor summary of Nvidia, built from the researcher's notes.
```

**How it flows:** The researcher runs first and produces facts. Because the writer's task lists `context=[research_task]`, it automatically receives those facts and writes the summary. The `sequential` process runs tasks in order; CrewAI also supports a `hierarchical` process where a manager agent delegates.

**Finance use case:** This researcher-plus-writer crew is a lightweight equity-research desk: one agent gathers company facts (ideally with a real search tool attached), the other packages them into an investor-ready note.

---

## 7.4 AutoGen: Conversational Multi-Agent Patterns

**Simple definition:** AutoGen (from Microsoft) frames multi-agent work as a **conversation** between agents that message each other until the task is solved.

**Intuition:** CrewAI feels like assigning tasks to employees; AutoGen feels like putting experts in a **chat room** and letting them talk it out. One agent proposes, another critiques or executes, and the dialogue continues until they agree the job is done. This conversational style is powerful for problem-solving that benefits from back-and-forth, like coding with a reviewer.

### 7.4.1 A two-agent conversation

The following uses the modern `autogen-agentchat` API.

```python
import asyncio
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.teams import RoundRobinGroupChat
from autogen_agentchat.conditions import TextMentionTermination
from autogen_ext.models.openai import OpenAIChatCompletionClient

async def main():
    model = OpenAIChatCompletionClient(model="gpt-4o-mini")

    # An analyst agent that proposes an answer.
    analyst = AssistantAgent(
        name="analyst",
        model_client=model,
        system_message="You estimate a company's valuation and explain your reasoning.",
    )

    # A critic agent that reviews and pushes back until satisfied.
    critic = AssistantAgent(
        name="critic",
        model_client=model,
        system_message=("You review the analyst's reasoning for errors. "
                        "When the analysis is sound, reply with the word APPROVE."),
    )

    # Stop the conversation once the critic says APPROVE.
    termination = TextMentionTermination("APPROVE")

    # A team where agents take turns talking.
    team = RoundRobinGroupChat([analyst, critic], termination_condition=termination)

    await team.run_stream(task="Roughly value a SaaS firm with $50M ARR growing 30%.")

asyncio.run(main())
```

**How it flows:** The analyst proposes a valuation, the critic reviews it and either objects (prompting a revision) or says `APPROVE`. The `TextMentionTermination` watches for "APPROVE" and stops the chat — a clean, natural stopping condition (and another loop-prevention technique).

**Finance use case:** This analyst-plus-critic pattern mirrors a real investment committee: one agent builds the thesis, another stress-tests it, reducing the chance of a sloppy, single-pass answer reaching a client.

---

## 7.5 smolagents: Hugging Face's Minimal Code-Agents

**Simple definition:** smolagents is a lightweight framework from Hugging Face whose signature idea is that the agent writes and runs **Python code** to take actions, rather than emitting structured JSON tool calls.

**Intuition:** Most frameworks make the agent fill out a form ("call tool X with args Y"). smolagents lets the agent *write a little program* instead. Because code can loop, branch, and combine several tool calls in one block, code-agents are often more expressive and efficient for multi-step logic. "smol" signals the philosophy: a tiny core, minimal abstractions.

### 7.5.1 A code-agent with a tool

```python
from smolagents import CodeAgent, tool, InferenceClientModel

# Define a tool with the @tool decorator; the docstring guides the agent.
@tool
def convert_currency(amount: float, rate: float) -> float:
    """Convert an amount using an exchange rate.

    Args:
        amount: the amount in the source currency.
        rate: units of target currency per one unit of source currency.
    """
    return amount * rate

# A CodeAgent reasons by WRITING PYTHON that calls the tools.
agent = CodeAgent(
    tools=[convert_currency],
    model=InferenceClientModel(),   # a hosted HF model; swap for any LLM
)

result = agent.run(
    "I have 1500 USD. If 1 USD = 83.2 INR, how many rupees is that?"
)
print(result)
# The agent writes and runs: convert_currency(1500, 83.2) -> 124800.0
```

**How it flows:** Instead of returning a JSON tool call, the model writes Python like `result = convert_currency(1500, 83.2)`, which smolagents executes in a sandbox, feeding the output back to the agent. For safety, code runs in a restricted environment (and can be sandboxed further, e.g., with E2B).

**E-commerce use case:** A pricing agent can, in a single code block, loop over a list of products, apply a discount function to each, and total the results — logic that would take many separate JSON tool calls in other frameworks.

---

## 7.6 Tool / Function Calling Deep Dive

Tools are what let agents *act*. Understanding how tool calling works under the hood makes you far better at designing reliable agents.

### 7.6.1 What a tool schema is

**Simple definition:** A tool schema is a structured description — name, purpose, and parameters (with types) — that tells the model what a tool does and how to call it.

**Intuition:** It's a job posting for the tool. The model reads the "posting" (schema) to decide whether and how to use it. The clearer the description and parameter names, the more reliably the model calls it correctly.

When you decorate a function, the framework generates a JSON schema roughly like this and sends it to the model:

```json
{
  "name": "get_stock_price",
  "description": "Return the current price for a stock ticker like 'AAPL'.",
  "parameters": {
    "type": "object",
    "properties": {
      "ticker": {"type": "string", "description": "The stock ticker symbol"}
    },
    "required": ["ticker"]
  }
}
```

The model doesn't run the function. It outputs a **request** to call it (name + arguments); *your code* runs the function and returns the result. This separation is important for security — you decide what actually executes.

### 7.6.2 The full tool-calling cycle

```python
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool

@tool
def order_total(price: float, quantity: int) -> float:
    """Compute the total cost for a quantity of an item at a unit price."""
    return price * quantity

llm = ChatOpenAI(model="gpt-4o-mini").bind_tools([order_total])

# 1. Ask the model something that needs the tool.
msg = llm.invoke("What's the total for 3 items at $19.99 each?")

# 2. The model returns a REQUEST to call the tool (it did not run it).
print(msg.tool_calls)
# -> [{'name': 'order_total', 'args': {'price': 19.99, 'quantity': 3}, 'id': '...'}]

# 3. YOUR code executes the tool with those args.
call = msg.tool_calls[0]
result = order_total.invoke(call["args"])   # -> 59.97
# 4. You send the result back to the model for a final natural-language answer.
```

### 7.6.3 Parallel tool calls

**Simple definition:** Modern models can request **several tool calls at once** when the calls are independent, so they run in parallel instead of one-by-one.

**Intuition:** If you need the prices of Apple *and* Microsoft, there's no reason to wait for one before asking the other. The model can request both simultaneously, cutting latency roughly in half.

```python
# Ask for two independent lookups; the model may return TWO tool_calls.
msg = llm2.invoke("Compare the totals: 2 items at $10, and 5 items at $4.")
for call in msg.tool_calls:      # potentially runs both concurrently
    print(call["name"], call["args"])
# -> order_total {'price': 10, 'quantity': 2}
# -> order_total {'price': 4,  'quantity': 5}
```

**Finance use case:** An agent comparing five stocks issues five parallel price lookups in one step, dramatically faster than five sequential round-trips — important for responsive trading dashboards.

---

## 7.7 Agent Design Patterns

A few reusable patterns solve most agent design problems. Knowing them helps you pick the right shape for a task.

### 7.7.1 Router

**Definition:** A router agent classifies the incoming request and *directs* it to the right specialized handler.

**Intuition:** A phone menu: "Press 1 for billing, 2 for tech support." The router just decides where the request should go; specialists do the real work.

**E-commerce use case:** A support router sends "where's my order?" to an order-tracking agent, "I want a refund" to a returns agent, and product questions to a catalog agent.

### 7.7.2 Planner-Executor

**Definition:** One component makes a step-by-step **plan**; another **executes** each step. Planning and doing are separated.

**Intuition:** A project manager writes the plan; the team executes it. Separating the two produces more coherent multi-step behavior than deciding everything one step at a time.

**Finance use case:** For "build me a report on Tesla," the planner lists steps (get financials, get news, get peer comparison, write report), and the executor runs each — exactly the shape of our use case in 7.8.

### 7.7.3 Reflection

**Definition:** The agent reviews and critiques its own output, then improves it — sometimes with a separate "critic" agent.

**Intuition:** Writing a first draft, then editing it. That second pass catches errors and raises quality. AutoGen's analyst-plus-critic (7.4) is reflection in action.

**Finance use case:** A valuation agent drafts an estimate, a critic checks the assumptions, and the analyst revises — reducing costly mistakes before a number reaches a client.

### 7.7.4 Human-in-the-loop

**Definition:** The agent pauses to get human approval before high-stakes actions.

**Intuition:** An intern who checks with the manager before sending money or emailing a client. Essential wherever a wrong action is expensive or irreversible.

**Finance use case:** An agent may draft a trade or a wire transfer but must **wait for a human to approve** before executing — a hard requirement in regulated finance. LangGraph's checkpointing (7.2) makes such pausing/resuming natural.

---

## 7.8 Use Case: An Automated Equity-Research Agent

Let's build a finance **equity-research assistant** that, given a company, searches the web for information and writes a structured research brief. It combines a real tool (web search), the ReAct loop, and the planner-executor spirit. We use LangGraph's prebuilt ReAct agent for a compact, production-shaped result.

```python
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent

# --- 1. A web-search tool. In production, wrap Tavily/SerpAPI/Bing here. ---
@tool
def web_search(query: str) -> str:
    """Search the web for recent information and return a short summary."""
    # Placeholder. Replace with a real search API call, e.g.:
    #   from langchain_community.tools.tavily_search import TavilySearchResults
    #   return TavilySearchResults(max_results=3).invoke(query)
    return f"[search results for '{query}': recent revenue, news, and analyst views]"

# --- 2. A tool to fetch structured financials (wrap yfinance / an API) ---
@tool
def get_financials(ticker: str) -> str:
    """Return key financial metrics (revenue, margin, P/E) for a ticker."""
    return f"{ticker}: revenue +14% YoY, net margin 22%, P/E 28x."

# --- 3. Create a ReAct agent with a research-focused system prompt ---
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

analyst_agent = create_react_agent(
    model=llm,
    tools=[web_search, get_financials],
    prompt=(
        "You are an equity research analyst. Given a company, gather facts using "
        "your tools, then produce a brief with these sections: "
        "Business Overview, Recent Performance, Key Risks, and a one-line View. "
        "Only state facts supported by tool results."
    ),
)

# --- 4. Run it. The agent loops: reason -> search -> read -> ... -> write. ---
result = analyst_agent.invoke(
    {"messages": [("user", "Write a research brief on Nvidia (NVDA).")]}
)
print(result["messages"][-1].content)
```

**What happens under the hood (the ReAct loop from 7.1):**

1. The agent **reasons** it needs financials, calls `get_financials("NVDA")`, **observes** the metrics.
2. It reasons it needs recent news, calls `web_search("Nvidia recent results and outlook")`, observes the summary.
3. It may loop again for risks (e.g., `web_search("Nvidia key risks 2026")`).
4. When it has enough, it **stops** and writes the structured brief.

A sample output shape:

```
NVIDIA (NVDA) — Research Brief
Business Overview: Designs GPUs and AI accelerators used across data centers...
Recent Performance: Revenue up ~14% YoY, net margin ~22%, P/E ~28x...
Key Risks: Customer concentration, export restrictions, competition...
View: Strong AI demand tailwind; valuation leaves little room for error.
```

**Making it production-grade:** swap the placeholder tools for real APIs (Tavily for search, yfinance/Bloomberg for financials), add a **reflection** step where a critic agent checks the brief for unsupported claims, and add **human-in-the-loop** approval before the brief is sent to clients. These map directly onto the patterns in 7.7.

---

## 7.9 Framework Comparison: When to Pick Which

| Framework | Core idea | Control style | Best for | Watch out for |
|---|---|---|---|---|
| **LangGraph** | Agent as a state-machine graph | Explicit, low-level, precise | Production agents needing reliability, branching, checkpoints, human-in-loop | More boilerplate; steeper learning curve |
| **CrewAI** | Role-based crews of agents | Higher-level, org-chart style | Fast prototyping of collaborative multi-agent teams | Less fine-grained control over each step |
| **AutoGen** | Agents solve tasks by conversation | Conversational, flexible | Problem-solving via agent dialogue (e.g., coding + review) | Conversations can wander; needs good stop conditions |
| **smolagents** | Agents write & run Python code | Minimal, code-first | Compact code-agents; multi-step logic in one block | Executing model-written code needs sandboxing |

**Rules of thumb:**

- Need **tight control, reliability, and the ability to pause/resume** (finance, production)? → **LangGraph**.
- Want to **prototype a team of role-based agents quickly**? → **CrewAI**.
- Task benefits from **back-and-forth reasoning between experts**? → **AutoGen**.
- Want a **tiny, code-writing agent** with minimal abstraction? → **smolagents**.

You can also mix them: e.g., LangGraph for the top-level control flow, with MCP tools (Chapter 6) plugged in for capabilities.

---

## 7.10 Interview Q&A

**Q1. How do agents differ from chains?**

**A:** A chain follows a **fixed, predetermined sequence** of steps — same path every time. An agent uses the LLM to **decide at runtime** what to do next: which tool to call, whether to loop again, when to stop. Chains are predictable and easy to debug but rigid; agents are flexible and can handle open-ended tasks but are less predictable and harder to control. Use a chain when the steps are known in advance; use an agent when the path depends on intermediate results.

**Q2. What is the ReAct pattern?**

**A:** ReAct = **Reason + Act**. The agent alternates between reasoning ("Thought": what should I do?), acting ("Action": calling a tool), and observing ("Observation": the tool's result), looping until it has enough information to give a final answer. Interleaving reasoning with tool use makes agents far more capable and grounded than reasoning alone.

**Q3. How do you prevent an agent from getting stuck in an infinite loop?**

**A:** Several complementary guards: (1) a **maximum iteration / step limit** so the loop can't run forever; (2) clear **termination conditions** (e.g., a specific keyword like "APPROVE," or the model choosing to give a final answer); (3) a **time or token budget**; (4) **loop/repetition detection** to catch the agent calling the same tool with the same args repeatedly; and (5) **human-in-the-loop** checkpoints for long-running tasks. In practice you combine a hard step cap with a good stop condition.

**Q4. What is a tool schema and why does its quality matter so much?**

**A:** A tool schema describes a tool's name, purpose, and parameters (with types) to the model. Its quality directly determines whether the model calls the tool correctly — a clear description and well-named parameters lead to reliable, correct calls, while a vague schema causes the model to pick the wrong tool or pass bad arguments. Writing good docstrings and type hints is one of the highest-leverage things you can do for agent reliability.

**Q5. Explain single-agent vs multi-agent systems and when you'd use each.**

**A:** A single agent handles the whole task with one reasoning loop; it's simpler and cheaper and is often enough. A multi-agent system splits work among specialized agents (e.g., researcher + writer + critic), which improves quality on complex tasks through specialization and review, at the cost of more complexity, latency, and token usage. Start with a single agent; move to multi-agent when the task clearly benefits from distinct roles or a review step.

**Q6. What are parallel tool calls and why do they matter?**

**A:** When a task requires several *independent* tool calls, modern models can request them all at once so they execute in parallel rather than sequentially. This cuts latency significantly — e.g., fetching prices for five stocks in one round instead of five. It matters most for responsiveness in user-facing apps like dashboards and chat.

**Q7. Compare LangGraph and CrewAI.**

**A:** LangGraph is a low-level, graph-based framework giving explicit control over state, nodes, edges, and checkpoints — ideal for reliable production agents that need branching and human-in-the-loop. CrewAI is higher-level and models agents as role-based crews collaborating on tasks — ideal for quickly prototyping multi-agent teams. LangGraph trades more boilerplate for control; CrewAI trades control for speed of development.

**Q8. What is human-in-the-loop and when is it essential?**

**A:** Human-in-the-loop means the agent pauses for human approval before taking a high-stakes or irreversible action. It's essential wherever mistakes are costly — executing trades or wire transfers in finance, deleting data, sending communications to customers, or any regulated action. Frameworks like LangGraph support it via checkpointing, which lets an agent pause, wait for approval, and resume.

**Q9. How would you make an agent's answers more trustworthy and reduce hallucination?**

**A:** Ground the agent in tools and retrieval so it uses real data rather than guessing; instruct it to only state facts supported by tool results; require citations/sources; add a reflection or critic step to catch unsupported claims; use temperature 0 for factual tasks; and add human review for high-stakes outputs. In finance, source citations and a critic pass are especially valuable.

**Q10. When would you NOT use an agent at all?**

**A:** When the task has a known, fixed sequence of steps with no need for runtime decisions — use a chain or a simple pipeline instead, since it's cheaper, faster, and more predictable. Agents add cost, latency, and unpredictability, so reserve them for genuinely open-ended tasks where the path can't be hard-coded in advance.

---

## Key Takeaways

- An **agent** decides its own next step (tool, loop, stop) at runtime, unlike a **chain**, which follows a fixed path.
- The **ReAct loop** (Reason → Act → Observe, repeat) is the core agent pattern; always cap it with a step limit and clear stop condition.
- **LangGraph** models agents as state-machine **graphs** (state, nodes, edges, checkpoints) for precise, resumable, production-grade control.
- **CrewAI** builds **role-based crews** (e.g., researcher + writer); **AutoGen** solves tasks via agent **conversation**; **smolagents** writes and runs **Python code** as its actions.
- **Tool calling** works by the model *requesting* a call (name + args) while *your code* runs it; good **schemas/docstrings** are critical, and **parallel calls** cut latency.
- Key **design patterns**: router, planner-executor, reflection, and human-in-the-loop — the last is mandatory for high-stakes finance actions.
- An **equity-research agent** combines search tools, the ReAct loop, and reflection to produce grounded, cited briefs.
- **Prevent infinite loops** with step caps, termination conditions, budgets, loop detection, and human checkpoints — a favorite interview topic.
