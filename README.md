# Athlete Concierge Agent

A multi-agent assistant that helps student-athletes juggle two demanding
schedules at once: their training program and their coursework. Built for
Kaggle's 5-Day AI Agents Intensive (Vibe Coding) course with Google, in the
**Concierge Agents** track.

## Problem

Student-athletes run two full-time jobs in parallel: a structured lifting/
training program with specific weights and volumes to hit, and a full
academic course load with deadlines that don't care what phase of training
they're in. Keeping both in your head -- or split across a workout tracker
and a syllabus -- means things get dropped. This agent puts both in one
place and lets you ask about either (or both) in plain language.

## Why agents (not just a dashboard)

A static spreadsheet or calendar can show you data, but it can't reason
across the two domains. This agent can be asked "plan my week" and will
pull from *both* the training program and the assignment list, then combine
them into one prioritized answer -- e.g. flagging that a heavy squat day
lands the day before an exam and suggesting a lighter session instead. That
kind of cross-domain judgment is what makes an agent worth building here
instead of a simple lookup tool.

## Architecture

```
                     ┌─────────────────────────┐
                     │   athlete_concierge      │
                     │   (root/orchestrator)    │
                     └────────────┬─────────────┘
                                  │ delegates based on intent
              ┌───────────────────┴───────────────────┐
              ▼                                        ▼
    ┌───────────────────┐                   ┌────────────────────┐
    │  training_agent     │                   │  academic_agent     │
    │  (workouts, soreness)│                   │  (deadlines, study) │
    └──────────┬──────────┘                   └──────────┬──────────┘
               │                                          │
               └───────────────┬──────────────────────────┘
                                ▼
                   ┌─────────────────────────┐
                   │  athlete-concierge MCP    │
                   │  server (FastMCP, stdio)  │
                   │  - get_todays_workout     │
                   │  - get_upcoming_deadlines │
                   │  - log_soreness           │
                   └────────────┬──────────────┘
                                ▼
                   ┌─────────────────────────┐
                   │  data/*.json (local)      │
                   └─────────────────────────┘
```

- **`athlete_concierge/agent.py`** -- the root orchestrator (`root_agent`),
  the entry point ADK looks for. Routes each request to the right sub-agent.
- **`athlete_concierge/sub_agents/training_agent.py`** -- handles workout and
  recovery questions.
- **`athlete_concierge/sub_agents/academic_agent.py`** -- handles deadline
  and study-planning questions.
- **`mcp_server/server.py`** -- a custom MCP server (built with FastMCP)
  that both sub-agents connect to over stdio. This is the "system of
  record": it reads the athlete's training program and assignment list from
  local JSON files, and writes soreness logs back out.

## Course concepts demonstrated

| Concept | Where |
|---|---|
| Multi-agent system (ADK) | `athlete_concierge/agent.py` orchestrates `training_agent` + `academic_agent` |
| MCP Server | `mcp_server/server.py`, a from-scratch FastMCP server with 3 tools |
| Security features | API key lives only in `.env` (gitignored); MCP server hardcodes its data paths rather than accepting arbitrary paths from the caller; soreness input is validated/clamped to 1-5 |
| Deployability | See "Deploying" below |
| Agent skills / CLI | Run and demoed via `adk run` / `adk web` |

## Setup

```bash
# 1. Clone and enter the repo
git clone <your-repo-url>
cd athlete-concierge-agent

# 2. Create a virtual environment
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Set up your API key
cp .env.example .env
# then edit .env and paste in your Gemini API key from https://aistudio.google.com/app/apikey
```

## Running it

```bash
# Chat UI in the browser
adk web

# or, terminal chat
adk run athlete_concierge
```

Try asking:
- "What's my workout for Day 1?"
- "What's due this week?"
- "My legs are really sore, like a 4 out of 5."
- "Help me plan my week."

## Security notes

- No API keys or credentials are hardcoded anywhere in this repo -- the only
  secret (`GOOGLE_API_KEY`) lives in a local `.env` file that is excluded via
  `.gitignore`.
- The MCP server does not accept file paths from the caller; it only ever
  reads/writes the three JSON files under `data/`, which limits what a
  malicious or malformed tool call could do.
- User-provided numeric input (soreness level) is validated and clamped to a
  safe range (1-5) before being written to disk.

## Deploying

This project runs locally via `adk web` / `adk run` for development. To take
it further, ADK agents can be containerized and deployed to **Cloud Run** (or
Vertex AI Agent Engine): package the agent with a small FastAPI wrapper,
build a container image, and deploy with `gcloud run deploy`. The MCP server
would either run as a sidecar process or be swapped for an HTTP/SSE-based
MCP connection (`SseServerParams`) instead of stdio, since Cloud Run
services don't share a local filesystem process the way stdio expects. This
project doesn't deploy live for judging purposes, per the course rubric.

## Built with

This project was built and tested in Google's **Antigravity** IDE as part of
the Kaggle 5-Day AI Agents Intensive course with Google.
