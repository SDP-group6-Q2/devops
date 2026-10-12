# AROL Customer Platform

A platform for AROL's customers to monitor their machine fleet (telemetry, alarms, maintenance),
manage commercial documents (quotes, orders) and browse machine manuals — plus an AI assistant
that can answer questions across all of that data, scoped to what the asking user is allowed to
see.

This repository is the integration project: it wires together four independently developed
services (each a git submodule, see [Repository layout](#repository-layout)) and provides the
Docker Compose setup, shared database and environment configuration to run them together.

## Modules

| Module | Role | Stack |
| --- | --- | --- |
| [`frontend/`](frontend) | Single-page app: fleet dashboards, commercial documents, manuals, chat UI. | React, Vite |
| [`backend/`](backend) | REST API: auth, tenancy, machines/telemetry/alarms/maintenance, quotes/orders, manuals, conversations. The only service the frontend talks to. | FastAPI, SQLAlchemy, PostgreSQL |
| [`ai-orchestrator/`](ai-orchestrator) | The AI assistant: one LangGraph agent that answers technical and commercial questions. Holds no data itself. | LangGraph, Anthropic Claude |
| [`mcp/`](mcp) | MCP server exposing the backend API to the agent as tools, plus a manuals RAG (vector + full-text search over manual PDFs). | FastMCP, pgvector |

Each module's own README has the detail (endpoints, tool list, internals, module-specific env
vars); this one covers how they fit together and how to run the whole stack.

## Architecture

```
 Browser
    │
    ▼
 frontend (Vite, :5173)             run separately (npm run dev), not containerized
    │  REST + JWT
    ▼
 backend "api" (FastAPI, :8000) ──────────────┐
    │  JWT                                     │  owns the "app" database
    ▼                                          ▼
 ai-orchestrator (LangGraph, :8001)   PostgreSQL ("db": app + rag databases, pgvector)
    │  MCP (streamable HTTP) + JWT              ▲
    ▼                                           │  owns the "rag" database
 mcp-server (FastMCP, :9000) ───────────────────┘
    │  REST + JWT (calls back into the backend)
    ▼
 backend "api"

 Supabase storage bucket ("manuals") ← read by backend (signed URLs) and mcp-server (RAG ingestion)
```

**How a request flows:**

- The browser talks only to the **frontend**, which calls the **backend**'s REST API directly
  (`http://localhost:8000` in development) using a JWT obtained from `POST /auth/jwt/login`.
- For the AI assistant, the frontend calls `POST /assistant/` on the **backend**. The backend
  forwards the question, conversation history and the *user's own JWT* to the
  **ai-orchestrator**'s `POST /chat`.
- The orchestrator's agent calls tools on the **mcp-server** over MCP (streamable HTTP), still
  carrying the user's JWT in the `Authorization` header.
- Each MCP tool calls the corresponding **backend** REST endpoint with that same JWT, so every
  piece of data the agent can see is governed by the same per-company, per-visibility-tier rules
  that apply to the frontend. Neither the orchestrator nor the MCP server hold any credentials or
  data of their own.
- The MCP server additionally runs manuals retrieval (vector + full-text search) against its own
  `rag` database, built by ingesting manual PDFs from the Supabase bucket.
- `backend` and `mcp-server` share one PostgreSQL instance but use **separate databases**
  (`app` and `rag`) so the RAG index stays independent of the platform's transactional data.

## Repository layout

Each service lives in its own repository, included here as a **git submodule**:

```
backend/          submodule → SDP-group6-Q2/backend
frontend/         submodule → SDP-group6-Q2/frontend
ai-orchestrator/  submodule → SDP-group6-Q2/ai-orchestrator
mcp/              submodule → SDP-group6-Q2/mcp
data/             sample dataset + manual PDFs, mounted read-only into containers (gitignored)
docker-compose.yml
.env.example
```

Clone with submodules, or initialize them afterwards:

```bash
git clone --recurse-submodules <this-repo-url>
# or, if already cloned:
git submodule update --init --recursive
```

## Configuration

Copy the root template and fill it in:

```bash
cp .env.example .env
```

| Variable | Purpose |
| --- | --- |
| `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` | Credentials/database for the shared Postgres container. |
| `RAG_DB` | Name of the second database (manuals index), created on first Postgres start. |
| `JWT_SECRET` | Signing secret for backend-issued access tokens. |
| `FIRST_SUPERUSER_EMAIL`, `FIRST_SUPERUSER_PASSWORD`, `FIRST_SUPERUSER_USERNAME` | Bootstrapped admin account, created by the backend on startup. |
| `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY` | Shared Supabase project used for manual PDF storage (backend) and RAG ingestion (mcp-server). |
| `ANTHROPIC_API_KEY`, `ANTHROPIC_MODEL` | Claude credentials/model used by the ai-orchestrator agent. |
| `REFERENCE_DATE` | "Today" the agent reasons relative dates from, pinned inside the sample dataset's date window. |
| `LANGSMITH_*` | Optional tracing for the assistant (off by default). |

See each module's README for additional, module-local variables (e.g. the backend's
`.env.example`), which Compose does not need to know about.

## Running

Requires Docker and Docker Compose.

```bash
docker compose up -d --build     # db, api, mcp-server, orchestrator
```

This starts:

- `db` — PostgreSQL with `pgvector`, creating the `app` and `rag` databases on first run.
- `api` (backend) — runs migrations, bootstraps the first superuser, and (if `./data` holds the
  sample dataset `.xlsx`) seeds machines/telemetry/quotes/orders and links manuals.
- `mcp-server` — starts immediately and indexes manuals from Supabase (or `./data/manuals` if
  mounted) in the background.
- `orchestrator` — the LangGraph agent, waiting on `mcp-server`.

The backend is then reachable at `http://localhost:8000` (`/docs` for the API reference), and
Postgres at `localhost:5432`.

The **frontend is not part of the Compose stack**; run it separately against the running backend:

```bash
cd frontend
npm install
npm run dev     # http://localhost:5173, calls VITE_API_BASE_URL (http://localhost:8000)
```

To load the sample fleet dataset and/or manual PDFs, drop them into `./data/` (see its contents
for the expected names) before `docker compose up` — the backend and mcp-server mount it
read-only and pick it up automatically.

Stop everything with `docker compose down` (add `-v` to also drop the database volume).
