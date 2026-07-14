# AI-First CRM — HCP Module: Log Interaction Screen

An AI-first Log Interaction screen for pharma field reps, letting them log
Healthcare Professional (HCP) interactions either through a **structured
form** or a **conversational chat interface** backed by a **LangGraph**
agent running on **Groq** (`gemma2-9b-it`, with `llama-3.3-70b-versatile`
as a fallback for harder reasoning steps).

Both entry points write through the *same* backend tool (`log_interaction`),
so the data model, summarization, and entity extraction stay consistent
regardless of how the rep chooses to log.

## Architecture

```
frontend/  React + Redux Toolkit UI (Google Inter font)
  ├─ StructuredForm.jsx   → POST /api/interactions
  └─ ChatInterface.jsx    → POST /api/chat  (talks to the LangGraph agent)

backend/   FastAPI
  ├─ routers/interactions.py   REST CRUD for the structured form
  ├─ routers/chat.py           Chat endpoint, session-scoped conversation
  ├─ agent/graph.py            LangGraph StateGraph (agent ↔ tools loop)
  ├─ agent/tools.py            5 tools the agent can call
  ├─ agent/llm.py              Groq client wrapper (gemma2-9b-it + fallback)
  └─ models.py                 SQLAlchemy models (HCP, Interaction)
```

### Why LangGraph here

The agent needs to hold a short conversation, decide *which* of several
actions a rep's message maps to, call a tool, then react to that tool's
result before replying (e.g. "log this, then also schedule a follow-up").
LangGraph's `StateGraph` gives an `agent → tools → agent` loop (via
`should_continue`/`ToolNode`) that naturally supports this multi-step,
tool-calling behavior instead of a single one-shot LLM call.

## The 5 LangGraph Tools

| # | Tool | Purpose |
|---|------|---------|
| 1 | **`log_interaction`** | Creates a new interaction. Takes the rep's raw free-text (typed or from chat), sends it to the Groq LLM to produce a **summary**, extract **products discussed** and **topics**, and infer **sentiment** — then persists a structured `Interaction` row. |
| 2 | **`edit_interaction`** | Updates a field (notes, type, sentiment, products, follow-up) on an already-logged interaction. Every edit is appended to an `edit_history` JSON audit trail. If `notes` is edited, the summary is regenerated. |
| 3 | **`get_hcp_history`** | Pulls an HCP's most recent logged interactions so the agent (or rep) has context, e.g. "what did we discuss with Dr. Sharma last time?" |
| 4 | **`schedule_follow_up`** | Sets/updates a follow-up date and reminder note on a specific interaction. |
| 5 | **`generate_call_prep_brief`** | Summarizes an HCP's recent history and suggests 2–3 talking points ahead of the rep's next visit — an AI-generated call-prep briefing. |

### `log_interaction` in detail
The tool sends the raw notes to Groq with a system prompt asking for a
JSON object (`summary`, `products_discussed`, `topics`, `sentiment`). The
parsed JSON is used to populate the `Interaction` row's structured columns
while the original raw text is preserved in `notes`. This means a rep can
type or *say* one paragraph and get a filled-out CRM record.

### `edit_interaction` in detail
Takes `interaction_id`, `field`, `new_value`. Looks up the row, records
`{timestamp, field, old_value, new_value}` in `edit_history`, applies the
change, and — if the edited field is `notes` — asks the LLM to
re-summarize so the summary never goes stale relative to the raw text.

## Role of the LangGraph Agent

The agent is the single conversational entry point for the chat tab. It:
1. Receives the rep's message plus lightweight context (`hcp_id`, `rep_id`).
2. Decides, via Groq's tool-calling, whether the message describes a new
   visit (→ `log_interaction`), a correction (→ `edit_interaction`), a
   history lookup (→ `get_hcp_history`), a reminder (→ `schedule_follow_up`),
   or a prep request (→ `generate_call_prep_brief`) — or needs no tool at all
   (plain clarifying question).
3. Executes the tool call via LangGraph's `ToolNode`, loops back to the
   agent node with the tool's result, and produces a natural-language reply
   summarizing what was done.
4. Maintains a short per-session message history (in `chat.py`) so the rep
   can have a multi-turn conversation ("actually change that to negative
   sentiment") without repeating context.

## Running Locally

### 1. Database
Create a Postgres or MySQL database named `hcp_crm`, then set `DATABASE_URL`
in `backend/.env` (see `.env.example` for both formats).

### 2. Backend
```bash
cd backend
python -m venv venv && source venv/bin/activate   # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env         # then fill in GROQ_API_KEY and DATABASE_URL
python seed.py                # creates 3 sample HCPs
uvicorn app.main:app --reload --port 8000
```
API docs available at `http://localhost:8000/docs`.

### 3. Frontend
```bash
cd frontend
npm install
npm start
```
Runs at `http://localhost:3000` (proxies API calls to `localhost:8000`).

### 4. Get a Groq API key
Create a key at [console.groq.com](https://console.groq.com/) and set it as
`GROQ_API_KEY` in `backend/.env`. Default model is `gemma2-9b-it`;
`llama-3.3-70b-versatile` is used automatically as a fallback if a call to
the primary model fails.

## Tech Stack
- **Frontend:** React, Redux Toolkit, Google Inter font
- **Backend:** Python, FastAPI
- **AI Agent Framework:** LangGraph
- **LLM:** Groq — `gemma2-9b-it` (primary), `llama-3.3-70b-versatile` (fallback)
- **Database:** MySQL / PostgreSQL via SQLAlchemy

## Notes / Assumptions
- Auth is stubbed (`rep_id` passed as a plain string) — production would add
  JWT-based auth and a `User`/`Rep` table.
- Chat session history is stored in-memory per `session_id`; production
  would persist this (Redis, or a `chat_sessions` table) for durability.
- The structured form and chat interface both call the same
  `log_interaction` tool so extraction/summarization logic lives in one
  place, per the "AI-first" brief.
