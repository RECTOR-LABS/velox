<!-- Satellite context file — extends the global hub (~/.claude/CLAUDE.md | ~/.pi/agent/AGENTS.md). Host-neutral; project-specific only. Do not duplicate hub standards here. -->

# Velox

> A swift Gemini 3 financial-services agent on Google Cloud Agent Builder (ADK), wired to the MongoDB MCP Server — grounded NL Q&A over transactions, with anomaly flagging.

**Status:** 📐 Pre-build spec — see `SPEC.md` and `PLAN.md`.

| | |
|---|---|
| **Hackathon** | Google Cloud "Rapid Agent" (Devpost) |
| **Submit by** | ⚠️ Jun 11, 2026 2:00 PM PDT (tight) · $60K (18 slots) |
| **Lane** | Financial Services category × MongoDB partner bucket |
| **Stack** | Python · ADK · Gemini 3 (`gemini-3.1-pro-preview`) · MCP · MongoDB · Cloud Run |

## Structure

`agent/` · `scripts/` · `tests/` · `SPEC.md` (design) · `PLAN.md` (build plan) · `README.md` · `LICENSE`.

## Notes

- Pre-build — read `SPEC.md` and `PLAN.md` before implementing.
- The differentiator: grounded NL Q&A (MongoDB MCP) + anomaly flagging, not free-form chat. Gemini 3 on ADK + Cloud Run.