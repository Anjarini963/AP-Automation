# CLAUDE.md — Agent Session Anchor

> Quick-reference working instructions for AI coding agents on this project.
> **For full scope, architecture, data model, and phase plan, read [SPEC.md](SPEC.md).** Re-read it whenever scope, stack, or design decisions come up.

## What this project is

End-to-end Accounts Payable automation system: vendor invoices flow from email arrival → extraction → validation → three-way match → Slack approval → posting in a mocked ERP. Three named AI agents handle judgment work (Vendor Research, Discrepancy Investigator, Cash Flow Chat). ZATCA Phase 2 (Saudi e-invoicing) aware. Built as a portfolio-grade product, not a tech demo.

## Current phase

**Phase 1 — Foundations.** VPS provisioning, Docker stack, Postgres schema + migrations, mock data seeding (20 vendors / 50 POs / 30 GRs, Saudi-aware), FastAPI scaffold, `/extract` endpoint using Claude vision on sample PDFs.
**Deliverable:** curl a PDF → clean structured JSON back.
*(Update this section as phases progress, with explicit human approval. Phases 2–6 in SPEC.md §6.)*

## Stack at a glance

- **Orchestration:** n8n, self-hosted via Docker on a Contabo VPS (Ubuntu 24.04).
- **Backend:** FastAPI (Python 3.12) — **single source of truth** for extraction, validation, three-way matching, agent logic. n8n and Streamlit both call its endpoints; logic is never duplicated.
- **LLM:** Claude (Anthropic API).
- **DB:** PostgreSQL via Docker, with pgvector (also serves as the vector store).
- **Dashboard:** Streamlit.
- **Approvals:** Slack interactive messages. **Notifications:** Slack + SMTP + Telegram.
- **Containerization:** Docker + docker-compose.

## Working principles (non-negotiable)

- **Reference SPEC.md** for any architectural or scope decision. If a request conflicts with the spec, flag the conflict before proceeding — do not silently choose.
- **No new dependencies, features, or tools** without flagging them and explaining the trade-off.
- **Ask before guessing.** When a design choice is ambiguous, ask.
- **Show commands before executing** for any infrastructure work — SSH, Docker, server config, DB migrations, anything touching production state.
- **Never auto-approve destructive operations:** `rm -rf`, `DROP TABLE`, `TRUNCATE`, force pushes, etc. Always require explicit human confirmation.
- **Prefer small, reversible changes.** If a task requires more than ~5 file changes, propose a plan first and wait for approval.
- **SPEC.md is a living document** — propose updates when meaningful decisions are made, but never edit it without explicit approval. Same rule for the "Current phase" section here.

## Code & repo conventions

- **Python:** type hints on all function signatures. `ruff` for lint, `black` for format (line length 100).
- **SQL:** snake_case tables/columns. Foreign keys named `<table>_id`. Every table has `created_at`; mutable tables also have `updated_at`.
- **Commits:** conventional commits — `feat:`, `fix:`, `chore:`, `docs:`, `refactor:`, `test:`. Scope encouraged: `feat(extraction): add ZATCA XML parser`.
- **Branches:** `feature/<short-description>` off `main`. Avoid force-pushing.
- **Secrets:** never commit `.env`. Use `.env.example` as the template. Never hardcode keys, passwords, or tokens.

## Repo layout

```
ap-automation/
├── backend/         # FastAPI service
├── n8n/             # n8n workflow exports and config
├── database/        # Postgres init scripts and migrations
├── dashboard/       # Streamlit app
├── scripts/         # Helper scripts (mock data, email test harness)
├── test-invoices/   # Sample invoices for testing
├── docs/            # Architecture diagrams, extra docs
├── docker-compose.yml
├── .env.example
├── README.md
├── CLAUDE.md        # This file
└── SPEC.md          # Full project context
```

## The three agents (one-line each — full detail in SPEC.md §3)

1. **Vendor Research Agent** — new vendor arrives → web-searches CR/VAT/website → drafts a vendor record for human review.
2. **Discrepancy Investigator Agent** — three-way match fails or fraud flag fires → analyzes invoice vs PO vs GR vs vendor history → drafts an investigation note + draft email (does not send).
3. **Cash Flow Chat Agent** — user-invoked from dashboard → conversational Q&A over finance views (cash forecast, discount opportunities, what-ifs).

Every agent invocation is logged to the `agent_runs` table.

## Session checklist

At the start of every session: **read SPEC.md**, check the "Current phase" section here, then proceed.
