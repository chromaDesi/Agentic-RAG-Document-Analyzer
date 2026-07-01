# System Requirements — Agentic RAG Pipeline (n8n)

This document lists everything required to run the Query Planner → Search Agent → Guardrail → Answer Synthesis workflow.

## 1. n8n Platform

| Requirement | Details |
|---|---|
| n8n version | 1.6x or later — must support `@n8n/n8n-nodes-langchain` package v1.x and Agent node `typeVersion 3.1` |
| Deployment | **Docker**, running the official `n8nio/n8n` image (this is the personal/local deployment target for this workflow) |
| Node.js | 20.x — bundled inside the `n8nio/n8n` image, no separate install needed |
| LangChain nodes package | `@n8n/n8n-nodes-langchain` must be installed and up to date inside the container — either pre-bundled in the image tag used, or installed into the container's community nodes directory (`~/.n8n/nodes` mounted volume) if using a base image without it |
| Execution mode | Queue mode recommended if concurrent chat sessions are expected, since each run can involve 10+ chained LLM calls — for a single-user personal Docker run, standard (main) mode is fine |
| Execution timeout | Increase default workflow timeout (`EXECUTIONS_TIMEOUT` env var passed to the container) to at least 300s — the retry loop (Planner → Search → Guardrail) can run multiple passes per query |
| Persistent volume | Mount a volume to `/home/node/.n8n` so credentials, workflow data, and execution history survive container restarts/image updates |
| Environment variables | Set `EXECUTIONS_TIMEOUT`, `N8N_HOST`/`WEBHOOK_URL` (needed for the Webhook node's public URL to resolve correctly if exposed externally), and standard `DB_*` vars only if pointing n8n's own internal database at Postgres instead of the default SQLite |

## 2. External Service Accounts & API Keys

| Service | Used by | Required for |
|---|---|---|
| **Google Gemini (PaLM) API** | Google Gemini Chat Model (×4 nodes), Embeddings Google Gemini1 | LLM calls for Query Planner, Search Agent, Guardrail Agent, Answer Synthesis Agent, and generating query embeddings (`gemini-embedding-2`) |
| **Groq API** | Groq Chat Model (×4 nodes) | Fallback LLM (`llama-3.3-70b-versatile`) wired as the `needsFallback` model on every Agent node |
| **Supabase project** | Supabase Vector Store | Vector similarity search against the `documents` table via the `match_documents` RPC function |
| **PostgreSQL database** | List Documents, Query Document Rows, Postgres Chat Memory1 | Document metadata lookups, structured row queries, and persistent chat history |

Credentials must be created in n8n's Credentials manager and match the names referenced in the workflow JSON:
- `Google Gemini(PaLM) Api account`
- `Groq account`
- `Supabase account`
- `Postgres account`

## 3. Supabase / Postgres Schema

The workflow assumes the following objects already exist in the connected Postgres/Supabase database:

### `documents` table (vector store)
- Must have a vector column matching the embedding dimensions produced by `gemini-embedding-2`
- Must have a corresponding `match_documents` RPC function (used by the Supabase Vector Store node's `queryName` parameter) that performs cosine similarity search and returns chunk text + metadata (chunk id, document id, document title, section, page)

### `document_metadata` table
- Queried by the **List Documents** tool
- Should contain one row per source document with descriptive fields (title, file id, schema info for CSV/Excel sources)

### `document_rows` table
- Queried by the **Query Document Rows** tool via dynamic SQL (`$fromAI('sql_query', ...)`)
- Expected columns: `dataset_id` (matches a `document_metadata` file id) and `row_data` (`jsonb`) containing the tabular row data
- Note: the tool's generated SQL is wrapped in `SELECT * FROM (...) AS limited_result LIMIT 100` — no additional application-level limiting is needed, but the Postgres user must have `SELECT` privileges on this table

### Chat memory tables
- Postgres Chat Memory1 (used by Answer Synthesis Agent) requires n8n's standard chat memory table to exist in the connected database — n8n creates this automatically on first run if the Postgres user has `CREATE TABLE` privileges, otherwise it must be pre-provisioned manually

### Database user privileges required
- `SELECT` on `documents`, `document_metadata`, `document_rows`
- `EXECUTE` on `match_documents`
- `SELECT`/`INSERT` (and `CREATE TABLE` on first run) for chat memory persistence

## 4. Workflow-Internal Nodes

| Node type | Package |
|---|---|
| `@n8n/n8n-nodes-langchain.chatTrigger` | langchain nodes |
| `@n8n/n8n-nodes-langchain.agent` (typeVersion 3.1) | langchain nodes |
| `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | langchain nodes |
| `@n8n/n8n-nodes-langchain.lmChatGroq` | langchain nodes |
| `@n8n/n8n-nodes-langchain.memoryBufferWindow` | langchain nodes |
| `@n8n/n8n-nodes-langchain.memoryPostgresChat` | langchain nodes |
| `@n8n/n8n-nodes-langchain.outputParserStructured` | langchain nodes |
| `@n8n/n8n-nodes-langchain.vectorStoreSupabase` | langchain nodes |
| `@n8n/n8n-nodes-langchain.embeddingsGoogleGemini` | langchain nodes |
| `n8n-nodes-base.webhook` | n8n core |
| `n8n-nodes-base.postgresTool` (×2) | n8n core |
| `n8n-nodes-base.if` | n8n core |
| `n8n-nodes-base.code` (×3) | n8n core |
| `n8n-nodes-base.respondToWebhook` | n8n core |

All of the above ship with a standard n8n installation except the langchain nodes package, which must be installed separately if self-hosting (`n8n-nodes-langchain` community/official package).

## 5. Entry Points

The workflow exposes two triggers, both feeding the Query Planner:
- **Chat Trigger** — n8n's built-in chat widget/testing interface (`chatInput` field)
- **Webhook** — `POST` to path `0762a10c-f2f9-4941-8e4d-db60994aef67`, expects `body.chatInput` or `body.question`

Only one needs to be active depending on how the workflow is consumed (embedded chat UI vs. external API call). The **Respond to Webhook** node at the end only fires meaningfully when triggered via the Webhook path.

## 6. Network / Egress

The n8n instance must have outbound HTTPS access to:
- `generativelanguage.googleapis.com` (Gemini API)
- `api.groq.com` (Groq API)
- Your Supabase project's REST/RPC endpoint (`https://<project-ref>.supabase.co`)
- Your Postgres host (direct connection or via Supabase connection pooler)

**Docker-specific note:** if Postgres is also running locally in another container (rather than hosted Supabase), make sure both containers are on the same Docker network and the n8n container references the Postgres container by its service/container name (not `localhost`) in the credential's host field. If n8n is exposed only on `localhost` on the host machine, the Webhook node's public URL won't be reachable externally — use a tunnel (e.g. `ngrok`, Cloudflare Tunnel) or proper port mapping/reverse proxy if the Webhook trigger needs to be called from outside the host.

## 7. Sizing / Performance Notes

- Each end-to-end query can trigger **4 separate LLM calls** (Planner, Search, Guardrail, Answer) per attempt, and the Guardrail can force up to ~3 retry loops before force-passing on `attempt_number >= 3` — budget for up to ~12 LLM calls on a worst-case query.
- The Search Agent's tool-calling (vector search + optional SQL/document metadata lookups) adds additional latency and sub-executions; ensure `EXECUTIONS_TIMEOUT` and any reverse-proxy timeout (e.g. nginx `proxy_read_timeout`) are set well above 60s.
- If running in queue mode, allocate enough worker concurrency to avoid webhook timeouts under multi-user load, since a single execution is long-running relative to typical n8n workflows.
