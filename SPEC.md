# AP Automation System with AI Agents — Full Project Context



## Career goal driving this project

Break into AI automation engineering, with a longer-term move into fintech automation roles or product management within finance-tech. The intentional positioning is "someone who bridges deep finance understanding with automation engineering skill" — rare in a market where most automation engineers don't understand finance, and most finance people can't build. This project is the flagship portfolio piece supporting that positioning.

Target employers: GCC B2B fintechs (Tabby, Tamara, Hala, Lean Technologies, Foodics), bank automation/digital-transformation teams, RPA consulting practices at the Big Four (Deloitte, EY, KPMG, PwC), and corporate finance-tech functions at mid-to-large companies.

A UiPath version of the same system is planned as a follow-up project after the n8n version ships, to demonstrate enterprise RPA skills alongside modern AI automation. This spec covers only the n8n version.

## What the system is, in one paragraph

An end-to-end Accounts Payable automation system that takes vendor invoices from email arrival to approved-and-posted in a mocked ERP. Built on n8n (self-hosted on a Contabo VPS) with a FastAPI backend service handling LLM extraction and agent logic. Includes three named AI agents that handle judgment-heavy work normally done by humans: a Vendor Research Agent, a Discrepancy Investigator Agent, and a Cash Flow Chat Agent. Includes ZATCA Phase 2 awareness for Saudi e-invoicing. Designed and presented as a product, not a tech demo — with tier-based product thinking, finance reports a CFO would actually read (aged payables, cash forecasts), and an honest production-gap analysis.

## Why these features specifically

Accounts Payable is the most-automated business process in the world, and the backlog is still enormous because every company's vendors, formats, approval rules, and ERPs are different. AP automation is a flagship use case for AI automation engineers, fintech companies, BPO/shared-services firms, and bank automation teams. In Saudi Arabia, ZATCA Phase 2 e-invoicing compliance has created additional regional demand for AP systems that can ingest both legacy PDFs and ZATCA-cleared XML invoices.

The agent layer (three named, purposeful agents rather than generic LLM calls scattered through the workflow) is the differentiating feature. Most AP automation portfolios show a workflow with "an LLM somewhere"; this one shows three named workers with explicit jobs, tools, and outputs.

The finance-controls layer (three-way matching, segregation of duties, audit trail, working-capital optimization, fraud detection) is what makes the project credible as the work of someone who understands finance, not just someone who can wire APIs together. These are finance master's concepts implemented as automation.

---

## 1. Architecture

Six pipeline layers, three AI agents, two cross-cutting modules.

### Pipeline layers

1. **Capture** — email inbox watcher and folder watcher pick up new invoices.
2. **Extraction** — three input paths converge to unified JSON: ZATCA XML (parsed directly), PDF (Claude with vision), image (Claude with vision).
3. **Validation & Enrichment** — math checks, duplicate detection, vendor lookup, ZATCA QR-code verification.
4. **Three-way matching** — match invoice against PO and goods receipt, score, route to auto-pass / needs-review / exception.
5. **Approval routing** — amount-based thresholds (under 1,000 SAR auto-approve; 1,000-10,000 SAR manager approval; above 10,000 SAR finance director approval) sent via Slack interactive messages with approve/reject buttons. 48-hour escalation timer.
6. **Posting & Reporting** — on approval, write to the "ERP" (Postgres), schedule payment, run fraud checks, archive source file, update dashboard.

### Cross-cutting modules

- **Working-capital optimization** — nightly job recommending payment timing based on cash position (mocked), due dates, and discount terms. Uses the implied-APR-of-discount calculation (e.g., 2/10 net 30 → 37% annualized).
- **Fraud detection** — runs on every invoice during validation. Scores risk based on: round amounts, new vendors with bank details matching existing vendors, abnormal volume spikes, duplicate invoice numbers, suspicious vendor name similarity.

---

## 2. Tech stack

| Component | Choice |
|---|---|
| Workflow orchestration | n8n, self-hosted via Docker on VPS |
| LLM | Claude (Anthropic API) — builder has Max subscription |
| Backend service | FastAPI (Python 3.12) |
| Database | PostgreSQL via Docker, with pgvector extension |
| Vector store | Postgres pgvector (avoids running a separate vector DB) |
| Frontend / Dashboard | Streamlit |
| Approval UI | Slack interactive messages |
| Notifications | Slack (approvals), SMTP (vendor emails), Telegram (system pings) |
| Hosting | Contabo Cloud VPS 10 (4 vCPU, 8 GB RAM, NVMe), Ubuntu 24.04, Frankfurt or Nuremberg datacenter |
| Containerization | Docker + docker-compose |
| Version control | Public GitHub repo, conventional commits, feature branches per module |

**Architectural rule:** the FastAPI service is the single source of truth for extraction, validation, three-way matching, and agent logic. n8n calls FastAPI endpoints. The Streamlit dashboard calls FastAPI endpoints. Logic is not duplicated between orchestration and UI.

---

## 3. The agent layer — three named agents

Each agent is built with LangGraph or n8n's AI Agent nodes (decide during implementation, document the choice). Each has a name, a defined job, a defined toolset, and a defined output.

### Agent 1: Vendor Research Agent

- **Triggered when:** an invoice arrives from a vendor not in the master list.
- **Job:** searches the web for the vendor's commercial registration, official website, and business profile. For Saudi vendors, attempts to find VAT number and CR (Commercial Registration) number from public sources. Fetches and parses the vendor's website for contact info, business category, and legitimacy signals. Writes a one-paragraph profile and a confidence score. Pre-fills a draft vendor record flagged for human review.
- **Tools:** web search, web fetch, vendors-table read access, "draft new vendor record" function.
- **Output:** draft vendor record + markdown profile, surfaced in the dashboard's "Pending Vendor Approvals" view.

### Agent 2: Discrepancy Investigator Agent

- **Triggered when:** a three-way match fails or a fraud flag fires.
- **Job:** pulls the invoice, the PO, the goods receipt, and the vendor's last 12 invoices. Analyzes what's off (price mismatch, quantity mismatch, missing GR, vendor pattern anomaly). Writes a short investigation summary in finance terms (example: "Invoice charges 50 SAR/unit but PO line specifies 45 SAR/unit; vendor has charged the higher rate on 3 of last 12 invoices, suggesting either price update not reflected in PO or systematic overbilling"). Drafts an email to the vendor requesting clarification, or an internal escalation note depending on severity. Drafts only — does not send.
- **Tools:** read access to invoices, POs, GRs, vendors. Web search for context. "Draft email" function that creates a draft.
- **Output:** investigation note + draft email/escalation, attached to the invoice's exception record.

### Agent 3: Cash Flow Chat Agent

- **Triggered when:** user invokes from the dashboard.
- **Job:** conversational agent users can ask questions like "What's my cash requirement next week?", "What happens if I delay the Aramex invoice by 7 days?", "Which discounts am I about to miss?", "Show me the riskiest unpaid invoices."
- **Tools:** read-only SQL execution against a finance view, the working-capital projector, a what-if simulator.
- **Output:** chat responses with citations to specific invoice rows and inline tables.

These three agents cover three distinct modes of useful AI work: research, investigation, conversation. Each agent's invocations are logged to the `agent_runs` table for observability.

---

## 4. Data model — nine core tables

```
vendors
  id, name, vat_number, cr_number, bank_account, address, email,
  status (active/blocked/pending_review), risk_score,
  agent_research_notes (text), created_at, updated_at

purchase_orders
  id, po_number, vendor_id, po_date, total_amount, currency,
  status (open/closed/cancelled), line_items (jsonb),
  requester, approver, created_at

goods_receipts
  id, gr_number, po_id, receipt_date, items_received (jsonb),
  receiver, condition_notes

invoices
  id, invoice_number, vendor_id, invoice_date, due_date,
  subtotal, vat_amount, total_amount, currency,
  source_format (pdf/image/xml), source_file_path,
  extraction_confidence,
  status (received/validated/matched/approved/rejected/paid/exception),
  zatca_qr (text), zatca_hash (text), risk_score,
  agent_investigation_notes (text), created_at

invoice_line_items
  id, invoice_id, description, quantity, unit_price,
  line_total, po_line_ref

approvals
  id, invoice_id, approver_user, decision (pending/approved/rejected),
  threshold_band, approval_time, comments

payments
  id, invoice_id, scheduled_payment_date, actual_payment_date,
  payment_method, discount_captured, amount_paid

audit_log
  id, entity_type, entity_id, action,
  actor (system/user_id/agent_name), timestamp,
  before_state (jsonb), after_state (jsonb)

agent_runs
  id, agent_name, trigger_event, input_context (jsonb),
  output (jsonb), tokens_used, cost, duration_ms, created_at
```

The `audit_log` table logs every change to every record — this is the controls foundation that makes the project credible as finance-grade work. The `agent_runs` table logs every agent invocation, providing analytics over the agents themselves (which gets used most, costs most, has highest correction rate).

---

## 5. Mock data and test inputs

**Database seed:** 20 mock vendors (mix of Saudi, GCC, international; mix of clean and slightly suspicious for fraud testing), 50 purchase orders, 30 goods receipts. Saudi VAT numbers follow the 15-digit format starting and ending with 3. CR numbers are 10 digits.

**Invoice inputs sourced from:**
- Public datasets: SROIE (Scanned Receipts OCR and Information Extraction), CORD (consolidated receipt/invoice dataset), RVL-CDIP.
- Synthetic PDFs generated with ReportLab or WeasyPrint from mock data, including deliberate edge cases (math errors, wrong VAT, duplicate numbers).
- ZATCA-compliant XML samples generated from ZATCA's published UBL 2.1 schema and example files.
- Edge-case images: lower-resolution screenshots of synthetic PDFs with rotation and noise to simulate phone-photo invoices.

**Email-based testing:** dedicated Gmail account with app-password access. A Python script reads invoices from a `test-invoices/` folder and emails them to the test inbox with realistic subject lines and bodies. The script doubles as a test harness and a demo-day driver.

---

## 6. Build phases — six phases plus a buffer

Estimated 6-7 weeks of focused work. Treat any week that takes 9-10 days as normal.

**Phase 1 — Foundations:** VPS provisioning and hardening, Docker stack via docker-compose, Postgres schema and migrations, mock data seeding (20 vendors, 50 POs, 30 GRs, Saudi-aware), FastAPI scaffold, `/extract` endpoint working on sample PDFs using Claude vision.
*Phase 1 deliverable:* curl a PDF, get clean structured JSON back.

**Phase 2 — Capture and validation:** n8n workflow watching the test Gmail inbox via IMAP, attachment handling, calls to FastAPI `/extract`, validation logic (math checks, VAT at 15%, duplicate detection, vendor lookup), Postgres writes, comprehensive audit logging.
*Phase 2 deliverable:* email a PDF to the test inbox, see it appear validated in the database within 30 seconds.

**Phase 3 — Matching, approvals, posting:** three-way match logic (PO → GR → invoice), Slack bot with interactive approve/reject messages, approval router based on amount thresholds, webhook handler for Slack responses, status updates, posting step, exception queue.
*Phase 3 deliverable:* full happy-path flow from email arrival to approved-and-posted invoice with Slack-based approval.

**Phase 4 — Agent layer + Saudi/finance modules:** all three agents implemented (Vendor Research, Discrepancy Investigator, Cash Flow Chat), ZATCA XML parser using lxml, QR code verifier using qrcode library, fraud detection rules layer, working-capital optimization nightly script, aged payables and cash forecast SQL views.
*Phase 4 deliverable:* feature-complete n8n build with agents running and finance reports queryable.

**Phase 5 — Streamlit dashboard:** invoice detail view where every action is one click from the invoice card (approve, reject, escalate, run discrepancy agent, draft vendor email, view audit trail, schedule payment, mark for audit). Agents page showing recent runs, success rate, average cost per run. Aged payables report. Cash requirements forecast (next 4 weeks). Working-capital projection chart. Fraud-flag log. Cash Flow Chat interface.
*Phase 5 deliverable:* a dashboard a CFO could click around in.

**Phase 6 — Polish and ship:** README following the structure in section 8, Excalidraw architecture diagram, Loom demo video (4-5 minutes, scripted, two takes max), production-gaps analysis, "if this were productized" section with three tiers (Basic/Pro/Enterprise), three persona sections (CFO / AP clerk / fintech PM).
*Phase 6 deliverable:* shippable portfolio piece ready for LinkedIn announcement.

**Buffer week:** almost certainly needed. Polish, bug-fix, video re-takes, push to LinkedIn with the framing post.

---

## 7. Testing approach

- **Unit tests** on the FastAPI service against 10 sample invoices with known correct extractions (pytest).
- **Integration tests** via Python script that drops emails through SMTP into the test inbox and asserts correct DB rows appear within 60 seconds.
- **Validation test cases** — deliberately broken invoices: math wrong, duplicate invoice number, VAT calculated at 14% instead of 15%, vendor not in master list. Assert each routes to the correct exception bucket. Document as "test scenarios" in the README.
- **Agent evaluation** — for each of the three agents, define 5 test cases with expected outputs/behaviors. Run them, log accuracy, document in the README. (Most portfolio projects skip this; including it is a senior signal.)
- **End-to-end manual test** — 15 mixed invoices (XMLs, PDFs, images, varying quality, some adversarial). Document extraction accuracy, processing time, exceptions raised, agent invocations, total cost. Becomes the "Performance" section of the README.

---

## 8. README structure for the public repo

1. Hero summary + animated demo GIF.
2. The business case — clerk-hours saved, error rate reduction, cycle-time reduction, with industry benchmarks cited (IOFM "cost per invoice").
3. Architecture — Excalidraw diagram + three-sentence flow explanation.
4. The agent layer — each agent profiled like a team member: name, job, tools available, example run.
5. Finance controls implemented — three-way match, segregation of duties, audit trail, ZATCA Phase 2, fraud detection. Explained in finance terms.
6. Tech stack and decisions — bulleted list with one-sentence justifications.
7. Test scenarios and results — table of test invoices + outcomes + agent eval results.
8. Production gaps — honest list of what would need to change to handle 5,000 invoices/day.
9. If this were productized — three tiers (Basic/Pro/Enterprise), GTM, integrations needed for v1, regulatory moat (ZATCA), competitor landscape (Wafeq, Qoyod, Pilot, Digits).
10. Persona sections — "If you're a CFO," "If you're an AP clerk," "If you're a fintech PM" — two paragraphs each.
11. What I learned / what I'd do differently — three to five honest bullets.

---

## 9. Demo video — 4-5 minutes, scripted

Recorded on Loom, two takes maximum, no over-editing.

1. *(20 sec)* Face on camera, hook: bridging finance and CS backgrounds, system processes invoices end-to-end with three named AI agents.
2. *(45 sec)* Architecture diagram, narrated.
3. *(60 sec)* Live demo — email an invoice, watch it flow, Slack approval, dashboard appearance.
4. *(45 sec)* Agent layer — Vendor Research on a new vendor, Discrepancy Investigator on a mismatch, Cash Flow Chat exchange.
5. *(30 sec)* ZATCA module — XML invoice with QR verification.
6. *(30 sec)* Finance controls — audit log, three-way match, fraud flag.
7. *(20 sec)* Close: looking for fintech and finance automation roles.

---

## 10. Pre-build setup checklist

- VPS provisioned (Contabo Cloud VPS 10, Frankfurt or Nuremberg, Ubuntu 24.04).
- SSH access verified from local Windows machine using OpenSSH.
- Public GitHub repo created.
- Anthropic API key generated.
- Dedicated Gmail account created with app password.
- Slack workspace (free tier) created with bot configured (Bot Token, App-Level Token, Signing Secret saved).
- Docker Desktop installed locally for testing.
- Python 3.12, Node.js LTS, Git for Windows installed.
- Antigravity IDE installed with Claude Code extension; VS Code settings imported; Remote-SSH configured to connect to VPS.

---

## 11. Working principles for the AI coding agent

- Always reference this document when making architectural or scope decisions. If a request conflicts with what's specified here, flag the conflict before proceeding rather than silently choosing.
- Do not introduce new dependencies, features, or tools without flagging them explicitly and explaining the trade-off.
- When in doubt about a design choice, ask before implementing rather than guessing.
- Show commands before executing them when working on infrastructure (SSH, Docker, server config, database migrations, anything touching production state).
- Never auto-approve destructive operations: `rm -rf`, `DROP TABLE`, `TRUNCATE`, force pushes, etc. Always require explicit human confirmation.
- Prefer small, reversible changes over large irreversible ones.
- If a task seems to require more than ~5 file changes, propose a plan first and wait for approval before executing.
- Treat this document as a living spec. Propose updates when meaningful decisions are made, but do not edit it without explicit approval.

## 12. Code conventions

- **Python:** type hints required on all function signatures. Use `ruff` for linting, `black` for formatting (line length 100).
- **SQL:** snake_case for tables and columns. Foreign keys named `<table>_id`. Every table has `created_at`; mutable tables also have `updated_at`.
- **Commits:** conventional commits format — `feat:`, `fix:`, `chore:`, `docs:`, `refactor:`, `test:`. Scope optional but encouraged: `feat(extraction): add ZATCA XML parser`.
- **Branches:** feature branches per module, named `feature/<short-description>`. Main branch is `main`. Avoid force-pushing.
- **Secrets:** never commit `.env`. Use `.env.example` as the template. Never hardcode API keys, passwords, or tokens.

## 13. File layout

```
ap-automation/
├── backend/          # FastAPI service
├── n8n/              # n8n workflow exports and config
├── database/         # Postgres init scripts and migrations
├── dashboard/        # Streamlit app
├── scripts/          # Helper scripts (mock data generation, email test harness)
├── test-invoices/    # Sample invoices for testing
├── docs/             # Architecture diagrams, extra documentation
├── docker-compose.yml
├── .env.example
├── .gitignore
├── README.md
├── LICENSE
├── CLAUDE.md         # Agent working instructions (to be generated from this doc)
└── SPEC.md           # This document
```

---

## 14. Current phase

**Phase 1: Foundations.** Setting up the Contabo VPS, Docker stack, Postgres schema, mock data seeding, and the FastAPI `/extract` endpoint. See section 6 for the full phase breakdown.

Update this section as phases progress so any new agent session sees the current state.

---

## How to use this document

This file is the single source of truth for the project's scope, architecture, conventions, and current state. Any AI coding agent working on this project should:

1. Read this document at the start of every session.
2. Reference it whenever scope, stack, or design decisions come up.
3. Propose updates here when meaningful decisions are made — but never edit it without the human's explicit approval.
4. Update the "Current phase" section as work progresses, with the human's approval.

A separate `CLAUDE.md` file should be generated from this document, focused on agent behavior and quick-reference working principles. CLAUDE.md is for the agent's session anchoring; this document is the full project ground truth.

---
