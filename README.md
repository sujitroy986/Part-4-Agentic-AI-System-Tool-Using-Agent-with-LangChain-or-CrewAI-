# Part 4 — Agentic AI System: Olist Data & Live Weather Agent

## Option chosen

**Option A — LangChain single autonomous agent.**

This project demonstrates a tool-using LangChain agent backed by Google Gemini. The agent can autonomously decide whether to use:

1. `get_weather` — a live Open-Meteo weather API tool.
2. `lookup_olist_order` — a safe read-only local-data lookup tool built from the supplied Olist Brazilian E-Commerce dataset.

The implementation also demonstrates:
- `@tool` decorated functions with explicit docstrings.
- `create_tool_calling_agent` + `AgentExecutor`.
- bounded `max_iterations`.
- native LangChain `AIMessage.tool_calls` inspection and logging as `{"tool": ..., "arguments": {...}}`.
- two-turn conversation memory.
- a separate `RunnablePassthrough.assign` + `RunnableBranch` conditional workflow.
- both conditional branches.
- three distinct end-to-end demonstration queries.

## Dataset used

The supplied Olist files were inspected. The project uses a compact 200-order sample derived from the supplied files so the GitHub repository remains lightweight.

The sample combines:
- `olist_orders_dataset.csv`
- `olist_customers_dataset.csv`
- `olist_order_reviews_dataset.csv`
- `olist_order_payments_dataset.csv`

The resulting `data/olist_order_lookup_sample.csv` contains:
`order_id`, `customer_id`, `order_status`, `order_purchase_timestamp`, `customer_state`, `review_score`, `review_comment_message`, `payment_value`.

The local tool is intentionally read-only.

## Architecture

```text
User query
    |
    v
Gemini Chat Model
    |
    +---- decides no tool ----> final response
    |
    +---- get_weather ---------> Open-Meteo
    |
    +---- lookup_olist_order --> local Olist sample
    |
    v
Tool result returned to agent
    |
    v
Gemini synthesizes final answer
```

## Tool-contract table

| Tool | One-line description | Parameters | Read/Write |
|---|---|---|---|
| `get_weather` | Fetches current weather for a city using the live Open-Meteo geocoding + forecast APIs. | `city: str` | **Read** |
| `lookup_olist_order` | Looks up one order in the local Olist sample and returns selected order/customer/review/payment facts. | `order_id: str` | **Read** |

### Good-tool properties

**Clear name:** names describe the action directly.

**Honest description:** docstrings explicitly state what each tool can and cannot return.

**Atomic:** each tool performs one job only.

**Safe:** network and data errors are caught and returned as structured text; the tools do not intentionally crash the agent. Neither tool changes state.

## Live API

`get_weather` uses Open-Meteo and requires no API key.

Weather data by Open-Meteo.com (CC BY 4.0).

## Secrets

For live Gemini execution, create a local `.env` file:

```text
GOOGLE_API_KEY=your_actual_key
```

Never commit `.env`.

The repository contains only `.env.example`.

## Installation

Recommended Python version: **3.11**.

Create a virtual environment:

```bash
python -m venv .venv
```

Windows:

```bash
.venv\Scripts\activate
```

macOS/Linux:

```bash
source .venv/bin/activate
```

Install exact pinned dependencies:

```bash
pip install -r requirements.txt
```

## Run the agent

### Live Gemini mode

1. Copy `.env.example` to `.env`.
2. Add your Google Gemini API key.
3. Run:

```bash
python -m src.demo --live
```

The program runs three distinct demonstrations and writes:

```text
outputs/demo_trace.json
```

### Deterministic mock mode

Mock mode is useful for checking the complete LangChain orchestration and producing a reproducible trace without consuming an API key:

```bash
python -m src.demo --mock
```

The mock mode uses LangChain's fake chat-message model to supply deterministic native tool calls. It does **not** claim that Gemini made those decisions. Live Gemini mode is the actual autonomous LLM demonstration.

## Three demonstrated queries

The demonstration script covers:

### Query 1 — live external API

> What is the current weather in New Delhi?

Expected routing:

```json
{"tool": "get_weather", "arguments": {"city": "New Delhi"}}
```

The final answer is generated after the tool result is returned to the model.

### Query 2 — local Olist data

The script selects an order ID from the supplied Olist-derived sample and asks:

> Look up order `<sample_order_id>` and tell me its status, review score and payment value.

Expected routing:

```json
{"tool": "lookup_olist_order", "arguments": {"order_id": "<sample_order_id>"}}
```

### Query 3 — combined task

> What is the weather in New Delhi, and also look up order `<sample_order_id>`?

The agent may call both tools. The exact call order is model-dependent; the native calls are captured from LangChain at runtime.

## Native tool-call contract

LangChain's standardized tool-calling interface exposes model-selected calls through `AIMessage.tool_calls`, with fields such as tool `name` and parsed `args`. This project does **not** regex-parse model text to discover tool calls.

The logger in `src/agent.py` uses LangChain's native `AgentAction` callback:

```python
record = {
    "tool": action.tool,
    "arguments": action.tool_input,
}
print(json.dumps(record))
```

This is emitted by `NativeToolDecisionLogger.on_agent_action()` at the moment
the framework resolves a tool call. The same native decision is also retained
in `outputs/demo_trace.json`. No raw-text JSON or regex parser is used.

This directly demonstrates the required `{tool, arguments}` contract.

## Conversation memory demonstration

The agent maintains a message history list.

Turn 1:

> Check the weather in New Delhi.

The agent uses `get_weather`.

Turn 2:

> What about the same city tomorrow?

The second turn includes the first turn's messages in `chat_history`, so the user does not need to repeat "New Delhi". The agent can reuse the city from conversation context.

The live demo records both turns in the trace.

## Conditional workflow

`src/workflow.py` implements a workflow separate from the main agent loop.

### Step 1 — state accumulation

`RunnablePassthrough.assign` adds a classification value to the incoming state.

### Step 2 — conditional routing

`RunnableBranch` checks that classification and sends the state to either:
- a **high-priority** reply chain, or
- a **normal-priority** reply chain.

Both branches are executed by the demo:

```text
urgent input  -> high-priority branch
normal input  -> normal-priority branch
```

This satisfies the requirement that both possible routes be shown running.

## Files

```text
part4/
├── README.md
├── requirements.txt
├── .env.example
├── .gitignore
├── data/
│   └── olist_order_lookup_sample.csv
├── outputs/
│   └── demo_trace.json
└── src/
    ├── __init__.py
    ├── tools.py
    ├── agent.py
    ├── workflow.py
    └── demo.py
```

## Reproducibility

All named libraries are pinned in `requirements.txt`.

No API key, password, or other secret is included.

The compact Olist sample is derived from the files supplied for this capstone. The original large CSV files do not need to be committed because the Part 4 local tool only requires the compact lookup sample.

## Acceptance-criteria audit

| Rubric requirement | Where demonstrated |
|---|---|
| Two Python tools | `src/tools.py` |
| At least one real live API | `get_weather` in `src/tools.py` calls Open-Meteo |
| Clear/honest/atomic/safe tools | Tool docstrings + error handling in `src/tools.py` |
| Tool contract table | This README |
| Native `{tool, arguments}` routing | `NativeToolDecisionLogger.on_agent_action()` in `src/agent.py` |
| Logged at moment of call | Callback prints each resolved `{"tool": ..., "arguments": ...}` before/at tool execution |
| Three distinct tool-using tasks | `src/demo.py` |
| `@tool` decorators | `src/tools.py` |
| Tool-calling agent | `create_tool_calling_agent` in `src/agent.py` |
| Bounded executor | `AgentExecutor(max_iterations=5)` |
| Two-turn memory | `run_live()` memory demonstration in `src/demo.py` |
| `RunnablePassthrough.assign` | `src/workflow.py` |
| `RunnableBranch` | `src/workflow.py` |
| Both branch routes | `run_both_routes()` |
| No secrets | `.env` ignored; `.env.example` only |
| Exact dependencies | `requirements.txt` |

**Submission evidence rule:** `outputs/demo_trace.json` shipped with this project is clearly labeled as deterministic mock-framework evidence. Before submitting, run the **live** demonstration with your own Gemini key and commit the resulting `outputs/demo_trace.json`. Do not claim mock output as a Gemini run.

## Evidence checklist

- [x] At least two Python tools.
- [x] At least one real live external API.
- [x] Clear/honest/atomic/safe tool design.
- [x] Read/write tool contract table.
- [x] Native LangChain tool-call representation.
- [x] Explicit `{"tool": ..., "arguments": {...}}` logging.
- [x] Three distinct demonstration queries.
- [x] `@tool` decorators.
- [x] `create_tool_calling_agent`.
- [x] `AgentExecutor`.
- [x] Bounded `max_iterations`.
- [x] Two-turn conversation memory.
- [x] `RunnablePassthrough.assign`.
- [x] `RunnableBranch`.
- [x] Both conditional routes demonstrated.
- [x] Secrets loaded from environment.
- [x] Pinned dependencies.
