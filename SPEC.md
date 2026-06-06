# Velox — Spec

> A "swift" AI agent for the **Financial Services** track: a transaction-intelligence copilot built on **Gemini 3 + Google Cloud Agent Builder (ADK)** that reads live account/transaction data through the **MongoDB MCP Server** to surface spending insights, anomaly/fraud flags, and natural-language Q&A over a personal-finance dataset.

## ⏰ Deadline reality check — READ FIRST
**Submit by Jun 11, 2026 @ 2:00 PM PDT.** Today is **Jun 6, 2026** → **~5 days, solo.** This is the dominant constraint shaping every decision below. The plan is YAGNI-tight on purpose. The single biggest risk is **not** the agent logic — it is the **GCP/Gemini stack ramp** (Vertex enablement, IAM, Agent Builder, Cloud Run) for a builder whose muscle memory is TypeScript/Next.js + Anthropic/OpenAI. Budget Day 1 almost entirely to "hello-world deployed," not to features.

## Hackathon context (authoritative)
- **Event:** Google Cloud "Rapid Agent" Hackathon (Devpost). Online. [rapid-agent.devpost.com](https://rapid-agent.devpost.com/)
- **Required stack (all three, non-negotiable):**
  1. **Gemini 3** for reasoning,
  2. **Google Cloud Agent Builder** (Vertex AI Agent Builder + **Agent Development Kit / ADK**) for orchestration,
  3. **Model Context Protocol (MCP)** — integrate **≥1 partner's MCP server**.
- **Partner tracks (6 buckets):** Arize, Elastic, Fivetran, GitLab, MongoDB, Dynatrace. Each bucket: **1st $5K / 2nd $3K / 3rd $2K** → 18 cash slots, **$60K** total. You compete *only within the partner bucket whose MCP you used.*
- **Challenge categories (pick one):** 2026 World Cup · **Financial Services** · Brick-and-Mortar Retail.
- **Deliverables:** hosted/public **URL** + **public open-source repo with a visible LICENSE** + **~3-min demo video** + Devpost form with track selected.
- **Judging:** Technological Implementation · Design · Potential Impact · Quality of the Idea.

**Our lane:** Category **Financial Services** (aligns with the builder's fintech/DeFi depth) × partner bucket **MongoDB** (its MCP server is mature, npx-runnable, and read-only-capable — the lowest-friction "real data" path for a finance agent). This is a deliberate edge: 18 slots across 6 buckets means **bucket choice is strategy** — MongoDB's MCP is well-documented and finance data maps cleanly to documents/collections.

## Problem & target user
Retail-banking / fintech customers (and the support agents serving them) can't get straight answers from their own transaction history. "Why was last month higher?" "Any duplicate or suspicious charges?" "How much did I spend on subscriptions this quarter?" today means dashboards, CSV exports, or a ticket. **Velox** is a conversational agent that answers those questions directly against a live transactions store, citing the actual records, and proactively flags anomalies — the kind of "agentic, not chatbot" experience the hackathon explicitly rewards.

**Primary user:** an individual account holder asking NL questions about their finances.
**Secondary user:** a support/ops agent triaging a flagged account.

## Concept

### ✅ Recommended: "Velox — Transaction Intelligence Agent" (Financial Services × MongoDB MCP)
A single ADK `LlmAgent` (Gemini 3) whose **only** external toolset is the **MongoDB MCP Server in read-only mode**, pointed at an Atlas cluster seeded with a synthetic `accounts` + `transactions` dataset. The agent:
- answers NL questions by composing MongoDB **`find` / `aggregate` / `count`** queries via MCP tools,
- runs a lightweight **anomaly heuristic** (statistical outliers + duplicate-charge detection) as a native ADK function tool over aggregation results,
- **cites the underlying documents** (ids, dates, amounts) in every answer so judges see grounded, verifiable output — not hallucinated numbers.

Why this wins on the rubric: *Technological Implementation* (real MCP + real DB + Gemini 3 tool-use, not a toy), *Quality of Idea* (clear finance use-case), *Design* (a clean chat UI + "here's the data" transparency), *Impact* (obvious productization path). Crucially it is **buildable in 5 days** because the data layer is "just" a read-only MCP and the agent is one well-scoped reasoning loop.

### Alternatives (briefly, not chosen)
- **Loan pre-qualification triage** (Financial Services × MongoDB MCP): agent reads applicant + bureau-like docs and explains an eligibility decision. Higher impact narrative but needs a defensible scoring policy → more design surface than 5 days allows. *Fallback if anomaly detection proves flaky.*
- **Elastic bucket** (transaction search/semantic fraud search via Elastic MCP): equally on-theme for finance; chosen against only because MongoDB's MCP has the smoother npx + read-only + Atlas-free-tier story. Keep as the **pivot** if the MongoDB MCP integration stalls — the ADK wiring is identical, only the `McpToolset` server command changes.

## MVP features (YAGNI-tight)
1. **One agent, one toolset.** ADK `LlmAgent` on Gemini 3 with a single `McpToolset` → MongoDB MCP (read-only). No multi-agent, no A2A, no memory service.
2. **Grounded NL Q&A** over `transactions`: spend-by-category, period comparisons, merchant rollups, "largest/duplicate charges." Every answer cites document fields.
3. **Anomaly flagging tool:** native ADK function tool that takes an account id, pulls an aggregation via MCP, and returns outliers + suspected duplicates with a plain-English rationale.
4. **Hosted chat UI:** ADK's built-in dev UI (`--with_ui`) on Cloud Run is the MVP front door — zero custom frontend needed to satisfy "hosted public URL." *(Optional stretch: a thin Next.js chat page hitting the ADK API — the builder's home turf — only if Days 1–3 land early.)*
5. **Seed + reset script** so judges (and the demo) hit deterministic data.

**Explicitly deferred:** auth/multi-tenant, write-back actions, streaming voice, Arize/Dynatrace observability, fine-tuning, RAG over docs.

## Architecture (grounded in current docs)

```
User ──chat──▶ ADK dev UI (Cloud Run, --with_ui)
                     │
                     ▼
        ADK LlmAgent  (model: gemini-3.1-pro-preview)   ← Gemini 3 family via Vertex AI
          ├─ native tool: detect_anomalies()  (Python)
          └─ McpToolset ──stdio/npx──▶ MongoDB MCP Server (--readOnly)
                                              │  MDB_MCP_CONNECTION_STRING (env)
                                              ▼
                                   MongoDB Atlas (M0 free tier)
                                   db: velox  ·  cols: accounts, transactions
```

**Agent + MCP wiring (verbatim-faithful to ADK docs).** ADK's `McpToolset` discovers a server's tools and adapts them into ADK `BaseTool`s for the `LlmAgent`. Real imports/usage per [adk.dev/tools-custom/mcp-tools](https://adk.dev/tools-custom/mcp-tools/):

```python
from google.adk.agents import LlmAgent
from google.adk.tools.mcp_tool import McpToolset
from google.adk.tools.mcp_tool.mcp_session_manager import StdioConnectionParams
from mcp import StdioServerParameters

root_agent = LlmAgent(
    model="gemini-3.1-pro-preview",          # Gemini 3 family on Vertex; see model note
    name="velox_finance_agent",
    instruction="You answer questions about the user's accounts and transactions. "
                "Always query the database via tools and cite document ids/dates/amounts. "
                "Never invent figures.",
    tools=[
        McpToolset(
            connection_params=StdioConnectionParams(
                server_params=StdioServerParameters(
                    command="npx",
                    args=["-y", "mongodb-mcp-server@latest", "--readOnly", "--telemetry", "disabled"],
                    # MDB_MCP_CONNECTION_STRING is read from the process env (NOT passed as an arg)
                ),
            ),
            # least-privilege: expose only the read tools we use
            tool_filter=["find", "aggregate", "count", "collection-schema",
                         "list-databases", "list-collections"],
        ),
        # detect_anomalies registered alongside as a native FunctionTool
    ],
)
```

**MongoDB MCP Server** ([mongodb-js/mongodb-mcp-server](https://github.com/mongodb-js/mongodb-mcp-server), [docs](https://www.mongodb.com/docs/mcp-server/configuration/options/)):
- Run via `npx -y mongodb-mcp-server@latest --readOnly`. Exposes **23 database tools + 15 Atlas tools** (plus local/assistant tools); we only need the read subset (`find`, `aggregate`, `count`, `collection-schema`, `list-databases`, `list-collections`, `explain`).
- **Connection string via `MDB_MCP_CONNECTION_STRING` env var — never as a CLI/positional arg** (docs: passing it at runtime "expose[s] the connection credentials to the large language model"). Public-repo rule: **only an env-var reference ships; the real Atlas URI lives in Cloud Run env / local `.env` (gitignored).**
- **`--readOnly` is mandatory here** (docs literally advise "always enable read-only mode") — guarantees the agent can never mutate/delete data, which also removes a whole class of demo-day risk. Disable telemetry with `--telemetry disabled`.

**Gemini 3 model note (important, verified):** `gemini-3-pro-preview` was **discontinued 2026-03-26**; current Pro member of the Gemini 3 family is **`gemini-3.1-pro-preview`** (GA-track on Vertex AI, Feb 2026). Use that as the primary model id; `gemini-flash-latest` / a Gemini 3 Flash id is the cheaper fallback for cost/latency. The hackathon says "Gemini 3" as a *family* requirement, which 3.1 satisfies. Sources: [Vertex model docs](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/models/gemini/3-1-pro), [model lifecycle](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/model-versions). **Confirm the exact accepted id in your region on Day 1 — model ids drift.**

**Hosting — Cloud Run via ADK CLI** ([adk.dev/deploy/cloud-run](https://adk.dev/deploy/cloud-run/)):
```bash
adk deploy cloud_run \
  --project=$GOOGLE_CLOUD_PROJECT \
  --region=$GOOGLE_CLOUD_LOCATION \
  --service_name=velox \
  --with_ui \
  $AGENT_PATH
```
Project layout is the ADK-standard `agent/ {__init__.py, agent.py, requirements.txt, .env}` with:
```
GOOGLE_GENAI_USE_VERTEXAI=TRUE
GOOGLE_CLOUD_PROJECT=<id>
GOOGLE_CLOUD_LOCATION=us-central1
# MDB_MCP_CONNECTION_STRING set as a Cloud Run env var (secret), NOT committed
```
`--with_ui` gives the hosted public chat URL with zero frontend code. *(Vertex AI Agent Engine via `adk deploy agent_engine` is the alternative managed runtime; Cloud Run chosen for speed + a visible web URL judges can click — [Agent Engine quickstart](https://cloud.google.com/agent-builder/agent-engine/quickstart-adk) kept as backup.)*

**Cost:** GCP free tier + hackathon credits for Vertex/Gemini calls; Atlas **M0 free** cluster; Cloud Run scales to zero. Target spend **~$0**. No blockchain (per rules).

## Non-goals
- No write/mutate operations against MongoDB (read-only by construction).
- No auth, multi-tenant, or PII handling (synthetic data only).
- No multi-agent orchestration / A2A, no custom RAG, no fine-tuning.
- No bespoke design system — ADK dev UI is acceptable for MVP; custom Next.js UI is *stretch only*.
- No second partner MCP (one bucket = one strategy; adding more dilutes focus and risks judging-bucket ambiguity).

## Risks & unknowns
| Risk | Likelihood | Mitigation |
|---|---|---|
| **GCP/Gemini stack ramp eats Day 1–2** (Vertex enablement, IAM/quota, Agent Builder, billing) | **High** | Day-1 goal is *deployed hello-world agent*, not features. Use `adk deploy cloud_run --with_ui` to skip frontend. Have credits/billing enabled before anything. |
| **MCP-in-Cloud-Run quirk:** MongoDB MCP launches via `npx` (Node) inside the ADK (Python) container; the deployed image must have Node available for the stdio subprocess | **Med-High** | Validate **locally first** (`adk web`) so the stdio bridge is proven before deploy. If the Cloud Run image lacks Node, switch the toolset to a **remote MCP** (`StreamableHTTPConnectionParams`) pointing at a separately-run MCP, or pivot MCP to a managed/HTTP variant. *This is the single most likely thing to bite — derisk Day 2.* |
| **Exact Gemini 3 model id / region availability** | Med | Confirm accepted id on Day 1; keep Flash fallback; `GOOGLE_GENAI_USE_VERTEXAI=TRUE`. |
| **Atlas IP allowlist / network** blocks Cloud Run egress | Med | Use Atlas "allow from anywhere" for the demo cluster (synthetic data) or configure egress; test connectivity Day 2. |
| **Anomaly tool produces weak/false results** on synthetic data | Med | Seed data with *planted* anomalies (a duplicate charge, an outlier) so detection is deterministic and demos cleanly; fall back to the loan-triage concept if needed. |
| **Credentials leaking to the LLM** via connection string | Low (by design) | Never pass URI as MCP arg; env var only; `--readOnly`. |
| **Solo bandwidth over 5 days** | High | Hard scope freeze after Day 3; Days 4–5 are *demo + polish + submission*, not new features. |

## Submission checklist
- [ ] **Hosted public URL** — Cloud Run service (`adk deploy cloud_run --with_ui`), reachable, loads chat, answers a seeded question live.
- [ ] **Public open-source repo** with a **visible LICENSE** (MIT), README (setup + architecture diagram + the model/MCP notes above), **zero secrets** (env-var references only; `.env` gitignored; sample `.env.example`).
- [ ] **~3-min demo video** — problem → ask Velox 2-3 finance questions → trigger an anomaly flag → show it citing real MongoDB docs → 1 line on stack (Gemini 3 + ADK + MongoDB MCP).
- [ ] **Devpost form** — **track = MongoDB**, category = Financial Services, links to URL + repo + video.
- [ ] **License file visible**, README states it's synthetic data / not financial advice.

---

**Doc sources (verified current, Jun 2026):**
- Hackathon: https://rapid-agent.devpost.com/
- ADK MCP tools: https://adk.dev/tools-custom/mcp-tools/ · https://google.github.io/adk-docs/mcp/
- ADK Cloud Run deploy: https://adk.dev/deploy/cloud-run/
- MongoDB MCP Server: https://github.com/mongodb-js/mongodb-mcp-server · https://www.mongodb.com/docs/mcp-server/configuration/options/
- Gemini 3.1 model id + lifecycle: https://docs.cloud.google.com/vertex-ai/generative-ai/docs/models/gemini/3-1-pro · https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/model-versions
