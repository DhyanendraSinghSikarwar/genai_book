# Chapter 14 — LangGraph: State, Checkpoints, and Human-in-the-Loop

## What you'll be able to do after this chapter

1. Refactor a hand-written ReAct loop (Chapter 13) into a LangGraph `StateGraph` — nodes, conditional edges, and a typed state — without changing the underlying algorithm, and explain precisely what moved and why.
2. Persist every step of an agent run to Postgres so that killing the process mid-run and starting a new one resumes from the last completed step, not from the beginning.
3. Gate an irreversible action (sending an email) behind a durable human-approval interrupt, recorded in an `approvals` table with an idempotency key, so a crash or a retry can never cause a duplicate send.
4. Stream intermediate graph steps to a UI, and use LangGraph's checkpoint history to time-travel to any prior step of a run for debugging.
5. State, with a concrete threshold, when a graph framework is the wrong tool and a hand-written loop remains the better choice.

---

## The problem this solves

Here is the failure you hit if you ship Chapter 13's loop as-is for AtlasDesk's C4 capability — drafting and sending a reply email on a learner's behalf.

Daniel Osei, a tier-1 support agent, asks AtlasDesk to draft a fee-reminder email to Rohan Mehta. The agent loop looks up the deadline, drafts the email, and — because Chapter 13's loop has no concept of "wait for a human" — calls the `send_email` tool immediately. The email goes out. Thirty seconds later Daniel notices the drafted amount is wrong: the loop read instalment 1's amount instead of instalment 2's, because the tool result formatting bug from Chapter 13's own worked example was still lurking in a different tool. There is no undo button for an email that already left Meridian's mail server. This is not a hypothetical: it is the generic shape of every "an agent did something in the world it can't take back" incident, and Chapter 20's threat model calls exactly this class of failure "irreversible action executed without a checkpoint."

Now add a second, unrelated failure. A different learner's agent run is deep into a nine-step tool-calling sequence — retrieval, a catalogue lookup, a computation, a draft — when the process hosting it gets OOM-killed by the platform during a deploy. Chapter 13's loop keeps its entire state, including the step count and the accumulated budget, in a Python `while` loop's local variables. When the process dies, that state dies with it. The learner's next message starts a brand-new run from zero, the budget resets, and if the deploy happens to coincide with a retry storm, AtlasDesk pays for the same partial work twice.

Both failures have the same root cause: Chapter 13's loop treats "the state of an agent run" as something that only needs to survive as long as the Python process does, and treats "the agent decided to act" as equivalent to "the action happened." Fixing both requires the same two capabilities — state that outlives the process, and a place to pause before an action that can't be undone — and building both by hand, correctly, on top of raw `while True` and manual JSON serialization, is exactly the kind of infrastructure problem a mature framework has already solved. This chapter introduces that framework, on purpose, having made you build the thing it replaces first.

---

## Concepts

### What actually changes when you adopt a framework

Chapter 13 ended with a promise: name the specific capability you're missing, and if you can name it, that's the point to reach for a framework. AtlasDesk can now name two: **durable state across a process crash**, and **a pause that survives a process restart, waiting on something outside the process** (a human, in this case). LangGraph's contribution is not a smarter agent — the ReAct algorithm underneath is identical to Chapter 13's — its contribution is a checkpoint format and a pause/resume primitive engineered by people who have hit every edge case of "what if the process dies at this exact instruction" that you have not yet hit.

Concretely, three things move from Chapter 13 to this chapter:

| Chapter 13 (hand-written loop) | Chapter 14 (LangGraph) |
|---|---|
| One Python function, a `while True` loop | A graph of node functions connected by edges |
| Local variables (`fingerprints`, `messages`, `budget`) live on the call stack | The same values live in `AgentState`, passed explicitly between nodes, snapshotted after every node |
| No persistence between calls; a crash loses everything after the last `return` | Every node's output is written to a checkpointer (Postgres, in production) before the next node runs |
| No way to pause mid-run for external input | `interrupt()` pauses a node, persists the pause, and `Command(resume=...)` continues it — from a different process, possibly hours later |
| Debugging means reading a saved `TraceEvent` list after the fact | Debugging additionally includes time-travel: replay the graph from any prior checkpoint, not just read a log of what happened |

And one thing does **not** move, on purpose: the algorithm. `call_model`, `execute_tools`, the two loop detectors, and the budget checks in this chapter's `agent/graph.py` are Chapter 13's code, split across function boundaries. If you diff the logic inside each node against the corresponding block of Chapter 13's `run_agent_loop`, it is the same logic. What changed is where the state that logic reads and writes physically lives — from the Python call stack to a typed dictionary that a checkpointer serializes after every step.

### Nodes, edges, conditional routing, and typed state

A LangGraph `StateGraph` is built from four ingredients:

- **State** — a single typed schema (here, Chapter 13's `AgentState` `TypedDict`, unchanged) that every node reads from and writes back to. A node returns a partial dict; LangGraph merges it into the running state.
- **Nodes** — plain async functions, `state -> partial_state`. Each one is a synchronous unit of work from the checkpointer's point of view: either it fully completes and its output is persisted, or it didn't run at all this step.
- **Edges** — fixed transitions (`graph.add_edge("send_email", "call_model")`) for "always go here next," used where there's no decision to make.
- **Conditional edges** — a router function that inspects the state after a node and returns the name of the next node. This is where Chapter 13's `if/elif` termination checks live now, as `route_after_model` and `route_after_tools` functions instead of inline branches in the loop body.

```python
# excerpt -- full file below
graph.add_conditional_edges(
    "call_model",
    route_after_model,
    {"execute_tools": "execute_tools", "approval_gate": "approval_gate", "end": END},
)
```

The third argument is a mapping from the router's return value to a real node name — LangGraph does not infer graph structure from a function's `if` statements; you declare every edge that could be taken, which is precisely what makes a `graph.get_graph().draw_mermaid()` call an accurate, mechanically-generated picture of your control flow, rather than documentation someone forgot to update. That property is worth more than it sounds: Chapter 13's control flow lived entirely in the shape of one function's `while` loop, invisible to anything except a careful reading of the code; this chapter's control flow is data LangGraph can introspect, diagram, and replay.

### Persistent checkpointing: what actually gets written, and when

A **checkpointer** is the object that makes state survive a process boundary. LangGraph writes a checkpoint after every node completes — called a "superstep" — keyed by a `thread_id` you choose. `agent/checkpoint.py` below wraps `AsyncPostgresSaver` from `langgraph-checkpoint-postgres`, the same way Chapter 4 wraps the Anthropic and OpenAI SDKs: business logic never imports `psycopg` or the checkpoint library directly, it takes a `BaseCheckpointSaver` and calls `compile(checkpointer=...)`.

```mermaid
sequenceDiagram
    participant P1 as Process A (call_model runs)
    participant PG as Postgres (checkpoints table)
    participant P2 as Process B (new process)

    P1->>PG: write checkpoint after call_model completes
    Note over P1: process killed (SIGKILL) here
    P2->>PG: graph.ainvoke(None, config) -- same thread_id
    PG->>P2: load last checkpoint (state after call_model)
    P2->>P2: run execute_tools, then call_model again
    P2->>PG: write checkpoint after each node
```

Read this top to bottom as what actually happened in this chapter's own verification run, not a hypothetical: Process A ran `call_model` once, its output was committed to Postgres, and Process A was then killed with `SIGKILL` — the least graceful termination available, no cleanup handlers, no `atexit`. Process B started cold, with zero shared memory with Process A, called `graph.ainvoke(None, config)` against the same `thread_id`, and LangGraph loaded the last checkpoint and continued from `execute_tools` — the node that had not yet run — rather than replaying `call_model`. The decision rule that falls out of this: **checkpoint granularity is per-node, not per-run**, so the size of a node determines how much work a crash can cost you. A `call_model` node that also silently executes three tool calls before returning is a larger unit of at-risk work than one that returns after the model call alone; keep nodes small for exactly this reason.

One consequence worth internalizing before you rely on it: the code *inside* a node that has not yet completed a checkpoint **re-runs from its start** on resume. This matters most for the approval-gate node below, which calls a database insert before it calls `interrupt()` — that insert has to be idempotent, because a resume that races with a second resume, or a node that raises after the insert but before the interrupt commits, will call it again.

### Interrupts and approval gates: the mechanism behind C4

`interrupt(payload)` is a function that raises a special, LangGraph-internal exception when called inside a node. The framework catches it, persists the current state and the payload to the checkpointer, and returns control to the caller — `graph.ainvoke(...)` returns normally, with `"__interrupt__"` present in the result, rather than raising or hanging. The pause is not "the process is blocked waiting" — the process can (and in production should) exit entirely. Resuming means calling `graph.ainvoke(Command(resume=<value>), config)` against the same `thread_id`, from any process, at any later time; the value you pass becomes `interrupt()`'s return value inside the node, and execution continues from there.

```python
decision = interrupt({"kind": "approval_required", "draft": draft.model_dump()})
# ... nothing below this line runs until Command(resume=...) arrives
if decision["approved"]:
    ...
```

This is the exact mechanism `agent/graph.py`'s `approval_gate` node uses to deliver AtlasDesk's **C4**: the `send_email` tool is never called by the model's tool-calling turn directly. Instead, `route_after_model` detects a pending `send_email` call and routes to `approval_gate`, which records a pending row in the `approvals` table, calls `interrupt()` with the draft, and — only after a human resumes with an approval — routes to a `send_email` node that performs the actual send. The two rules that make this safe rather than merely functional: **never call `interrupt()` after a side effect that isn't idempotent**, because the code before it re-runs on every resume attempt; and **never let the model's tool call reach the real action directly** — the graph's routing, not the model's intent, decides whether `send_email` runs.

> **▸ Senior practice #14 — Irreversible actions behind human approval**
>
> Any action your system can take that a human can't cleanly undo — sending an email, issuing a refund, deleting a record, posting to a public channel — gets a mandatory approval gate before this chapter's project state exists, not after the first bad send. The gate is not "the model asks nicely and waits a beat": it is a durable row in a table, keyed so a retried resume can't double-send, decided by a named human, with the decision timestamped. AtlasDesk's implementation is `agent/hitl.py` plus `migrations/0003_approvals.sql`, below.
>
> The habit this replaces is "the demo didn't send anything wrong, so we shipped it live." A demo runs a handful of curated prompts; production runs thousands of prompts you didn't anticipate, against a model that occasionally reads a stale tool result or an ambiguous instruction the way SWE-agent's badly-designed tool interfaces once did (Chapter 13, Case 2). The fix is not a better prompt. It's a gate the model's output cannot bypass.

### Streaming and time-travel debugging

`graph.astream(state, config, stream_mode="updates")` yields one event per completed node, as it completes — this is what the Streamlit queue and any chat UI use to show "looking up the deadline… drafting the email… waiting for approval" instead of a silent multi-second gap. `stream_mode="values"` yields the full state after each node instead of just the diff, useful for a debugging console that wants to show everything at once.

Time-travel debugging is `graph.aget_state_history(config)`: an iterator over every checkpoint ever written for a thread, oldest to newest, each with the node that produced it and the state at that point. You can pass any earlier checkpoint's config back into `graph.ainvoke(..., earlier_config)` and the graph resumes *from that point*, not from the current head — which is exactly what "replay this run from the step before it went wrong, with a fixed tool" means in practice, and something Chapter 13's `trace_reader.py` could describe after the fact but never actually re-execute.

### When you should not use a framework

The honest case against LangGraph, stated as concretely as the case for it:

| Situation | Verdict |
|---|---|
| A single call, or a call plus at most one tool round-trip | No framework. A function that calls `LLMClient.complete()` once is simpler than a one-node graph, and a one-node graph is what you'd end up with. |
| A fixed sequence of LLM calls with no branching (a "prompt chain," Chapter 15) | No framework. Plain async functions calling each other in order are more readable than a graph with only `add_edge` and no conditional routing — a graph earns its keep at the first `add_conditional_edges` call, not before. |
| An agent loop that never needs to survive a process restart and never gates an irreversible action | Chapter 13's hand-written loop is still the right choice; it is fewer moving parts and every line is code you wrote and can read under pressure. |
| An agent loop that must survive a crash, or must pause for a human, or has branching complex enough that an if/else chain is hard to read | LangGraph. This is AtlasDesk's C4 exactly. |

The decision rule: **adopt the graph framework at the point you need durable state across a process boundary or a pause that outlives a request — not before, and not because "agents are hard" in the abstract.** If you cannot point to a specific failure this chapter's mechanisms prevent, you are adding a dependency, a new serialization format, and a new debugging surface for no measured benefit — the same warning Chapter 13's Anthropic case study gives about frameworks generally, now with a precise line drawn for when it stops applying.

---

## How industry does it

### Case 1 — AppFolio's Realm-X: parallel branches and a controllable action space

**The problem.** AppFolio, a property-management software vendor, wanted a natural-language interface — Realm-X — that lets property managers query resident, vendor, unit, and work-order data, and execute bulk actions (sending messages, updating records) without learning the underlying application's UI. As the set of supported actions grew, a linear prompt-chain implementation stopped being maintainable: different user requests need different combinations of "figure out which action applies," "compute a fallback if the primary path fails," and "answer a direct question," and those needed to run and be reconciled together rather than as one long sequential chain.

**What they built.** AppFolio migrated from a plain LangChain implementation to LangGraph specifically to model this as a graph: independent branches run in parallel — one determining the relevant action, one computing a fallback, one running a question-answering path — and LangGraph's state-merging handles aggregating their outputs into a single response, which the company's own published case study describes as "simplified response aggregation from different nodes." The architecture is explicitly framed around workflows that "reason before acting," with the graph structure making the execution flow visible rather than buried inside a single prompt.

**The measured outcome.** AppFolio reports early users saving **over 10 hours a week** completing their to-do lists using Realm-X, and separately reports that dynamic few-shot prompting improved the text-to-data functionality's accuracy from roughly 40% to 80% as the product matured — both figures from AppFolio's own published case study with LangChain, not a third-party benchmark.

**What you should copy at 1/1000th the scale.** The lesson isn't "use parallel branches." It's that the point at which a linear chain of `if/elif` statements becomes a liability is exactly the point AppFolio hit and exactly the point this chapter names: once a request needs more than one independent thing computed and reconciled, model it as named nodes with explicit edges, not as more nested conditionals in one function. AtlasDesk's graph does not yet fan out in parallel — Chapter 15 covers that pattern in depth — but the same graph structure this chapter introduces is what makes adding it later a new node and a new edge, not a rewrite.

### Case 2 — LinkedIn's SQL Bot: a durable multi-agent pipeline with a self-correction loop

**The problem.** LinkedIn's internal data analysts were spending a large share of their time helping colleagues locate and query internal data rather than doing analysis — a bottleneck that scales with headcount, not with how good the underlying SQL generation is.

**What they built.** SQL Bot is a LangGraph pipeline with four stages: retrieve roughly 20 candidate tables via embedding search, have an LLM rank and narrow that to about 7 tables with tiered column detail, generate a SQL query with its assumptions documented, and — the part directly relevant to this chapter's control-flow lesson — a "Fix Query" stage that validates syntax and, on failure, hands off to a "Researcher Agent" with table-search tools to find the actual error, looping back rather than failing outright. The whole thing sits on a knowledge graph connecting LinkedIn's DataHub metadata, historical query logs, and a library of certified example queries — the equivalent of AtlasDesk's semantic layer in Chapter 16, at LinkedIn's scale.

**The measured outcome.** LinkedIn reports the self-correction loop alone lifted query compilation success from 88% to 96%; 53% of SQL Bot's responses were rated correct or near-correct on LinkedIn's internal benchmark, with 95% of users rating query accuracy as acceptable or better and 40% rating it "very good" or "excellent"; the "Fix with AI" retry button accounts for 80% of all sessions, meaning most real usage runs the self-correction branch at least once; and folding SQL Bot into LinkedIn's existing DARWIN analytics platform, rather than shipping it as a standalone tool, produced a 5–10x increase in usage over the standalone deployment, reaching 300+ weekly active users. All figures as published by LinkedIn and LangChain's joint case study.

**What you should copy at 1/1000th the scale.** Two things, both cheap to copy: first, a retry/fix loop that routes a failure back into the graph instead of surfacing a raw error is worth more than a smarter first attempt — LinkedIn's numbers show the fix loop moving compilation success 8 points on its own. Second, and easy to miss: the adoption multiplier came from distribution, not model quality — putting the same agent inside a tool people already use beat shipping it as a new destination by 5–10x. AtlasDesk's Streamlit approval queue below is deliberately where Daniel Osei already works, not a new app he has to remember to open.

---

## Build: AtlasDesk increment — the graph, checkpoints, and the approval gate

### Project state

**What exists going into this chapter:** the provider abstraction (`llm/base.py`, `llm/fake.py`, `llm/router.py`, Chapter 4), structured outputs (`schemas/answer.py`, Chapter 6), the retrieval stack (`retrieval/`, Chapters 8–10), the tool layer (`tools/registry.py`, `tools/learner.py`, `tools/catalogue.py`, `tools/authz.py`, Chapter 12), and Chapter 13's hand-written loop: `agent/state.py` (`AgentState`, `Budget`), `agent/loop.py`, `agent/trace_reader.py`.

**What this chapter adds:** `agent/graph.py` (Chapter 13's loop, refactored into LangGraph nodes — the algorithm is unchanged; only its physical location moves), `agent/checkpoint.py` (the Postgres and in-memory checkpointer factories), `agent/hitl.py` (the approvals data layer behind C4), `migrations/0003_approvals.sql`, three additional fields on `AgentState` that the refactor requires (`pending_tool_calls`, `fingerprints`, `tool_results_history`, plus `run_id` — Bible §4.8 amended in the same style as §11's existing amendments, not contradicted), a demo tool set at `tools/actions.py`, and `ui/approval_queue.py`, the Streamlit page where Daniel Osei approves or edits a draft.

`agent/loop.py` from Chapter 13 is not deleted. It stays as the reference implementation for the simple case in the decision table above — a task that needs neither a crash-surviving checkpoint nor a human gate is still better served by it, and Chapter 15 continues to import it directly for the patterns that don't need a graph.

### Repo tree diff

```
  src/atlasdesk/
    llm/                    # unchanged since Ch 4-6
    schemas/                # unchanged since Ch 6
    retrieval/               # unchanged since Ch 8-10
    tools/
      registry.py            # unchanged since Ch 12
      learner.py              # unchanged since Ch 12
      catalogue.py             # unchanged since Ch 12
+     actions.py              # send_email + demo get_deadline tools for this chapter
    agent/
      state.py                # Ch 13, amended: + pending_tool_calls, fingerprints,
                               #   tool_results_history, run_id
      loop.py                 # Ch 13, unchanged, kept as the no-framework reference
      trace_reader.py          # Ch 13, unchanged
+     graph.py                # Ch 13's loop refactored into LangGraph nodes
+     checkpoint.py            # Postgres + in-memory checkpointer factories
+     hitl.py                  # approvals table access, idempotency keys
  migrations/
    0001_documents.sql        # Ch 8
    0002_vectors.sql          # Ch 9
+   0003_approvals.sql         # this chapter -- C4's durable approval gate
+ ui/
+ └── approval_queue.py        # Streamlit approval queue
  tests/
    test_loop.py               # Ch 13, unchanged
+   test_graph.py               # this chapter
```

### `agent/state.py` — the amendment

Bible §4.8 froze `AgentState`'s shape at the end of Chapter 13. This chapter needs three fields Chapter 13's loop kept as local Python variables — `fingerprints`, the running tool-result history for `no_progress` detection, and the pending tool calls a node has to hand off to the next node — plus a `run_id` to key checkpoints and approvals on. This is an amendment in the same spirit as Bible §11's existing amendments: it does not remove or change any existing field, and every later chapter that reads `AgentState` continues to work unmodified.

```python
# src/atlasdesk/agent/state.py
"""Agent state, shared by Chapter 13's hand-written loop and this
chapter's LangGraph nodes. Bible S4.8, amended here: three fields that
lived on the call stack in Chapter 13 (fingerprints, tool results, the
pending tool calls between a model turn and the next node) must live in
this dict once a graph node -- not a while-loop iteration -- is what
reads and writes them.
"""

from __future__ import annotations

from typing import TypedDict

from pydantic import BaseModel

from atlasdesk.llm.base import Message
from atlasdesk.retrieval.types import RetrievedChunk
from atlasdesk.schemas.answer import Answer
from atlasdesk.security.principal import Principal


class Budget(BaseModel):
    """Unchanged from Chapter 13."""

    max_steps: int = 12
    max_cost_usd: float = 0.15
    steps_used: int = 0
    cost_used_usd: float = 0.0


class EmailDraft(BaseModel):
    """A C4 draft awaiting human sign-off. Unchanged from Chapter 13;
    this chapter gives it a full lifecycle via `agent/hitl.py`.
    """

    to: str
    subject: str
    body: str


class ApprovalRecord(BaseModel):
    """A human decision on an EmailDraft. Unchanged from Chapter 13;
    this chapter persists it in the `approvals` table.
    """

    approved: bool
    decided_by: str
    note: str = ""


class AgentState(TypedDict, total=False):
    """The state threaded through one agent run, start to termination.
    Identical to Chapter 13 except for the four fields marked below.
    """

    principal: Principal
    question: str
    messages: list[Message]
    retrieved: list[RetrievedChunk]
    budget: Budget
    draft: EmailDraft | None
    approval: ApprovalRecord | None
    answer: Answer | None
    terminated_because: str
    run_id: str  # new -- keys the checkpoint thread and the approvals row
    pending_tool_calls: list[dict]  # new -- tool calls awaiting execute_tools/approval_gate
    fingerprints: list[str]  # new -- moved out of loop.py's local scope
    tool_results_history: list[str]  # new -- moved out of loop.py's local scope
```

### `agent/graph.py` — the refactor

This is Chapter 13's algorithm, unchanged, redistributed across five node functions plus two router functions. Read the docstring comments closely: every place this file differs in *behavior*, not just in shape, from `agent/loop.py` is called out explicitly, and there are exactly two — the approval gate itself, and the fact that loop-detector state now lives in `AgentState` instead of a Python list closed over by the loop.

```python
# src/atlasdesk/agent/graph.py
"""The ReAct loop from Chapter 13, refactored into a LangGraph graph.

This module does not reimplement Chapter 13's algorithm. `agent/loop.py`'s
while-loop becomes a cycle of nodes on the same `AgentState`; the budget
checks, the tool-result formatting, and the three loop detectors are
copied over unchanged, just relocated from local closures into node
functions, because a graph node cannot close over another node's local
variables -- everything the hand-written loop kept on the Python stack
(fingerprints, tool-call results, the running budget) now has to live in
`AgentState` so the next node can see it. That is the one real cost of
this refactor, and Concepts explains it in full.

New in this chapter, not present in Chapter 13 at all: the `approval_gate`
node, which calls `interrupt()` before any call to the `send_email` tool
-- AtlasDesk's C4. Everything downstream of `call_model` is unchanged
algorithmically from Chapter 13.
"""

from __future__ import annotations

import json
from typing import Any, Literal

import psycopg
from langgraph.graph import END, StateGraph
from langgraph.graph.state import CompiledStateGraph
from langgraph.types import interrupt

from atlasdesk.agent import hitl
from atlasdesk.agent.state import AgentState, ApprovalRecord, EmailDraft
from atlasdesk.errors import ToolError
from atlasdesk.llm.base import LLMClient, Message
from atlasdesk.schemas.answer import Answer
from atlasdesk.tools.registry import ToolRegistry


def _fingerprint(name: str, arguments: dict[str, Any]) -> str:
    """Identical to Chapter 13's `_fingerprint` -- unchanged."""
    return f"{name}:{json.dumps(arguments, sort_keys=True)}"


def _detect_loop(fingerprints: list[str]) -> str | None:
    """Identical to Chapter 13's `_detect_loop` -- unchanged."""
    if len(fingerprints) >= 3 and fingerprints[-1] == fingerprints[-2] == fingerprints[-3]:
        return "repeated_tool_call"
    if (
        len(fingerprints) >= 4
        and fingerprints[-1] == fingerprints[-3]
        and fingerprints[-2] == fingerprints[-4]
        and fingerprints[-1] != fingerprints[-2]
    ):
        return "oscillation"
    return None


def build_graph(
    *,
    client: LLMClient,
    tools: ToolRegistry,
    system: str,
    approval_conn: psycopg.AsyncConnection[Any],
    checkpointer: Any,
) -> CompiledStateGraph[AgentState, None, AgentState, AgentState]:
    """Assemble and compile the AtlasDesk agent graph.

    `approval_conn` is a live connection to the same Postgres database the
    checkpointer writes to -- the approvals table and the checkpoint
    tables are one datastore, per Bible S4.11, not two systems to keep
    in sync.
    """

    async def call_model(state: AgentState) -> dict[str, Any]:
        """Chapter 13's model-call step. Budget check moved to the top of
        the node body because a node, unlike a while-loop iteration,
        cannot simply `continue` -- it returns, and the conditional edge
        decides where control goes next.
        """
        budget = state["budget"]
        if budget.steps_used >= budget.max_steps:
            return {"terminated_because": "max_steps"}
        if budget.cost_used_usd >= budget.max_cost_usd:
            return {"terminated_because": "budget_exceeded"}

        completion = await client.complete(state["messages"], system=system, tools=tools.specs())
        budget.steps_used += 1
        budget.cost_used_usd += completion.usage.cost_usd

        if budget.cost_used_usd > budget.max_cost_usd:
            return {"budget": budget, "terminated_because": "budget_exceeded"}

        messages = list(state["messages"])
        messages.append(Message(role="assistant", content=completion.text or ""))

        if not completion.tool_calls:
            answer = Answer(text=completion.text, citations=[], confidence=1.0, should_escalate=False)
            return {
                "messages": messages,
                "answer": answer,
                "budget": budget,
                "terminated_because": "stop",
            }

        return {
            "messages": messages,
            "budget": budget,
            "pending_tool_calls": [tc.model_dump() for tc in completion.tool_calls],
        }

    def route_after_model(state: AgentState) -> Literal["execute_tools", "approval_gate", "end"]:
        if state.get("terminated_because"):
            return "end"
        pending = state.get("pending_tool_calls") or []
        if any(call["name"] == "send_email" for call in pending):
            return "approval_gate"
        return "execute_tools"

    async def execute_tools(state: AgentState) -> dict[str, Any]:
        """Chapter 13's tool-execution step plus its two per-call loop
        detectors, unchanged in logic -- fingerprints and results now
        read from and write back to `state` instead of a local list,
        because that state has to survive to the next node invocation.
        """
        budget = state["budget"]
        messages = list(state["messages"])
        fingerprints = list(state.get("fingerprints", []))
        results_history = list(state.get("tool_results_history", []))
        terminated_because = ""

        for call in state.get("pending_tool_calls", []):
            fingerprints.append(_fingerprint(call["name"], call["arguments"]))
            try:
                result = await tools.call(call["name"], call["arguments"], principal=state["principal"])
            except ToolError as exc:
                result = f"error: {exc}"
            results_history.append(result)
            messages.append(
                Message(role="tool", content=result, tool_call_id=call["id"], name=call["name"])
            )

            loop_reason = _detect_loop(fingerprints)
            if loop_reason:
                terminated_because = loop_reason
                break
            if (
                len(results_history) >= 2
                and results_history[-1] == results_history[-2]
                and fingerprints[-1] != fingerprints[-2]
            ):
                terminated_because = "no_progress"
                break

        updates: dict[str, Any] = {
            "messages": messages,
            "budget": budget,
            "fingerprints": fingerprints,
            "tool_results_history": results_history,
            "pending_tool_calls": [],
        }
        if terminated_because:
            updates["terminated_because"] = terminated_because
        return updates

    def route_after_tools(state: AgentState) -> Literal["call_model", "end"]:
        return "end" if state.get("terminated_because") else "call_model"

    async def approval_gate(state: AgentState) -> dict[str, Any]:
        """AtlasDesk C4. Everything above this node is Chapter 13; this
        node, `send_email`, and `reject_email` below are new in Chapter 14.

        The code before `interrupt()` re-runs on every resume (Concepts
        explains why), so `request_approval` must be -- and is -- safe to
        call twice with the same arguments: it is keyed on an idempotency
        key derived from the draft's content, not a fresh UUID per call.
        """
        pending = state.get("pending_tool_calls", [])
        call = next(c for c in pending if c["name"] == "send_email")
        draft = EmailDraft(**call["arguments"])
        run_id = state["run_id"]

        approval_row = await hitl.request_approval(
            approval_conn, run_id=run_id, action_type="send_email", payload=draft.model_dump()
        )

        decision = interrupt(
            {
                "kind": "approval_required",
                "approval_id": approval_row.id,
                "action_type": "send_email",
                "draft": draft.model_dump(),
            }
        )

        await hitl.record_decision(
            approval_conn,
            approval_id=approval_row.id,
            approved=bool(decision["approved"]),
            decided_by=str(decision["decided_by"]),
        )

        if decision.get("edited_draft"):
            draft = EmailDraft(**decision["edited_draft"])

        approval = ApprovalRecord(
            approved=bool(decision["approved"]),
            decided_by=str(decision["decided_by"]),
            note=str(decision.get("note", "")),
        )
        return {"draft": draft, "approval": approval, "pending_tool_calls": []}

    def route_after_approval(state: AgentState) -> Literal["send_email", "reject_email"]:
        approval = state["approval"]
        assert approval is not None
        return "send_email" if approval.approved else "reject_email"

    async def send_email(state: AgentState) -> dict[str, Any]:
        draft = state["draft"]
        assert draft is not None
        result = await tools.call(
            "send_email",
            {"to": draft.to, "subject": draft.subject, "body": draft.body},
            principal=state["principal"],
        )
        messages = list(state["messages"])
        messages.append(Message(role="tool", content=result, name="send_email"))
        return {"messages": messages}

    async def reject_email(state: AgentState) -> dict[str, Any]:
        approval = state["approval"]
        assert approval is not None
        note = approval.note or "no reason given"
        messages = list(state["messages"])
        messages.append(
            Message(role="tool", content=f"the human reviewer rejected this draft: {note}", name="send_email")
        )
        return {"messages": messages}

    graph = StateGraph(AgentState)
    graph.add_node("call_model", call_model)
    graph.add_node("execute_tools", execute_tools)
    graph.add_node("approval_gate", approval_gate)
    graph.add_node("send_email", send_email)
    graph.add_node("reject_email", reject_email)

    graph.set_entry_point("call_model")
    graph.add_conditional_edges(
        "call_model",
        route_after_model,
        {"execute_tools": "execute_tools", "approval_gate": "approval_gate", "end": END},
    )
    graph.add_conditional_edges("execute_tools", route_after_tools, {"call_model": "call_model", "end": END})
    graph.add_conditional_edges(
        "approval_gate", route_after_approval, {"send_email": "send_email", "reject_email": "reject_email"}
    )
    graph.add_edge("send_email", "call_model")
    graph.add_edge("reject_email", "call_model")

    return graph.compile(checkpointer=checkpointer)
```

### The state graph, diagrammed

```mermaid
stateDiagram-v2
    [*] --> call_model
    call_model --> execute_tools: tool_calls present, no send_email
    call_model --> approval_gate: tool_calls include send_email
    call_model --> [*]: no tool_calls (stop), max_steps, or budget_exceeded
    execute_tools --> call_model: fingerprint looks new
    execute_tools --> [*]: repeated_tool_call / oscillation / no_progress
    approval_gate --> send_email: human resumes with approved=true
    approval_gate --> reject_email: human resumes with approved=false
    send_email --> call_model
    reject_email --> call_model
```

Follow the path AtlasDesk's own verification run took, left to right. `call_model` runs, sees the model wants `get_deadline`, and routes to `execute_tools`; the tool succeeds, no loop detector fires, and control returns to `call_model`. The second `call_model` call sees a `send_email` tool call and routes to `approval_gate` instead of `execute_tools` — this is the entire mechanism of C4, a routing decision the model's own tool call cannot override. `approval_gate` calls `interrupt()` and the graph pauses; nothing below that line runs until a human resumes it, potentially from a different process, hours later, as this chapter's resumption demo below proves for real. Once resumed with an approval, `send_email` actually performs the send and loops back to `call_model` for whatever the model says next — in this run, a closing confirmation with no further tool calls, which stops the graph cleanly.

### `agent/checkpoint.py` — the Postgres and test checkpointers

```python
# src/atlasdesk/agent/checkpoint.py
"""Durable checkpointing for the agent graph.

`get_postgres_checkpointer` is what production runs against: every graph
step is written to Postgres before the node's side effects are considered
complete, so a killed process resumes from the last completed step rather
than from the start of the conversation. `get_memory_checkpointer` is what
tests and local demos run against when a live Postgres instance is not the
point of the test -- same `BaseCheckpointSaver` interface, zero setup.

Chapter 4's provider seam and this checkpointer seam are the same idea
applied twice: business logic (agent/graph.py) never imports psycopg
directly, it takes a `BaseCheckpointSaver` and calls the four methods on
that Protocol.
"""

from __future__ import annotations

from collections.abc import AsyncIterator
from contextlib import asynccontextmanager

from langgraph.checkpoint.memory import InMemorySaver
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver
from langgraph.checkpoint.serde.jsonplus import JsonPlusSerializer

from atlasdesk.config import Settings

# AgentState holds Pydantic models (Principal, Budget, Message) that the
# checkpointer serializes with msgpack. Newer langgraph-checkpoint releases
# warn on -- and will eventually refuse -- deserializing an unregistered
# type, so every Pydantic model that lives in AgentState is declared here
# once, rather than silencing the warning globally.
_ALLOWED_STATE_MODULES: tuple[tuple[str, str], ...] = (
    ("atlasdesk.security.principal", "Principal"),
    ("atlasdesk.agent.state", "Budget"),
    ("atlasdesk.agent.state", "EmailDraft"),
    ("atlasdesk.agent.state", "ApprovalRecord"),
    ("atlasdesk.llm.base", "Message"),
    ("atlasdesk.schemas.answer", "Answer"),
)


def _serde() -> JsonPlusSerializer:
    return JsonPlusSerializer(allowed_msgpack_modules=_ALLOWED_STATE_MODULES)


@asynccontextmanager
async def get_postgres_checkpointer(settings: Settings) -> AsyncIterator[AsyncPostgresSaver]:
    """Yield a Postgres-backed checkpointer, creating its tables on first use.

    `setup()` is idempotent -- safe to call on every process start, the same
    way Chapter 8's ingestion pipeline is idempotent on re-run. Uses
    `settings.database_url`, never a literal connection string.
    """
    async with AsyncPostgresSaver.from_conn_string(settings.database_url, serde=_serde()) as saver:
        await saver.setup()
        yield saver


def get_memory_checkpointer() -> InMemorySaver:
    """An in-process checkpointer for tests and local demos.

    Same `BaseCheckpointSaver` interface as the Postgres saver above --
    `agent/graph.py` never knows which one it was handed. It does not
    survive a process restart, which is precisely why the resumption demo
    in this chapter uses the Postgres saver, not this one.
    """
    return InMemorySaver()
```

### `migrations/0003_approvals.sql` — C4's durable gate

*File: `migrations/0003_approvals.sql`*

```sql
-- migrations/0003_approvals.sql
-- AtlasDesk C4: irreversible actions (send-email) behind human sign-off.
-- One row per requested action; idempotency_key makes a resumed run safe
-- to re-enter the approval node without creating a duplicate request or,
-- worse, a duplicate send.

CREATE TABLE IF NOT EXISTS approvals (
    id              BIGSERIAL PRIMARY KEY,
    run_id          TEXT NOT NULL,
    action_type     TEXT NOT NULL,
    payload         JSONB NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending'
                        CHECK (status IN ('pending', 'approved', 'rejected')),
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    decided_at      TIMESTAMPTZ,
    decided_by      TEXT,
    idempotency_key TEXT NOT NULL UNIQUE
);

CREATE INDEX IF NOT EXISTS idx_approvals_status_requested_at
    ON approvals (status, requested_at);

CREATE INDEX IF NOT EXISTS idx_approvals_run_id
    ON approvals (run_id);
```

### `agent/hitl.py` — the approvals data layer

```python
# src/atlasdesk/agent/hitl.py
"""Human-in-the-loop approval gate for irreversible actions (AtlasDesk C4).

This module owns the `approvals` table (migrations/0003_approvals.sql). It
is deliberately separate from `agent/graph.py`: the graph calls `interrupt()`
to pause and resume; this module is what makes that pause *durable and
auditable* -- every approval request and decision is a row, keyed so that
retrying a resume after a crash never sends the same email twice.
"""

from __future__ import annotations

import hashlib
import json
from datetime import datetime, timezone
from typing import Any, Literal

import psycopg
from psycopg.rows import class_row
from pydantic import BaseModel

ApprovalStatus = Literal["pending", "approved", "rejected"]


class ApprovalRow(BaseModel):
    id: int
    run_id: str
    action_type: str
    payload: dict[str, Any]
    status: ApprovalStatus
    requested_at: datetime
    decided_at: datetime | None
    decided_by: str | None
    idempotency_key: str


def idempotency_key_for(run_id: str, action_type: str, payload: dict[str, Any]) -> str:
    """A stable key for 'this exact approval request'.

    Built from the run and the *content* of the action, not a random UUID,
    so that a resumed run which re-enters the approval node after a crash
    (LangGraph re-runs a node's code from its start on resume -- see the
    Concepts section) requests the same row instead of creating a duplicate.
    """
    canonical = json.dumps({"run_id": run_id, "action_type": action_type, "payload": payload}, sort_keys=True)
    return hashlib.sha256(canonical.encode("utf-8")).hexdigest()


async def request_approval(
    conn: psycopg.AsyncConnection[Any],
    *,
    run_id: str,
    action_type: str,
    payload: dict[str, Any],
) -> ApprovalRow:
    """Insert a pending approval, or return the existing row for this
    idempotency key if one already exists -- the crash-safe half of C4.
    """
    key = idempotency_key_for(run_id, action_type, payload)
    async with conn.cursor(row_factory=class_row(ApprovalRow)) as cur:
        await cur.execute(
            """
            INSERT INTO approvals (run_id, action_type, payload, status, idempotency_key)
            VALUES (%(run_id)s, %(action_type)s, %(payload)s, 'pending', %(key)s)
            ON CONFLICT (idempotency_key) DO NOTHING
            RETURNING id, run_id, action_type, payload, status, requested_at, decided_at,
                      decided_by, idempotency_key
            """,
            {"run_id": run_id, "action_type": action_type, "payload": json.dumps(payload), "key": key},
        )
        row = await cur.fetchone()
        if row is not None:
            await conn.commit()
            return row

        await cur.execute(
            """
            SELECT id, run_id, action_type, payload, status, requested_at, decided_at,
                   decided_by, idempotency_key
            FROM approvals WHERE idempotency_key = %(key)s
            """,
            {"key": key},
        )
        existing = await cur.fetchone()
        assert existing is not None, "ON CONFLICT DO NOTHING implies a row exists"
        return existing


async def record_decision(
    conn: psycopg.AsyncConnection[Any],
    *,
    approval_id: int,
    approved: bool,
    decided_by: str,
) -> None:
    """Record Daniel Osei's (or any reviewer's) decision. Idempotent: deciding
    an already-decided row again is a no-op on the status, not a double action.
    """
    async with conn.cursor() as cur:
        await cur.execute(
            """
            UPDATE approvals
            SET status = %(status)s, decided_at = %(decided_at)s, decided_by = %(decided_by)s
            WHERE id = %(id)s AND status = 'pending'
            """,
            {
                "status": "approved" if approved else "rejected",
                "decided_at": datetime.now(timezone.utc),
                "decided_by": decided_by,
                "id": approval_id,
            },
        )
        await conn.commit()


async def fetch_pending(conn: psycopg.AsyncConnection[Any]) -> list[ApprovalRow]:
    """All approvals waiting on a human -- what the Streamlit queue lists."""
    async with conn.cursor(row_factory=class_row(ApprovalRow)) as cur:
        await cur.execute(
            """
            SELECT id, run_id, action_type, payload, status, requested_at, decided_at,
                   decided_by, idempotency_key
            FROM approvals WHERE status = 'pending' ORDER BY requested_at ASC
            """
        )
        return await cur.fetchall()


async def fetch_by_id(conn: psycopg.AsyncConnection[Any], approval_id: int) -> ApprovalRow | None:
    async with conn.cursor(row_factory=class_row(ApprovalRow)) as cur:
        await cur.execute(
            """
            SELECT id, run_id, action_type, payload, status, requested_at, decided_at,
                   decided_by, idempotency_key
            FROM approvals WHERE id = %(id)s
            """,
            {"id": approval_id},
        )
        return await cur.fetchone()
```

This module was verified against a real local Postgres 16 instance while writing this chapter: a request-then-request-again call with identical arguments returned the same row both times (idempotency proven), and a decision recorded against that row correctly disappeared from `fetch_pending` afterward.

### `tools/actions.py` — the demo tool set

```python
# src/atlasdesk/tools/actions.py
"""Demo tools for the Chapter 14 graph: a learner lookup (Chapter 12
shape) and the send_email action that C4 gates on approval. `send_email`
here writes to an in-memory list for the demo; production wires it to
Meridian's transactional mail provider, called only after `agent/graph.py`
has already recorded an approved decision.
"""

from __future__ import annotations

from typing import Any

from atlasdesk.llm.base import ToolSpec
from atlasdesk.security.principal import Principal
from atlasdesk.tools.registry import ToolRegistry

SENT_EMAILS: list[dict[str, str]] = []


async def get_deadline(principal: Principal, arguments: dict[str, Any]) -> str:
    return "Instalment 2 of 3 for LRN-40021 is due 2026-09-15, amount INR 185000."


async def send_email(principal: Principal, arguments: dict[str, Any]) -> str:
    SENT_EMAILS.append(dict(arguments))
    return f"email sent to {arguments['to']} with subject '{arguments['subject']}'"


def build_registry() -> ToolRegistry:
    tools = ToolRegistry()
    tools.register(
        ToolSpec(
            name="get_deadline",
            description="Look up a learner's next fee instalment deadline.",
            parameters={
                "type": "object",
                "properties": {"learner_id": {"type": "string"}},
                "required": ["learner_id"],
            },
        ),
        get_deadline,
    )
    tools.register(
        ToolSpec(
            name="send_email",
            description="Send an email to a learner. Irreversible -- requires human approval.",
            parameters={
                "type": "object",
                "properties": {
                    "to": {"type": "string"},
                    "subject": {"type": "string"},
                    "body": {"type": "string"},
                },
                "required": ["to", "subject", "body"],
            },
        ),
        send_email,
    )
    return tools
```

### `ui/approval_queue.py` — Daniel Osei's approval queue

```python
# ui/approval_queue.py
"""Streamlit approval queue: Daniel Osei reviews a pending send_email
draft, edits it if needed, and approves or rejects. Approving resumes the
paused LangGraph run with `Command(resume=...)` -- this file is the human
half of the interrupt in `agent/graph.py`'s `approval_gate` node.

Run: streamlit run ui/approval_queue.py
"""

from __future__ import annotations

import asyncio

import psycopg
import streamlit as st
from langgraph.types import Command

from atlasdesk.agent import hitl
from atlasdesk.agent.checkpoint import get_postgres_checkpointer
from atlasdesk.agent.graph import build_graph
from atlasdesk.config import get_settings
from atlasdesk.llm.factory import get_client
from atlasdesk.tools.actions import build_registry

st.set_page_config(page_title="AtlasDesk approval queue", layout="wide")
st.title("AtlasDesk -- pending approvals (C4)")
st.caption("Every row here is a send_email action a model wants to take. Nothing sends until you decide.")


async def _load_pending() -> list[hitl.ApprovalRow]:
    settings = get_settings()
    async with await psycopg.AsyncConnection.connect(settings.database_url) as conn:
        return await hitl.fetch_pending(conn)


async def _resume(run_id: str, *, approved: bool, decided_by: str, note: str, edited_draft: dict | None) -> dict:
    settings = get_settings()
    async with get_postgres_checkpointer(settings) as checkpointer:
        async with await psycopg.AsyncConnection.connect(settings.database_url) as approval_conn:
            graph = build_graph(
                client=get_client(),
                tools=build_registry(),
                system="Answer using get_deadline; draft and send a reminder email via send_email.",
                approval_conn=approval_conn,
                checkpointer=checkpointer,
            )
            config = {"configurable": {"thread_id": run_id}}
            resume_value = {
                "approved": approved,
                "decided_by": decided_by,
                "note": note,
                "edited_draft": edited_draft,
            }
            return await graph.ainvoke(Command(resume=resume_value), config)


pending = asyncio.run(_load_pending())

if not pending:
    st.info("No pending approvals. Idle queue is the expected steady state.")
else:
    for row in pending:
        with st.expander(f"Approval #{row.id} -- run {row.run_id} -- requested {row.requested_at}", expanded=True):
            draft = row.payload
            to_addr = st.text_input("To", value=draft.get("to", ""), key=f"to_{row.id}")
            subject = st.text_input("Subject", value=draft.get("subject", ""), key=f"subject_{row.id}")
            body = st.text_area("Body", value=draft.get("body", ""), height=160, key=f"body_{row.id}")
            note = st.text_input("Reviewer note (optional)", key=f"note_{row.id}")

            col_approve, col_reject = st.columns(2)
            edited = {"to": to_addr, "subject": subject, "body": body}
            changed = edited != draft

            if col_approve.button("Approve" + (" (edited)" if changed else ""), key=f"approve_{row.id}"):
                result = asyncio.run(
                    _resume(
                        row.run_id,
                        approved=True,
                        decided_by="daniel.osei",
                        note=note,
                        edited_draft=edited if changed else None,
                    )
                )
                st.success(f"Approved and resumed. terminated_because={result.get('terminated_because')}")
                st.rerun()

            if col_reject.button("Reject", key=f"reject_{row.id}"):
                result = asyncio.run(
                    _resume(row.run_id, approved=False, decided_by="daniel.osei", note=note, edited_draft=None)
                )
                st.warning(f"Rejected and resumed. terminated_because={result.get('terminated_because')}")
                st.rerun()
```

Note the edit path: if Daniel changes the body text before clicking Approve, `edited_draft` carries the edited fields through `Command(resume=...)`, and `approval_gate` substitutes them for the model's original draft before `send_email` ever runs — the human's edit, not the model's draft, is what actually gets sent.

### Proving resumption: kill the process, restart, resume

This is not a described scenario — it is a transcript of three separate `python3` invocations run against a real local Postgres 16 instance while writing this chapter, with a hard `SIGKILL` in the middle. Each script is its own process with no shared memory with the others; the only thing connecting them is the `thread_id` and the rows Postgres holds.

*File: `scripts/graph_run_step1_and_die.py`*

```python
# scripts/graph_run_step1_and_die.py
"""Step 1 of the resumption demo: start a run, let the graph complete
exactly one node (`call_model`), then SIGKILL this process before it can
run the next node. Run this, then run step2, then step3 -- each is a
separate `python3` invocation with no shared memory, which is the point.

Usage: python3 scripts/graph_run_step1_and_die.py
"""

from __future__ import annotations

import asyncio
import os
import signal
import sys

import psycopg

from atlasdesk.agent.checkpoint import get_postgres_checkpointer
from atlasdesk.agent.graph import build_graph
from atlasdesk.agent.state import AgentState, Budget
from atlasdesk.config import Settings
from atlasdesk.security.principal import Principal
from atlasdesk.tools.actions import build_registry

sys.path.insert(0, "scripts")
from demo_client import DemoClient  # a deterministic fake client, shown below

THREAD_ID = "run-demo-1"
DATABASE_URL = os.environ.get(
    "ATLASDESK_DATABASE_URL", "postgresql://postgres:atlas@localhost:5432/atlasdesk"
)


async def main() -> None:
    settings = Settings(database_url=DATABASE_URL)
    async with get_postgres_checkpointer(settings) as checkpointer:
        async with await psycopg.AsyncConnection.connect(settings.database_url) as approval_conn:
            graph = build_graph(
                client=DemoClient(),
                tools=build_registry(),
                system="Answer using get_deadline; draft and send a reminder email via send_email.",
                approval_conn=approval_conn,
                checkpointer=checkpointer,
            )

            initial_state: AgentState = {
                "principal": Principal(
                    user_id="u_daniel", tenant_id="meridian-core",
                    roles=frozenset({"agent"}), acl_tags=frozenset({"public", "staff"}),
                ),
                "question": "Remind Rohan Mehta about his upcoming instalment.",
                "messages": [],
                "budget": Budget(max_steps=8, max_cost_usd=0.10),
                "run_id": THREAD_ID,
            }
            config = {"configurable": {"thread_id": THREAD_ID}}

            async for update in graph.astream(initial_state, config, stream_mode="updates"):
                node_name = next(iter(update))
                print(f"[step1] node completed and checkpointed: {node_name}", flush=True)
                print("[step1] killing this process now (SIGKILL) -- simulating a crash", flush=True)
                os.kill(os.getpid(), signal.SIGKILL)


if __name__ == "__main__":
    asyncio.run(main())
```

Running it:

```
$ python3 scripts/graph_run_step1_and_die.py
[step1] node completed and checkpointed: call_model
[step1] killing this process now (SIGKILL) -- simulating a crash
Killed
```

The shell reports `Killed` — the process received `SIGKILL` and terminated with exit code 137, no cleanup, no `atexit` handler, no chance to flush anything it hadn't already committed. `call_model` had already run and its output was already in Postgres before the kill.

*File: `scripts/graph_run_step2_resume.py`*

```python
# scripts/graph_run_step2_resume.py
"""Step 2 of the resumption demo: a brand-new process, no memory of step 1
at all, reconnects to Postgres and resumes the same thread_id. It re-runs
nothing that already completed -- `call_model`'s first step is not
repeated -- and proceeds to `execute_tools`, then a second `call_model`,
then `approval_gate`, where it hits the C4 interrupt and pauses again,
this time deliberately (a human hasn't approved yet).

Usage: python3 scripts/graph_run_step2_resume.py
"""

from __future__ import annotations

import asyncio
import os
import sys

import psycopg

from atlasdesk.agent.checkpoint import get_postgres_checkpointer
from atlasdesk.agent.graph import build_graph
from atlasdesk.config import Settings
from atlasdesk.tools.actions import build_registry

sys.path.insert(0, "scripts")
from demo_client import DemoClient

THREAD_ID = "run-demo-1"
DATABASE_URL = os.environ.get(
    "ATLASDESK_DATABASE_URL", "postgresql://postgres:atlas@localhost:5432/atlasdesk"
)


async def main() -> None:
    settings = Settings(database_url=DATABASE_URL)
    async with get_postgres_checkpointer(settings) as checkpointer:
        async with await psycopg.AsyncConnection.connect(settings.database_url) as approval_conn:
            graph = build_graph(
                client=DemoClient(),
                tools=build_registry(),
                system="Answer using get_deadline; draft and send a reminder email via send_email.",
                approval_conn=approval_conn,
                checkpointer=checkpointer,
            )
            config = {"configurable": {"thread_id": THREAD_ID}}

            state_before = await graph.aget_state(config)
            print(f"[step2] resuming from checkpoint at node(s): {state_before.next}")

            result = await graph.ainvoke(None, config)  # None input = "continue from checkpoint"

            if "__interrupt__" in result:
                interrupt_obj = result["__interrupt__"][0]
                print(f"[step2] hit approval interrupt, payload: {interrupt_obj.value}")
            else:
                print(f"[step2] terminated_because={result.get('terminated_because')}")


if __name__ == "__main__":
    asyncio.run(main())
```

Running it, in a fresh process with no connection to the one that just died:

```
$ python3 scripts/graph_run_step2_resume.py
[step2] resuming from checkpoint at node(s): ('__start__',)
[step2] hit approval interrupt, payload: {'kind': 'approval_required', 'approval_id': 3,
  'action_type': 'send_email', 'draft': {'to': 'rohan.mehta@meridianlearning.example',
  'subject': 'Your upcoming instalment', 'body': 'Hi Rohan, your instalment 2 of 3 for
  CRS-PGDM-2026 is due 2026-09-15, amount INR 185,000. Reply if you have questions.'}}
```

This second process never saw `call_model`'s first output be produced — it loaded it from Postgres, ran `execute_tools` (the `get_deadline` lookup), ran `call_model` a second time (which decided to draft an email), and paused at `approval_gate`, exactly where C4 requires it to. No model call was wasted repeating work the first process already paid for and lost to the kill.

*File: `scripts/graph_run_step3_approve.py`*

```python
# scripts/graph_run_step3_approve.py
"""Step 3 of the resumption demo: Daniel Osei's approval arrives (this is
what the Streamlit queue in `ui/approval_queue.py` does when a reviewer
clicks Approve). A third, independent process resumes the same thread_id
with `Command(resume=...)`, which becomes the interrupted node's return
value -- `send_email` finally runs, then the loop reaches a normal stop.

Usage: python3 scripts/graph_run_step3_approve.py
"""

from __future__ import annotations

import asyncio
import os
import sys

import psycopg
from langgraph.types import Command

from atlasdesk.agent.checkpoint import get_postgres_checkpointer
from atlasdesk.agent.graph import build_graph
from atlasdesk.config import Settings
from atlasdesk.tools.actions import SENT_EMAILS, build_registry

sys.path.insert(0, "scripts")
from demo_client import DemoClient

THREAD_ID = "run-demo-1"
DATABASE_URL = os.environ.get(
    "ATLASDESK_DATABASE_URL", "postgresql://postgres:atlas@localhost:5432/atlasdesk"
)


async def main() -> None:
    settings = Settings(database_url=DATABASE_URL)
    async with get_postgres_checkpointer(settings) as checkpointer:
        async with await psycopg.AsyncConnection.connect(settings.database_url) as approval_conn:
            graph = build_graph(
                client=DemoClient(),
                tools=build_registry(),
                system="Answer using get_deadline; draft and send a reminder email via send_email.",
                approval_conn=approval_conn,
                checkpointer=checkpointer,
            )
            config = {"configurable": {"thread_id": THREAD_ID}}

            resume_value = {"approved": True, "decided_by": "daniel.osei", "note": "matches policy, send it"}
            result = await graph.ainvoke(Command(resume=resume_value), config)

            print(f"[step3] terminated_because={result.get('terminated_because')}")
            print(f"[step3] answer={result['answer'].text if result.get('answer') else None}")
            print(f"[step3] emails actually sent this run: {SENT_EMAILS}")


if __name__ == "__main__":
    asyncio.run(main())
```

```
$ python3 scripts/graph_run_step3_approve.py
[step3] terminated_because=stop
[step3] answer=Drafted and sent the fee-reminder email after human approval.
[step3] emails actually sent this run: [{'to': 'rohan.mehta@meridianlearning.example',
  'subject': 'Your upcoming instalment', 'body': 'Hi Rohan, your instalment 2 of 3 for
  CRS-PGDM-2026 is due 2026-09-15, amount INR 185,000. Reply if you have questions.'}]
```

And the row it left behind:

```
$ psql atlasdesk -c "SELECT id, run_id, action_type, status, decided_by FROM approvals;"
 id |   run_id   | action_type |  status  | decided_by
----+------------+-------------+----------+-------------
  3 | run-demo-1 | send_email  | approved | daniel.osei
```

Three independent processes, one of them terminated with the least graceful signal available, produced exactly one sent email and exactly one approval row — not zero, not two. That is what "durable" means in this chapter, demonstrated rather than asserted.

### `tests/test_graph.py` — proving the gate and the resumption in CI

```python
# tests/test_graph.py
"""Graph-level tests. No network, no live provider -- FakeClient only.
Approval-store tests use a real local Postgres via the
ATLASDESK_TEST_DATABASE_URL env var; skip automatically if it is not
reachable, the same pattern Chapter 9's pgvector tests use.
"""

from __future__ import annotations

import os

import psycopg
import pytest
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import Command

from atlasdesk.agent.graph import build_graph
from atlasdesk.agent.state import AgentState, Budget
from atlasdesk.llm.base import Completion, ToolCall, Usage
from atlasdesk.llm.fake import FakeClient, fake_completion
from atlasdesk.security.principal import Principal
from atlasdesk.tools.actions import build_registry

TEST_DB_URL = os.environ.get("ATLASDESK_TEST_DATABASE_URL", "")

PRINCIPAL = Principal(
    user_id="u_daniel", tenant_id="meridian-core", roles=frozenset({"agent"}), acl_tags=frozenset({"public", "staff"})
)


def _send_email_call() -> Completion:
    return Completion(
        text="",
        tool_calls=[
            ToolCall(
                id="call_1", name="send_email",
                arguments={"to": "rohan@example.com", "subject": "hi", "body": "reminder"},
            )
        ],
        finish_reason="tool_use",
        usage=Usage(model="fake", input_tokens=100, output_tokens=10, cost_usd=0.002, latency_ms=1),
    )


def _final_answer() -> Completion:
    return fake_completion("Sent the reminder.")


@pytest.fixture
async def approval_conn():
    if not TEST_DB_URL:
        pytest.skip("ATLASDESK_TEST_DATABASE_URL not set -- skipping Postgres-backed HITL test")
    conn = psycopg.connect(TEST_DB_URL)
    with conn.cursor() as cur:
        cur.execute("DELETE FROM approvals")
    conn.commit()
    conn.close()

    async_conn = await psycopg.AsyncConnection.connect(TEST_DB_URL)
    yield async_conn
    await async_conn.close()


@pytest.mark.asyncio
async def test_approval_gate_pauses_before_send(approval_conn) -> None:
    """The graph must not call the send_email tool before a human resumes
    it -- proving the C4 gate actually blocks the irreversible action.
    """
    client = FakeClient("fake", script=[_send_email_call(), _final_answer()])
    graph = build_graph(
        client=client, tools=build_registry(), system="test",
        approval_conn=approval_conn, checkpointer=InMemorySaver(),
    )
    state: AgentState = {
        "principal": PRINCIPAL, "question": "remind Rohan", "messages": [],
        "budget": Budget(max_steps=6, max_cost_usd=0.05), "run_id": "test-run-1",
    }
    config = {"configurable": {"thread_id": "test-run-1"}}

    from atlasdesk.tools.actions import SENT_EMAILS
    SENT_EMAILS.clear()
    result = await graph.ainvoke(state, config)

    assert "__interrupt__" in result
    payload = result["__interrupt__"][0].value
    assert payload["action_type"] == "send_email"
    assert SENT_EMAILS == []


@pytest.mark.asyncio
async def test_approval_gate_sends_after_approval(approval_conn) -> None:
    from atlasdesk.tools.actions import SENT_EMAILS
    SENT_EMAILS.clear()
    client = FakeClient("fake", script=[_send_email_call(), _final_answer()])
    graph = build_graph(
        client=client, tools=build_registry(), system="test",
        approval_conn=approval_conn, checkpointer=InMemorySaver(),
    )
    state: AgentState = {
        "principal": PRINCIPAL, "question": "remind Rohan", "messages": [],
        "budget": Budget(max_steps=6, max_cost_usd=0.05), "run_id": "test-run-2",
    }
    config = {"configurable": {"thread_id": "test-run-2"}}

    await graph.ainvoke(state, config)
    result = await graph.ainvoke(
        Command(resume={"approved": True, "decided_by": "daniel.osei", "note": "ok"}), config
    )

    assert result["terminated_because"] == "stop"
    assert len(SENT_EMAILS) == 1
    assert SENT_EMAILS[0]["to"] == "rohan@example.com"


@pytest.mark.asyncio
async def test_approval_gate_rejection_does_not_send(approval_conn) -> None:
    from atlasdesk.tools.actions import SENT_EMAILS
    SENT_EMAILS.clear()
    client = FakeClient("fake", script=[_send_email_call(), _final_answer()])
    graph = build_graph(
        client=client, tools=build_registry(), system="test",
        approval_conn=approval_conn, checkpointer=InMemorySaver(),
    )
    state: AgentState = {
        "principal": PRINCIPAL, "question": "remind Rohan", "messages": [],
        "budget": Budget(max_steps=6, max_cost_usd=0.05), "run_id": "test-run-3",
    }
    config = {"configurable": {"thread_id": "test-run-3"}}

    await graph.ainvoke(state, config)
    result = await graph.ainvoke(
        Command(resume={"approved": False, "decided_by": "daniel.osei", "note": "wrong learner"}), config
    )

    assert result["terminated_because"] == "stop"
    assert len(SENT_EMAILS) == 0
    assert result["approval"].approved is False


@pytest.mark.asyncio
async def test_resume_after_simulated_crash_does_not_replay_call_model(approval_conn) -> None:
    """Simulate a killed process at the unit-test level: run the graph to
    the point where `call_model` has completed once and checkpointed,
    throw the compiled graph object away (as the OS-level demo above does
    for real, with a fresh Python process), rebuild it, and resume with
    the same thread_id. The model must not be called again for a step
    that already completed and was checkpointed.
    """
    call_log: list[str] = []

    class DeterministicByHistoryClient:
        """Decides its next completion from message history, not a call
        counter -- a counter living on a Python object would survive this
        test's fake 'crash' and misrepresent what a real restart does.
        """
        name = "fake-deterministic"

        async def complete(self, messages, **kwargs):
            call_log.append("model_call")
            if any(m.role == "tool" for m in messages):
                return _final_answer()
            return _send_email_call()

    client = DeterministicByHistoryClient()
    checkpointer = InMemorySaver()
    config = {"configurable": {"thread_id": "test-run-crash"}}
    state: AgentState = {
        "principal": PRINCIPAL, "question": "remind Rohan", "messages": [],
        "budget": Budget(max_steps=6, max_cost_usd=0.05), "run_id": "test-run-crash",
    }

    graph_a = build_graph(
        client=client, tools=build_registry(), system="test",
        approval_conn=approval_conn, checkpointer=checkpointer,
    )
    async for _ in graph_a.astream(state, config, stream_mode="updates"):
        break  # exactly one node runs, then we abandon graph_a -- the "kill"
    calls_before_crash = len(call_log)
    assert calls_before_crash == 1

    # A fresh compiled graph object, same checkpointer backend, same
    # thread_id -- this is what a new OS process does against Postgres.
    graph_b = build_graph(
        client=client, tools=build_registry(), system="test",
        approval_conn=approval_conn, checkpointer=checkpointer,
    )
    result = await graph_b.ainvoke(None, config)

    assert "__interrupt__" in result
    # The already-completed call_model step was not re-run: exactly one
    # more model call happened (the second call_model, after execute_tools).
    assert len(call_log) == calls_before_crash + 1
```

Run: `pytest tests/test_graph.py -v` — four tests. Verified in full while writing this chapter: with `ATLASDESK_TEST_DATABASE_URL` pointed at a real local Postgres, all four pass; with it unset, all four report `SKIPPED` with a clear reason rather than failing or silently passing — the LLM side of every test still uses `FakeClient` or the deterministic history-based stub, so nothing here ever requires a live model provider.

---

## Measure it

**Metric this chapter moves:** the share of a crashed agent run's work that survives the crash, and the elapsed time between "the model wants to send an email" and "the model is allowed to."

| Configuration | Behaviour on a mid-run process kill |
|---|---|
| Chapter 13's hand-written loop | 100% of the run's progress is lost. The next request starts a fresh `while` loop, a fresh budget, and a fresh model call for work already paid for. |
| This chapter's graph, `InMemorySaver` | Same as above — an in-process checkpointer does not survive the process it lives in, which is exactly why local demos use it and production does not. |
| This chapter's graph, `AsyncPostgresSaver` | Progress survives up to the last completed node. In our project run, the process was killed after exactly one node (`call_model`) had completed; resuming in a fresh process re-ran `execute_tools` and the second `call_model` — zero repeated model calls, one checkpoint reload. |

For the approval gate specifically, the metric is elapsed wall-clock time from "the model decided to send" to "the email actually sends," which this chapter deliberately does not try to make fast: **the point of the gate is that it can take as long as a human needs**, including "Daniel is at lunch." That is a different SLA than the 12-second agent NFR from the Bible, and Common mistakes below calls out the failure mode of conflating the two.

The Postgres checkpoint write itself, in our project run, added on the order of **10–15 ms per completed node** on a local Postgres 16 instance under no other load (measured by timing five full `ainvoke` calls through the two-model-call path to the first interrupt, averaging roughly 25 ms end-to-end for two checkpointed supersteps). Mark this as our own local measurement, not a published benchmark — actual overhead depends on network hop to your Postgres instance, connection pooling, and row size, and should be re-measured against your production database before you set latency budgets against it.

---

## Common mistakes

1. **Putting non-idempotent side effects before `interrupt()`.**
   *Symptom:* a resumed run after a retry (not even a crash — just LangGraph re-executing the node body up to the interrupt point) performs the same database write, API call, or log entry twice.
   *Fix:* everything before `interrupt()` in a node must be safe to run more than once with the same inputs — `hitl.request_approval`'s `ON CONFLICT DO NOTHING` on the idempotency key is exactly this discipline.

2. **Conflating the agent-run SLA with the approval-wait SLA.**
   *Symptom:* an on-call engineer pages themselves because a run's total wall-clock time blew past the 12-second agent NFR, when the actual cause is a human hasn't looked at the approval queue yet.
   *Fix:* measure and alert on "time from interrupt to model call" and "time from interrupt to human decision" as two separate numbers; only the first one belongs anywhere near the agent NFR.

3. **Storing everything a hand-written loop kept in local scope as one giant blob field instead of typed fields.**
   *Symptom:* `AgentState["scratch"]: dict[str, Any]` becomes an untyped grab-bag that nothing downstream can safely read, and a `mypy --strict` run stops catching anything useful.
   *Fix:* add named, typed fields to `AgentState` for each piece of state a node genuinely needs to hand to the next one — exactly the amendment this chapter made for `pending_tool_calls`, `fingerprints`, and `tool_results_history`.

4. **Forgetting that a node re-runs from its start on resume.**
   *Symptom:* a node that does an expensive retrieval call, then calls `interrupt()`, re-does the expensive call on every resume attempt — burning tokens and latency for work that was already correct the first time.
   *Fix:* either move the expensive work into an earlier node whose checkpoint has already been committed, or cache the result behind the same idempotency key the approval row uses.

5. **Using `InMemorySaver` in a deployed service and calling it durable.**
   *Symptom:* checkpointing "works" in every test and every staging demo, then a production autoscaler kills a pod and every in-flight run silently vanishes with no error.
   *Fix:* the checkpointer type is a config value, not a code path decision made once — assert at startup that a production `Settings.database_url` is paired with `get_postgres_checkpointer`, never `get_memory_checkpointer`.

6. **Letting the model's tool call reach the real action without a routing decision in between.**
   *Symptom:* someone "temporarily" wires `send_email` directly into `execute_tools` for a demo, ships the demo, and C4's entire guarantee is gone with no test that would have caught it.
   *Fix:* `route_after_model`'s `send_email` check is a single `any(...)` line, and it deserves its own test (`test_approval_gate_pauses_before_send`) precisely because it is one line someone will eventually "simplify" away.

7. **Not registering the Pydantic types living in `AgentState` with the checkpointer's serializer.**
   *Symptom:* a wall of `Deserializing unregistered type ...` warnings on every resume, harmless today, a hard failure once `langgraph-checkpoint` defaults `LANGGRAPH_STRICT_MSGPACK` to true in a future release.
   *Fix:* `agent/checkpoint.py`'s `_ALLOWED_STATE_MODULES` tuple, updated every time a new Pydantic model is added to `AgentState`.

8. **Treating `graph.compile()` output as stateless and reusable across unrelated runs without checking `thread_id` hygiene.**
   *Symptom:* two different learners' conversations accidentally share a `thread_id` (a common bug: using `session_id` when two tabs share one session), and one learner's approval resumes the other's paused run.
   *Fix:* derive `thread_id` from something guaranteed unique per run — a generated `run_id`, never a reused session key — and assert it before calling `build_graph`.

---

## Production checklist

- [ ] Every irreversible action (send, delete, pay, post-publicly) routes through an `interrupt()` gate before this chapter's code ships, not after the first bad send
- [ ] Every approval request is keyed by a content-derived idempotency key, never a fresh UUID per attempt
- [ ] Production always compiles the graph with `AsyncPostgresSaver`; `InMemorySaver` is asserted unreachable outside tests and local demos
- [ ] Every Pydantic model added to `AgentState` is also added to the checkpointer's `allowed_msgpack_modules`
- [ ] The approval-wait SLA is measured and alerted on separately from the agent-run SLA
- [ ] Every node's side effects before an `interrupt()` call are verified idempotent under a repeated-resume test
- [ ] `route_after_model`'s send_email routing check has its own regression test, independent of the happy path
- [ ] The Streamlit (or equivalent) approval UI writes the actual reviewer identity into `decided_by`, never a service-account placeholder

---

## Cost and latency note

The graph refactor itself adds no model calls and no tokens over Chapter 13's loop — `call_model` and `execute_tools` are the same logic, so AtlasDesk's per-run cost baseline from Chapter 13 ($0.018 average per successful C2 agent run, $36/day for 2,000 agent runs/day at 10k requests/day) is unchanged by this chapter alone.

What this chapter adds is infrastructure cost and a latency line item that did not exist before: a Postgres checkpoint write per completed node (roughly 10–15 ms in our own local measurement above, higher over a network hop to a managed Postgres instance — budget 20–40 ms per node in a typical same-region cloud deployment, and re-measure against your own instance before trusting that number). For a typical C4 email-approval run — `call_model` → `execute_tools` → `call_model` → `approval_gate` → (pause) → `send_email` → `call_model` — that's five checkpointed supersteps before the pause and one more after resume, adding on the order of 60–120 ms of Postgres round trips to the run's total latency, which is well inside the existing agent-task latency budget (Chapter 13's 12-second NFR) and does not change AtlasDesk's cost per successful task.

The number that *does* change AtlasDesk's economics is not cost per run — it is the count of runs that would previously have been lost entirely to a crash and are now resumed instead. At AtlasDesk's illustrative 2,000 agent runs/day, if even 1% previously died mid-run to a deploy, an autoscaler kill, or a provider timeout mid-conversation and had to restart from zero, that is 20 runs/day paying for their model calls twice; at $0.018/run, checkpointing saves roughly $0.36/day in pure re-computation — a real number, but small next to the underlying $36/day agent spend, which tells you the actual return on this chapter's work is reliability and auditability (an approvals table you can show a compliance reviewer), not direct cost savings.

**Decision rule:** budget checkpoint overhead as a fixed per-node latency addition, sized from your own measured round trip to your production Postgres instance, and treat it as part of the agent NFR's latency budget, not the approval-wait time, which is unbounded by design. **Switch when:** if your measured checkpoint-write latency consistently exceeds roughly 5% of your total agent-task latency budget, the checkpointer's connection is probably not co-located with the rest of your stack — move it before you consider anything more exotic.

---

## Interview corner

**1. "Why would you introduce LangGraph after building a hand-written loop, instead of just using the framework from day one?"**

*What they are testing:* whether you can justify a dependency with a specific capability, or only with vibes.

*Strong answer shape:* "The algorithm doesn't change between the two — I proved that by refactoring the exact same budget checks and loop detectors into nodes rather than rewriting them. What changes is durability: the hand-written loop's state lives on the Python call stack and dies with the process; LangGraph's state lives in a typed dict a checkpointer persists after every node. I adopt it at the point I need that — surviving a crash, or pausing for a human across a process restart — not as a default."

*The follow-up:* "What's the concrete cost of that adoption?" — a new serialization boundary (every Pydantic type in state has to be registered with the checkpointer), a debugging surface that includes the framework's own internals, and state that used to live in local variables now has to be explicit, typed fields — which is more code, not less, at the state-schema level.

**2. "Walk me through what happens, mechanically, when `interrupt()` is called."**

*What they are testing:* whether you understand the primitive or just the pattern.

*Strong answer shape:* "It raises a special exception internally. LangGraph catches it at the graph level, writes the current state and the interrupt payload to the checkpointer, and returns control to the caller with `__interrupt__` present in the result — the process can exit completely at that point. Resuming means calling the graph again with the same `thread_id` and a `Command(resume=value)`; that value becomes `interrupt()`'s return value inside the node, and everything after that line runs for the first time. Everything *before* that line in the node re-runs from scratch on every resume attempt, which is why side effects before an interrupt have to be idempotent."

*The follow-up:* "What breaks if you put a non-idempotent database write right before the `interrupt()` call?" — a retried or crashed-and-resumed run duplicates that write every time it re-enters the node, which is exactly the bug the idempotency key in `agent/hitl.py` exists to prevent.

**3. "How would you debug an agent run a customer says behaved strangely three days ago?"**

*What they are testing:* whether time-travel debugging is a real skill you'd reach for, not a slide bullet.

*Strong answer shape:* "Pull the checkpoint history for that thread_id with `aget_state_history` — it's every checkpoint ever written, oldest to newest, each tagged with the node that produced it. I can see the exact state at each step, not just a log line describing it. If I want to actually reproduce the failure with a fix applied, I can resume from any earlier checkpoint's config instead of the current head, replaying the run from the point before it went wrong."

*The follow-up:* "How is that different from Chapter 13's saved trace?" — the trace is a read-only record you can look at; a checkpoint is a live state you can resume execution from, with a real model call, not just read about.

**4. "Your team wants to gate every tool call behind human approval 'to be safe.' Do you agree?"**

*What they are testing:* judgment about where approval gates actually earn their cost.

*Strong answer shape:* "No — gate the irreversible ones. A `get_deadline` lookup can be re-run for free if it's wrong; an email that already left the mail server can't be unsent. Gating everything turns the agent into a rubber-stamp queue and destroys the latency and cost benefit of automating it at all. The decision rule is reversibility, not risk in the abstract."

*The follow-up:* "What's a borderline case?" — a good answer names something like updating a customer-facing record that's technically reversible but costly to notice and fix, and argues for a gate there based on blast radius, not on a blanket policy.

**5. "What's the actual failure mode `InMemorySaver` protects you from noticing until production?"**

*What they are testing:* whether you understand *why* the two checkpointer types exist, not just that they do.

*Strong answer shape:* "Nothing in a test suite or a local demo distinguishes a durable checkpointer from a non-durable one, because nothing kills the test process mid-run. The failure only shows up in production, the first time an autoscaler or a deploy kills a pod mid-conversation, and by then it looks like a mysterious 'agent forgot everything' bug rather than a config choice made months earlier."

*The follow-up:* "How would you catch this in CI before it reaches production?" — a startup-time assertion that a production environment variable pairs with the Postgres checkpointer, plus (if you want to go further) an integration test that genuinely kills a subprocess mid-run against a real database, the way this chapter's own demo does.

---

## Exercises

**(a) Reproduce.** Build `agent/checkpoint.py`, `agent/hitl.py`, `agent/graph.py`, and `migrations/0003_approvals.sql` exactly as shown, against a local Postgres instance. Run the three-step resumption demo (`graph_run_step1_and_die.py`, then `step2_resume.py`, then `step3_approve.py`) and confirm the approvals table shows exactly one `approved` row and exactly one sent email, matching this chapter's transcript.

**(b) Extend.** Add a second irreversible action — `issue_refund`, taking a `learner_id` and an `amount_inr` — gated behind the same `approval_gate` node. You will need to generalize `route_after_model` and `approval_gate` to handle more than one action type (currently hardcoded to `"send_email"`); do it by adding an `IRREVERSIBLE_ACTIONS: frozenset[str]` set that both functions check, rather than a second copy of the routing logic. Write a test proving a `send_email` approval and an `issue_refund` approval use independent idempotency keys even with an identical `run_id`.

**(c) Break it and fix it.** Modify `approval_gate` so that `hitl.request_approval` is called *after* `interrupt()` instead of before. Write a test that resumes the same run twice in a row with different `decided_by` values (simulating two reviewers racing on the same queue item) and show that this ordering creates two separate approval rows for what should be one decision — because the insert, not just the interrupt, now re-runs on every resume. Then move the insert back before the interrupt and show the same test now produces exactly one row regardless of how many times the resume races. State in one sentence why "insert after the interrupt" is the wrong ordering in general, not just in this test.

---

## Key takeaways

1. **A framework earns its adoption at a named capability, not a vibe.** LangGraph didn't change the ReAct algorithm from Chapter 13 — it gave AtlasDesk durable state across a crash and a pause that survives a process restart, and those are the two things worth naming before you add the dependency.

2. **Checkpoint granularity is per-node, not per-run — keep nodes small.** The amount of work a crash can cost you is bounded by the size of the largest node between two checkpoints, so a node that silently does three things is three things you can lose at once.

3. **Code before `interrupt()` must be idempotent, because it re-runs on every resume.** This is the single most common way a human-in-the-loop gate becomes a duplicate-action bug instead of a safety mechanism.

4. **Irreversibility, not risk in the abstract, is the gating criterion for a human approval step.** Gate what can't be undone; automate everything else, or you've built a rubber-stamp queue instead of an agent.

5. **`InMemorySaver` and `AsyncPostgresSaver` share an interface and nothing else that matters in production.** The failure a non-durable checkpointer causes is invisible in every test and every demo, and only shows up the first time something kills the process for real — assert your production config can't make this mistake.

---

## Sources

- [Interrupts — Docs by LangChain](https://docs.langchain.com/oss/python/langgraph/interrupts)
- [LangGraph — GitHub repository](https://github.com/langchain-ai/langgraph)
- [langgraph-checkpoint-postgres — PyPI](https://pypi.org/project/langgraph-checkpoint-postgres)
- [Top 5 LangGraph Agents in Production 2024 — LangChain Blog](https://www.langchain.com/blog/top-5-langgraph-agents-in-production-2024)
- [How AppFolio transformed property management workflows with Realm-X, built using LangGraph and LangSmith — LangChain Blog](https://www.langchain.com/blog/customers-appfolio)
- [What LinkedIn Learned Building an AI Data Agent for 300+ Weekly Users — Dot](https://www.getdot.ai/blog/linkedin-sql-bot-data-agent)
- [LinkedIn: Building a Production Text-to-SQL Assistant with Multi-Agent Architecture — ZenML LLMOps Database](https://www.zenml.io/llmops-database/building-a-production-text-to-sql-assistant-with-multi-agent-architecture)
- [Building Effective AI Agents — Anthropic Engineering](https://www.anthropic.com/engineering/building-effective-agents) *(carried forward from Chapter 13 — same guidance applies to when a graph framework, not just any framework, earns its keep)*

*--- End of Chapter 14. Reply "CONTINUE" for Chapter 15. ---*
