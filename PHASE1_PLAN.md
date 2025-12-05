# Phase 1 Foundation and MVP Core (Months 1-4)

This document outlines the implementation plan for the Phase 1 scope, covering multi-tenant infrastructure, the Tally integration layer, bank statement parsing, and the basic React dashboard.

## Milestones

1. **Month 1: Multi-tenant groundwork**
   - Bootstrap Django project with PostgreSQL schema-based tenancy (`django-tenants`).
   - Define shared vs. tenant schemas and migrations.
   - Establish JWT auth, invitation, and role model baseline.
   - Docker Compose environment (Django API, PostgreSQL, Redis, Celery worker/beat).

2. **Month 2: Background processing and Tally integration skeleton**
   - Celery task queue wired for async jobs (statement parsing, Tally sync).
   - REST endpoints and polling queue contracts for desktop connector.
   - XML request/response helpers and voucher templates scaffolded.

3. **Month 3: Bank parsing MVP**
   - PDF/Excel/CSV ingestion service with parser abstraction.
   - Transaction extraction and rule-based ledger suggestion engine (v1).
   - Bulk voucher generation pipeline producing Tally XML payloads.

4. **Month 4: Frontend and stabilization**
   - React 18 + TS + Vite dashboard with Ant Design.
   - Auth flow (login/refresh), file upload, transaction review UI.
   - Tally sync status views, connector health, retries.

## Backend Architecture

### Multi-tenancy
- Use `django-tenants` with PostgreSQL schemas: `public` (shared) and per-tenant schemas.
- Shared models: `Tenant`, `Domain`, `Plan`, `Subscription`.
- Tenant models: `User`, `Company`, `BankAccount`, `BankStatement`, `ParsedTransaction`, `LedgerMappingRule`.
- Subdomain routing via `Domain.is_primary`; middleware selects schema per request.

### Authentication & Authorization
- JWT via `djangorestframework-simplejwt` (access + refresh tokens).
- Invitation workflow: tenant admin invites by email → tokenized accept endpoint → sets password and role.
- Roles stored per-tenant user (e.g., Admin, Accountant, Viewer) with DRF permission classes.

### Background Processing
- Celery + Redis for async jobs.
- Queues: `ingestion` (file parsing), `tally_sync` (voucher pushes, ledger sync), `maintenance` (schema migrations per tenant, subscription checks).
- Retry policy with exponential backoff; LINEERROR parsing feeds retryable failures.

### Local Development Environment
- Docker Compose services: `api` (Django + Gunicorn), `db` (PostgreSQL with multiple schemas), `redis`, `worker` (Celery), `beat` (Celery beat), `frontend` (Vite dev server placeholder), `connector-mock` (HTTP server to simulate desktop connector polling cycle during development).
- `.env` for database, Redis, JWT secret, and Tally connector endpoints.

## Tally Integration Layer

### Desktop Connector Strategy
- Desktop connector (Python/Electron) runs on Tally PC and polls SaaS queue endpoints.
- Polling contract:
  - `GET /connector/tasks?tenant=<id>&limit=...` returns pending operations.
  - `POST /connector/tasks/{id}/result` uploads Tally XML response + status.
- Connector invokes TallyPrime at `localhost:9000` with generated XML payloads and returns raw XML (including `LINEERROR`).

### XML Handling
- Library of XML templates (Payment, Receipt, Journal, Sales, Purchase) with mandatory fields (`PERSISTEDVIEW` etc.).
- Parser utilities to extract errors and normalize ledger/item references.
- Voucher creation envelope example retained from requirements for validation tests.

### Ledger/Master Sync
- Scheduled Celery job pulls ledgers/masters from Tally via connector and caches mapping per tenant/company.
- Changes feed rule engine for ledger suggestions.

## Bank Statement Parsing (MVP)

### Ingestion Pipeline
1. Upload endpoint stores file metadata (`BankStatement`) and schedules parsing task.
2. Parser registry selects handler (PDF via `PyPDF2` + `tabula-py`; Excel/CSV via `pandas`).
3. Normalization yields `ParsedTransaction` records with date, amount, description, balance.
4. Rule-based ledger suggestion engine:
   - Pattern matching on description (`LedgerMappingRule` with priorities).
   - Heuristics (sign-based ledger classes, recurring payees, GST pattern detection).
   - Confidence scoring; flagged for review if below threshold.
5. Bulk voucher generation batches transactions into Tally XML (Payment/Receipt/Sales/Purchase determined by sign and mappings).

### Format Coverage
- Start with top 10 banks (SBI, HDFC, ICICI, Axis, Kotak, Yes Bank, IDFC First, IndusInd, HSBC, Citi) and iterate toward 50+ via configurable column maps and regex-based normalizers.
- Maintain fixtures and sample statements for regression tests.

## Frontend (React 18 + TypeScript + Vite)

- Ant Design setup with theme tokens for brand colors.
- Auth flow: login form → store access/refresh tokens → auto-refresh interceptor.
- Key screens:
  - **File Upload**: bank statement upload with drag/drop, shows parsing status.
  - **Transaction Review**: table with suggested ledger, confidence indicator, inline edits, bulk approve.
  - **Ledger Mapping**: drag-and-drop or selectable ledger linking, shows rule preview and "Speedy Recommendations".
  - **Tally Sync Dashboard**: connector status, pending tasks, retry counts, last sync time.

## Acceptance Criteria (Phase 1)
- Multi-tenant API can create tenants/domains and route per-subdomain requests.
- JWT auth with invitation + role assignment works end-to-end.
- Celery tasks operate for parsing and Tally sync with Redis backend.
- Connector polling endpoints deliver/accept tasks; voucher XML is generated and accepted by Tally (via mock/sandbox).
- Bank statement parsing supports PDF/Excel/CSV for initial bank set and produces vouchers with ledger suggestions.
- React dashboard allows login, file upload, transaction review, and displays Tally sync status.
