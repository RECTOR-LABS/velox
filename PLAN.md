# Velox — Plan

> Timezone-anchored to the **Jun 11, 2026 2:00 PM PDT** cutoff. Today **Jun 6**. Effective working runway: **Jun 6 (partial) → Jun 11 morning**. Treat Jun 11 as **submission + buffer only**, so the real build window is **Jun 7–10 (4 focused days)**. Plan accordingly — do not leave deploy or video to the last day.

## Strategy in one line
**Get a deployed, Gemini-3-powered ADK agent talking to the MongoDB MCP on Day 1; spend the rest making it grounded, useful, and demoable.** The stack ramp is the project; the agent logic is comparatively easy for this builder.

## Day-by-day milestones

### Day 0 — Jun 6 (today, partial): Account & access groundwork
*Goal: remove every "I'm blocked on access" surprise before real work starts.*
- [ ] Create/confirm **GCP project**, **enable billing**, apply **hackathon credits**. Enable **Vertex AI API**, **Cloud Run**, **Cloud Build**, **Artifact Registry**.
- [ ] Verify **Gemini 3** access in your region (`gemini-3.1-pro-preview`) via a one-shot Vertex call; record the exact accepted model id.
- [ ] Spin up **MongoDB Atlas M0** free cluster; create db `velox`; grab the SRV connection string; set Atlas network access for the demo.
- [ ] `gcloud auth login`, `gcloud config set project`, install **ADK** (`pip install google-adk`), install Node (for `npx`).
- [ ] Register on **Devpost**, select **MongoDB** track + **Financial Services** category, read the official rules end-to-end once.
- **DoD:** one successful Gemini 3 call AND `mongosh`/driver connection to Atlas both proven from your machine.

### Day 1 — Jun 7: Hello-world deployed (the make-or-break day)
*Goal: a trivial ADK agent, wired to MongoDB MCP, running locally AND on Cloud Run.*
- [ ] Scaffold ADK project (`agent/__init__.py`, `agent.py`, `requirements.txt`, `.env`); `.env` gitignored; commit `.env.example`.
- [ ] Wire **`McpToolset` → MongoDB MCP** (`npx -y mongodb-mcp-server@latest --readOnly --telemetry disabled`), `MDB_MCP_CONNECTION_STRING` from env, `tool_filter` to read tools.
- [ ] `adk web` locally → confirm the agent **lists tools from the MCP** and can run a `find` against `velox`. *(This proves the stdio/npx bridge — the riskiest seam.)*
- [ ] First **Cloud Run deploy:** `adk deploy cloud_run --project … --region … --service_name=velox --with_ui $AGENT_PATH`; set `MDB_MCP_CONNECTION_STRING` as a Cloud Run env var.
- [ ] Hit the hosted URL, ask "list collections" — get a real answer.
- **DoD:** **public Cloud Run URL answers one question backed by live MongoDB.** If the npx-in-container bridge fails, *today* is when you pivot to remote MCP (`StreamableHTTPConnectionParams`) — do not carry this risk forward.

### Day 2 — Jun 8: Real data + grounded Q&A
*Goal: the agent answers genuine finance questions and cites documents.*
- [ ] Write **seed script**: `accounts` + `transactions` (a few accounts, ~12 months, realistic categories/merchants) with **planted anomalies** (1 duplicate charge, 1 outlier) for deterministic demos. Add a **reset** command.
- [ ] Tighten the **system instruction**: always query via tools, always cite ids/dates/amounts, never fabricate numbers.
- [ ] Validate the agent composes correct **`aggregate`** pipelines for: spend-by-category, month-over-month, merchant rollups, largest charges.
- [ ] Confirm Atlas↔Cloud Run connectivity (allowlist/egress) on the *deployed* service, not just locally.
- **DoD:** 5 canned finance questions answered correctly **on the hosted URL**, each citing real data.

### Day 3 — Jun 9: Anomaly tool + scope freeze
*Goal: the "agentic, not chatbot" differentiator lands; then freeze scope.*
- [ ] Implement **`detect_anomalies(account_id)`** native ADK function tool: pulls an aggregation via MCP, computes outliers (e.g., z-score / IQR) + duplicate-charge detection, returns flags + plain-English rationale.
- [ ] Register it on the agent; prompt the agent to call it for "anything suspicious?" intents.
- [ ] **Stretch (only if green):** thin **Next.js chat page** hitting the ADK API (builder's home turf) for a nicer demo surface. Skip without guilt if behind.
- [ ] **SCOPE FREEZE at end of day.** No new features after this.
- **DoD:** anomaly flagging works end-to-end on hosted URL and reliably catches the planted anomalies.

### Day 4 — Jun 10: Polish, README, video
*Goal: everything a judge touches is clean; video shot.*
- [ ] README: overview, **architecture diagram** (SVG), setup steps, the **Gemini 3 model-version note** + **MCP read-only / env-var** security note, "synthetic data / not financial advice."
- [ ] Repo hygiene: **MIT LICENSE visible**, secret-scan (`gitleaks`/`git grep` for URIs/keys), `.env.example` only, CI-free is fine.
- [ ] Record **~3-min demo:** problem → 2-3 grounded questions → trigger anomaly flag → show citations → one line on Gemini 3 + ADK + MongoDB MCP. Re-record once for tightness.
- [ ] Dry-run the **Devpost submission** (don't submit yet): URL, repo, video, track=MongoDB.
- **DoD:** repo public + clean, video uploaded, hosted URL stable.

### Day 5 — Jun 11 (morning, before 2:00 PM PDT): Submit + buffer
- [ ] Final smoke test of hosted URL (cold start included).
- [ ] **Submit on Devpost by ~11:00 AM PDT** — leave a 3-hour cushion for surprises.
- [ ] Confirm submission shows correct track/category and all three links resolve.
- **DoD:** submitted with confirmation, **well before** 2:00 PM PDT.

## Task breakdown (by area)
- **Infra/access (Day 0–1):** GCP project, APIs, credits, Gemini 3 region check, Atlas M0, gcloud/ADK/Node installs.
- **Agent core (Day 1–2):** ADK scaffold, `McpToolset` wiring, system instruction, model id.
- **Data (Day 2):** seed + reset scripts with planted anomalies.
- **Feature (Day 3):** `detect_anomalies` function tool.
- **Deploy (Day 1, 2, 4):** Cloud Run via `adk deploy cloud_run --with_ui`; env-var secret handling; connectivity checks.
- **Submission (Day 4–5):** README, LICENSE, secret-scan, video, Devpost.

## Setup quick-reference
```bash
# Auth + project
gcloud auth login && gcloud config set project $GOOGLE_CLOUD_PROJECT
# ADK
pip install google-adk
# .env (gitignored) — see .env.example in repo
#   GOOGLE_GENAI_USE_VERTEXAI=TRUE
#   GOOGLE_CLOUD_PROJECT=...
#   GOOGLE_CLOUD_LOCATION=us-central1
#   MDB_MCP_CONNECTION_STRING=mongodb+srv://...   ← NEVER committed; set in Cloud Run env
# Local run
adk web
# Deploy
adk deploy cloud_run --project=$GOOGLE_CLOUD_PROJECT --region=$GOOGLE_CLOUD_LOCATION \
  --service_name=velox --with_ui $AGENT_PATH
```
**Secrets policy (public repo):** real Atlas URI and any keys live ONLY in local `.env` (gitignored) and Cloud Run env vars. Repo ships `.env.example` with placeholders. MongoDB MCP connection string is passed via `MDB_MCP_CONNECTION_STRING` env — never as a CLI arg (avoids leaking creds to the LLM and to git).

## Definition of Done (project-level)
- Hosted Cloud Run URL: loads, answers ≥5 finance questions with **document citations**, and flags the planted anomalies — live, no local deps.
- Public repo: MIT LICENSE visible, README with architecture + setup + security/model notes, **no secrets** (verified by scan), `.env.example` present.
- ~3-min video demonstrating the agentic behavior + naming the required stack.
- Devpost submitted before 2:00 PM PDT with track=**MongoDB**, category=**Financial Services**, all links live.
- Sanity: read-only MCP enforced; Gemini 3 family model id confirmed accepted in-region.

## 🧭 BLUNT feasibility verdict (5-day solo)
**Feasible, but only with ruthless scope discipline — and the risk is front-loaded, not in the agent.** For this builder, the agent reasoning loop, MongoDB queries, and anomaly heuristic are *easy*; they're squarely in his fintech/agentic wheelhouse. The genuine threat is the **Google Cloud / Gemini ramp** — Vertex enablement, IAM/quota, the exact Gemini 3 model id, and especially **getting the `npx`-launched MongoDB MCP to run inside the ADK Python container on Cloud Run.** If Day 1 ends without a deployed hello-world that queries MongoDB, the timeline is in trouble and you should immediately pivot the MCP transport to remote HTTP.

Three rules make or break it:
1. **Day 1 = deployed + MCP-connected, or bust.** Treat it as a tracer bullet through the whole stack before writing any feature.
2. **Use `--with_ui`; skip the custom frontend.** A bespoke Next.js UI is the most tempting time-sink and the least rewarded by the rubric. It's stretch-only.
3. **Freeze scope after Day 3.** Days 4–5 are submission, not engineering.

**Edge worth pressing:** the **MongoDB bucket** is winnable — finance data maps naturally to documents, the MCP is read-only-safe and well-documented, and "grounded answers that cite real records + an anomaly flag" is a clean, judge-legible story across all four criteria. Concept is intentionally small so it can actually be *finished and polished*, which beats an ambitious half-built agent every time. If the Day-1 tracer lands, this ships comfortably.
