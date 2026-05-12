# AP Automation System with AI Agents

An end-to-end Accounts Payable automation system that takes vendor invoices from email arrival to approved-and-posted in a mocked ERP. Built on n8n (self-hosted on a Contabo VPS) with a FastAPI backend handling LLM extraction and agent logic, it pairs three named AI agents — a Vendor Research Agent, a Discrepancy Investigator Agent, and a Cash Flow Chat Agent — with a finance-controls layer (three-way matching, segregation of duties, audit trail, working-capital optimization, fraud detection) and ZATCA Phase 2 awareness for Saudi e-invoicing. Designed and presented as a product, not a tech demo, with tier-based product thinking, finance reports a CFO would actually read, and an honest production-gap analysis.

See [SPEC.md](SPEC.md) for the full project context and [CLAUDE.md](CLAUDE.md) for agent working principles.
