# Claude Agent with Tools — Complete Notes

## What Is a Claude Agent?

A standard Claude interaction is **stateless and single-turn** — you send a prompt, Claude responds, done. A **Claude agent** is different. It operates in a loop, using tools to interact with the real world, gather information, take actions, and reason across multiple steps before delivering a final answer.

The key shift: Claude moves from being a **responder** to being an **actor**.

---

## The Mental Model: Think → Act → Observe → Repeat

```
User Prompt
    ↓
[THINK]  Claude reasons about what tool(s) are needed
    ↓
[ACT]    Claude emits a tool_use block — generation pauses
    ↓
[OBSERVE] Tool runs externally — result injected into context
    ↓
[REPEAT] Does Claude need more tools? → loop back
    ↓
[RESPOND] All information gathered → generate final text response
```

This loop can repeat **zero times** (plain answer) or **many times** (complex multi-step tasks). The number of iterations is not fixed — Claude decides dynamically.

---

## How Tools Are Defined

Tools are not magic. They are just **text descriptions in the context window**, structured as JSON schemas. Claude reads them the same way it reads your prompt.

```json
{
  "name": "web_search",
  "description": "Search the web for current, real-time information.",
  "input_schema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "The search query string"
      }
    },
    "required": ["query"]
  }
}
```

Claude was trained to recognize these definitions and generate correctly structured calls against them. There is no special mechanism — it is all next-token prediction, but learned behavior from training.

---

## What a Tool Call Looks Like (Inside the Model)

When Claude decides to use a tool, it does **not** write prose. It generates a structured `tool_use` content block:

```json
{
  "type": "tool_use",
  "id": "toolu_01Abc123",
  "name": "gmail_search",
  "input": {
    "query": "budget Q2"
  }
}
```

Generation **stops** here. The model outputs no more tokens until the tool result arrives.

### The tool result is then injected back:

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_01Abc123",
  "content": "Email found — Subject: Budget Q2 Approved. Body: Total $450k approved for Q2..."
}
```

This result is **appended to the context window** as a new message, and Claude resumes generation from there.

---

## The Context Window as Working Memory

This is the most important architectural point:

> **Claude has no persistent state. The context window is its entire working memory for a task.**

Every message, every tool call, every tool result — all appended to one growing sequence of tokens. Here is what the context looks like mid-task:

```
[SYSTEM PROMPT — behavior rules, tool definitions]
[USER] "Find my budget email and create a calendar event"
[ASSISTANT] <tool_use: gmail_search("budget")>
[TOOL RESULT] "Email found: Budget Q2 — $450k approved, meeting Fri 3pm"
[ASSISTANT] <tool_use: calendar_create(title="Budget Review", date="Friday 3pm")>
[TOOL RESULT] "Event created. ID: evt_abc123. Link: calendar.google.com/..."
[ASSISTANT] "Done! I found your budget email ($450k Q2 approved) and created
             a 'Budget Review' event for Friday at 3pm. Here's your calendar link..."
```

The model never "remembers" anything — it re-reads this entire history on every pass. This means longer tasks consume more context budget.

---

## Multi-Step Example Walkthrough

**Prompt:** *"Find my latest email about the budget, then create a calendar event for it."*

### Pass 1 — First tool call
- Claude reads the prompt and tool definitions
- Decides: *"I need to search Gmail first"*
- Outputs: `tool_use: gmail_search(query="budget")`
- Generation pauses

### Pass 2 — After Gmail result
- Gmail result injected into context: *"Budget Q2 email found, mentions Friday 3pm meeting"*
- Claude reads updated context
- Decides: *"Now I can create the calendar event"*
- Outputs: `tool_use: calendar_create(title="Budget Review", date="Friday 3pm")`
- Generation pauses again

### Pass 3 — After Calendar result
- Calendar result injected: *"Event created successfully"*
- Claude reads final context
- Decides: *"Both tasks complete, I can now write the answer"*
- Outputs: Final text response, streamed token by token to the user

**Total passes through the transformer:** 3 (one per tool call) + 1 (final response) = **4 full forward passes**

---

## Types of Tools Available

| Category | Examples | What it enables |
|---|---|---|
| **Web search** | `web_search`, `arxiv_search` | Access real-time, current information |
| **Code execution** | `code_exec`, `bash` | Run scripts, process data, do math |
| **File system** | `read_file`, `write_file` | Read/write documents on disk |
| **Email** | `gmail_search`, `send_email` | Read inbox, compose and send emails |
| **Calendar** | `create_event`, `list_events` | Schedule and manage time |
| **Databases** | SQL queries, vector search | Retrieve structured or semantic data |
| **External APIs** | Slack, Notion, Salesforce, GitHub | Interact with any connected service |
| **Browser** | `navigate`, `click`, `screenshot` | Control a real web browser |
| **Sub-agents** | Another Claude instance | Delegate sub-tasks to specialist agents |

---

## Critical Design Principles

### 1. Claude never directly executes tools
Claude only **generates text describing a call**. The actual execution — hitting an API, running code, accessing files — always happens in external, trusted infrastructure. Claude cannot directly touch the internet, your filesystem, or any service. It can only *ask* for it.

This separation is fundamental to safety. A compromised or hallucinating Claude can generate a bad tool call, but it cannot execute it without the orchestrating system running it.

### 2. Tool calling is just text generation
There is no special "tool mode" switch. Claude generates a `tool_use` block because its training made that the highest-probability continuation when a tool is appropriate. The structured format emerges from learned behavior, not from hard-coded logic.

### 3. Claude decides how many tools to use
There is no external planner telling Claude "call tool A, then tool B." Claude figures this out autonomously by reading the task and the available tool definitions, then generating the most appropriate next action — which might be a tool call, or might be a final answer.

### 4. Errors are part of the loop
If a tool returns an error, that error is injected into the context just like a success result. Claude reads it and decides what to do — retry with different parameters, call a different tool, or tell the user what went wrong.

---

## Multi-Agent Architecture

In advanced setups, tools can invoke **other Claude instances**, creating hierarchies:

```
Manager Claude (sees the big picture)
    ├── calls → Research Agent (web search, summarization)
    ├── calls → Coder Agent (writes and runs code)
    └── calls → Writer Agent (drafts final document)
```

Each sub-agent has its own context window, its own tool set, and its own loop. The manager Claude receives their outputs as tool results and synthesizes a final answer. This is how very long, complex tasks are decomposed.

---

## What Makes an Agent Different from a Chatbot

| Dimension | Chatbot | Agent |
|---|---|---|
| **Actions** | Generates text only | Can act in the real world via tools |
| **Information** | Limited to training data | Can fetch live, external data |
| **Steps** | Single pass | Multiple reasoning + action steps |
| **Memory** | Context window only | Can read/write files, databases |
| **Autonomy** | Responds when asked | Can execute multi-step plans |
| **Side effects** | None | Can send emails, create files, etc. |

---

## The Agentic Loop in One Sentence

> Claude is a text predictor that was trained to recognize when the right next token is a structured tool call — and an orchestration layer outside the model that actually runs those calls and feeds the results back in, turning a single-turn text generator into a multi-step autonomous agent.
