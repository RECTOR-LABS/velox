# Velox

> A swift Gemini 3 financial-services agent on Google Cloud Agent Builder (ADK), wired to the MongoDB MCP Server — grounded NL Q&A over transactions, with anomaly flagging.

**Status:** 📐 Pre-build spec — see [`SPEC.md`](./SPEC.md) and [`PLAN.md`](./PLAN.md).

| | |
|---|---|
| **Hackathon** | Google Cloud "Rapid Agent" (Devpost) |
| **Submit by** | ⚠️ Jun 11, 2026 2:00 PM PDT (tight) · $60K (18 slots) |
| **Lane** | Financial Services category × MongoDB partner bucket |
| **Stack** | Python · ADK · Gemini 3 (`gemini-3.1-pro-preview`) · MCP · MongoDB · Cloud Run |

*Velox* (Latin "swift"): a fast, rapid agent.

## Notes
- **Public repo.** No secrets in source — env-var only. The MongoDB connection string is passed via `MDB_MCP_CONNECTION_STRING` (never a CLI arg), and the MCP runs `--readOnly`.
- Standalone (no blockchain; Google Cloud stack).

## License
MIT — see [`LICENSE`](./LICENSE).
