# Architecture Firm Management — Phased Development Plan

> Project: 325-architecture-firm-management · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three `data-model-suggestion-*.md` documents into an implementation specification for an AI-native, open-source practice management platform for architecture and engineering (A/E) firms. The canonical database schema adopts **data-model-suggestion-1 (Entity-Centric Normalized Relational)** as its backbone, because the project → phase → time → invoice → payment pipeline benefits from database-enforced referential integrity, AIA phases as first-class entities, and auditable DCAA/EVM columns. JSONB is used selectively (firm settings, invoice line items, AI analysis payloads) where the variability argument from suggestion-2 applies. The event/audit philosophy of suggestion-3 is incorporated as an append-only `audit_log` plus typed domain events for invoice and timesheet lifecycles, rather than full CQRS, to keep the MVP tractable.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language (backend) | Python 3.12 | The product's differentiators (AI timesheet drafting, fee estimation, predictive health, report generation, zoning intelligence) are LLM- and data-heavy. Python has the strongest LLM/ML ecosystem and first-class SDKs for OpenAI/Anthropic, plus mature accounting-integration clients (intuit-oauth, xero-python). |
| API framework | FastAPI | Async-native (needed for long-running LLM calls and webhook fan-out), auto-generates an OpenAPI 3.1 spec (standards.md requires OAS 3.2-class output; FastAPI emits 3.1 which is a strict subset upgrade target), and integrates Pydantic v2 for request/response validation. |
| Data validation / models | Pydantic v2 | Single source of truth for request/response schemas, config, and LLM structured-output parsing. |
| ORM / DB toolkit | SQLAlchemy 2.0 (async) + Alembic | Mature async ORM for the 18-table normalized model; Alembic gives versioned migrations required by the per-phase Definition of Done. |
| Database | PostgreSQL 16 | The normalized model uses CHECK constraints, partial indexes, partitioned `audit_log`, JSONB with GIN indexes, and TIMESTAMPTZ — all PostgreSQL features. Multi-tenant row isolation (firm_id) maps to PostgreSQL RLS in a later phase. |
| Task queue / async jobs | Celery + Redis | Webhook delivery, accounting sync, LLM batch jobs (timesheet drafting from calendar, nightly project-health recompute) are async and retryable. Redis doubles as cache and Celery broker/result backend. |
| LLM provider abstraction | `litellm` (or thin internal `LLMClient`) | Lets the firm self-host with a choice of provider (Anthropic, OpenAI, Azure, local Ollama). Self-hosting is a first-class deployment mode per README. |
| Frontend | Next.js 15 (App Router) + React + TypeScript | Two surfaces — the internal firm dashboard and the external client portal — share a component library. App Router supports server components for fast financial dashboards. Deployable to Vercel or self-hosted Node. |
| UI component library | shadcn/ui + Tailwind CSS | Fast to build accessible, modern dashboards; addresses the "weak UX / clunky" gap cited for incumbents. |
| Charts | Recharts | Budget burn, EVM (SPI/CPI), utilisation, and AR-aging visualisations. |
| Auth (internal) | OAuth 2.0 + JWT (RFC 6749/6750/7519); OIDC SSO later | standards.md mandates OAuth 2.0; enterprise firms expect OIDC SSO (Entra ID, Google, Okta). Implemented with `authlib`. |
| Accounting integrations | `intuit-oauth` + QuickBooks Online API; `xero-python` | Table-stakes per features.md; both use OAuth 2.0 with PKCE. |
| PDF generation | WeasyPrint (HTML/CSS → PDF) | Invoices, AIA-style G702/G703-equivalent output, and client reports. Avoids reproducing copyrighted AIA templates — generates an equivalent layout. |
| Containerisation | Docker + docker-compose | Self-hosting is first-class; compose bundles API, worker, Postgres, Redis, and frontend. |
| Testing | pytest + pytest-asyncio + httpx + testcontainers | Unit, mocked-integration, and real-DB integration tests. testcontainers spins up real Postgres for migration/integration tests. |
| Frontend testing | Vitest + Playwright | Component unit tests and E2E flows (login → create project → log time → generate invoice). |
| Code quality | ruff (lint+format), mypy (types), pre-commit; eslint + prettier (frontend) | Enforced in the per-phase Definition of Done. |
| Package management | uv (Python), pnpm (frontend) | Fast, reproducible installs. |
| API style | REST + JSON, OpenAPI 3.1, cursor pagination, RFC 7807 problem+json errors | Mirrors incumbent API patterns (BQE, Deltek, Productive's JSON:API); RFC 7807 standardises error bodies. |
| MCP server | `mcp` Python SDK (later phase) | Scoro shipped an MCP server in 2026; exposing project/resource data over MCP positions the product for AI-agent workflows. |

### Project Structure

```
architecture-firm-management/
├── README.md
├── LICENSE
├── docker-compose.yml
├── Dockerfile                      # API + worker image
├── pyproject.toml                  # uv-managed; ruff, mypy, pytest config
├── alembic.ini
├── .env.example
├── backend/
│   ├── alembic/
│   │   ├── env.py
│   │   └── versions/               # one migration per schema-changing phase
│   ├── afm/                        # application package ("architecture firm mgmt")
│   │   ├── __init__.py
│   │   ├── main.py                 # FastAPI app factory, router registration
│   │   ├── config.py               # Pydantic Settings (env-driven)
│   │   ├── db.py                   # async engine, session, Base
│   │   ├── deps.py                 # FastAPI dependencies (current_firm, current_user, db session)
│   │   ├── security/
│   │   │   ├── auth.py             # OAuth2 password+refresh, JWT issue/verify
│   │   │   ├── rbac.py             # role checks, object-level authorisation
│   │   │   └── oidc.py             # SSO (later)
│   │   ├── models/                 # SQLAlchemy ORM models (one module per domain)
│   │   │   ├── firm.py             # firms, offices, staff
│   │   │   ├── client.py           # clients, opportunities
│   │   │   ├── project.py          # projects, project_phases
│   │   │   ├── subconsultant.py
│   │   │   ├── time_expense.py     # time_entries, expenses
│   │   │   ├── billing.py          # invoices, invoice_line_items, payments
│   │   │   ├── indirect.py         # indirect_cost_pools
│   │   │   ├── deliverable.py
│   │   │   └── audit_ai.py         # audit_log, ai_analyses
│   │   ├── schemas/                # Pydantic request/response DTOs (mirrors models/)
│   │   ├── services/               # business logic (no FastAPI imports)
│   │   │   ├── projects.py
│   │   │   ├── time_tracking.py
│   │   │   ├── budgets.py          # burn, EVM (BCWS/BCWP/ACWP, SPI/CPI)
│   │   │   ├── billing.py          # invoice generation per fee_type
│   │   │   ├── aia.py              # G702/G703-equivalent Schedule of Values
│   │   │   ├── revenue.py          # ASC 606 / IFRS 15 recognition
│   │   │   ├── indirect_cost.py    # DCAA/FAR indirect rate calc
│   │   │   ├── reporting.py
│   │   │   └── dashboards.py
│   │   ├── api/                    # FastAPI routers (thin; call services)
│   │   │   └── v1/
│   │   │       ├── auth.py
│   │   │       ├── firms.py
│   │   │       ├── staff.py
│   │   │       ├── clients.py
│   │   │       ├── opportunities.py
│   │   │       ├── projects.py
│   │   │       ├── phases.py
│   │   │       ├── time_entries.py
│   │   │       ├── expenses.py
│   │   │       ├── invoices.py
│   │   │       ├── payments.py
│   │   │       ├── deliverables.py
│   │   │       ├── reports.py
│   │   │       ├── dashboards.py
│   │   │       ├── ai.py
│   │   │       ├── integrations.py # QBO/Xero connect, webhooks out
│   │   │       └── portal.py       # client portal endpoints
│   │   ├── integrations/
│   │   │   ├── accounting/
│   │   │   │   ├── base.py         # AccountingProvider protocol
│   │   │   │   ├── quickbooks.py
│   │   │   │   └── xero.py
│   │   │   ├── calendar/           # calendar/email pull for AI timesheets
│   │   │   └── webhooks.py         # outbound webhook dispatch
│   │   ├── ai/
│   │   │   ├── client.py           # LLMClient abstraction
│   │   │   ├── prompts.py          # versioned prompt templates
│   │   │   ├── timesheet_draft.py
│   │   │   ├── fee_estimate.py
│   │   │   ├── health_alerts.py
│   │   │   ├── client_report.py
│   │   │   └── zoning.py
│   │   ├── pdf/
│   │   │   ├── templates/          # Jinja2 HTML for invoices, reports, AIA SoV
│   │   │   └── render.py           # WeasyPrint wrapper
│   │   ├── tasks/                  # Celery tasks
│   │   │   ├── celery_app.py
│   │   │   ├── accounting_sync.py
│   │   │   ├── ai_jobs.py
│   │   │   ├── health_recompute.py
│   │   │   └── webhook_delivery.py
│   │   └── events.py               # typed domain events + audit_log writer
│   └── tests/
│       ├── conftest.py             # fixtures: db, client, firm/staff factories
│       ├── unit/
│       ├── integration/
│       ├── e2e/
│       └── fixtures/               # sample diffs, sample timesheets, golden PDFs
├── frontend/
│   ├── package.json
│   ├── app/
│   │   ├── (dashboard)/            # internal app: projects, time, billing, reports
│   │   ├── (portal)/              # client portal: invoices, payments, status
│   │   └── api/                    # BFF route handlers (proxy to backend)
│   ├── components/                 # shadcn/ui-based shared components
│   ├── lib/
│   │   └── api-client.ts           # typed client generated from OpenAPI spec
│   └── tests/
└── docs/
    └── openapi.json                # exported spec (CI artifact)
```

The structure is grouped by concern (models, services, api, integrations, ai) so each phase adds files without restructuring.

---

## Phase 1: Foundation & Core Domain Model

### Purpose
Establish the project skeleton, configuration, database connectivity, and the firm/staff/auth core that every other feature depends on. After this phase, a multi-tenant application boots, runs migrations, authenticates a staff user, and enforces firm-scoped object-level authorisation (OWASP API1:2023). This phase is small but load-bearing.

### Tasks

#### 1.1 — Project scaffold, config, and tooling

**What**: Create the repository skeleton, `pyproject.toml`, Docker/compose, and a bootable FastAPI app with health check.

**Design**:
- `afm/config.py` — Pydantic `Settings` loaded from environment:
```python
class Settings(BaseSettings):
    database_url: str            # postgresql+asyncpg://...
    redis_url: str = "redis://localhost:6379/0"
    jwt_secret: str
    jwt_alg: str = "HS256"
    access_token_ttl_min: int = 30
    refresh_token_ttl_days: int = 30
    llm_provider: str = "anthropic"      # anthropic|openai|azure|ollama
    llm_model: str = "claude-3-7-sonnet"
    environment: str = "development"
    model_config = SettingsConfigDict(env_file=".env")
```
- `afm/main.py` — `create_app()` factory: registers v1 routers, RFC 7807 exception handlers, CORS, request-ID middleware.
- `GET /healthz` → `{"status": "ok", "version": <git_sha>}`.
- `docker-compose.yml` services: `api`, `worker`, `db` (postgres:16), `redis`, `frontend`.
- ruff + mypy + pre-commit configured.

**Testing**:
- `Unit: Settings loads from env → fields populated with correct types/defaults`
- `Unit: missing required env (jwt_secret) → ValidationError naming the field`
- `Integration: GET /healthz → 200, body {"status":"ok"}`
- `Integration: docker compose build → image builds, api container passes healthcheck`

#### 1.2 — Database layer, Base, and firm/office/staff models

**What**: Async SQLAlchemy engine/session, declarative `Base`, and the `firms`, `offices`, `staff` tables with the first Alembic migration.

**Design**:
- Adopt DDL from data-model-suggestion-1 "Firm & Staff" verbatim (UUID PKs, `firms.fiscal_year_start_month`, `staff.role` CHECK constraint with the nine roles, `hourly_cost_cents`/`hourly_bill_rate_cents` as BIGINT, `utilisation_target_pct`).
- `afm/db.py`: `async_engine`, `async_session_maker`, `get_session()` dependency.
- ORM models map columns 1:1; relationships `firm.offices`, `firm.staff`, `staff.office`.
- Alembic migration `0001_firms_offices_staff`.

**Testing**:
- `Integration (real Postgres via testcontainers): run migration 0001 → firms/offices/staff tables exist with expected columns and constraints`
- `Unit: staff.role = 'wizard' → IntegrityError (CHECK violation)` (real DB)
- `Unit: insert firm with duplicate slug → UniqueViolation`
- `Unit: ORM round-trip — create firm + office + staff, reload, relationships resolve`

#### 1.3 — Authentication and object-level authorisation

**What**: OAuth 2.0 password+refresh-token flow issuing JWTs, plus dependencies that resolve the current staff user and their firm, and enforce firm-scoped access.

**Design**:
- `POST /v1/auth/token` (OAuth2 password grant): body `{username, password}` → `{access_token, refresh_token, token_type:"bearer", expires_in}`. JWT claims: `sub` (staff_id), `firm` (firm_id), `role`, `exp`, `iat` (RFC 7519).
- `POST /v1/auth/refresh`: `{refresh_token}` → new access token.
- Passwords hashed with argon2 (`argon2-cffi`). Add `staff.password_hash TEXT` (migration `0002`).
- `deps.py`:
  - `current_user(token) -> Staff`
  - `current_firm() -> UUID` (from JWT `firm` claim)
  - `require_role(*roles)` dependency factory.
- **Object-level authorisation helper** `authorize_firm(resource_firm_id, current_firm_id)` raises 403 if mismatched — mitigates OWASP API1:2023 (BOLA). Every resource query filters by `firm_id`.

**Testing**:
- `Integration: valid credentials → 200 with access+refresh tokens; JWT decodes with correct sub/firm/role`
- `Integration: wrong password → 401, no token`
- `Integration: expired access token → 401`
- `Integration: refresh with valid refresh token → new access token`
- `Unit: authorize_firm(firmA, firmB) → 403 raised`
- `Integration (BOLA): staff of firm A requests GET /v1/projects/{id} for firm B's project → 404/403, no data leak`

---

## Phase 2: Clients, Projects & AIA Phase Structure

### Purpose
Build the heart of the domain: clients, the CRM opportunity pipeline, projects with AIA-aligned fee structures, and phase breakdowns. After this phase, a firm can model real engagements — win an opportunity, create a project with a fee type, and lay out standard AIA design phases with budgets. This is the structure all time, billing, and reporting hang off.

### Tasks

#### 2.1 — Clients and opportunities (CRM)

**What**: CRUD for `clients` and `opportunities` with pipeline-stage transitions.

**Design**:
- DDL from suggestion-1 "Clients & CRM" (`clients.type` CHECK; `opportunities.stage` CHECK with lead→won/lost; partial index on open stages).
- Endpoints (all firm-scoped):
  - `POST/GET/PATCH/DELETE /v1/clients[/{id}]`
  - `POST/GET/PATCH /v1/opportunities[/{id}]`
  - `POST /v1/opportunities/{id}/transition` body `{stage, reason?}` — validates allowed transitions; emits `opportunity.stage_changed` audit event; `won` is the only stage that may spawn a project (see 2.3).
- Pydantic schemas: `ClientCreate/Read`, `OpportunityCreate/Read`, `OpportunityTransition`.
- List endpoints: cursor pagination, `?stage=`, `?client_id=` filters.

**Testing**:
- `Unit: OpportunityCreate with probability_pct=120 → ValidationError (0–100 bound)`
- `Integration: create client → 201; GET lists it scoped to firm only`
- `Integration: transition lead→won → 200, stage persisted, audit_log row written`
- `Integration: transition won→lead → 422 (illegal transition)`

#### 2.2 — Projects with fee structures

**What**: Project CRUD supporting all five fee types from the AIA B-series.

**Design**:
- DDL from suggestion-1 "Projects" (`fee_type` CHECK: fixed/hourly/percentage/cost_plus/not_to_exceed; `revenue_recognition` CHECK; `is_government_contract`; `UNIQUE(firm_id, number)`; partial index on active/on_hold).
- `ProjectCreate` schema with conditional validation:
  - `fixed`/`not_to_exceed` → `contract_amount_cents` required
  - `percentage` → `construction_cost_cents` + `fee_percentage` required; derived contract = cost × pct
  - `hourly`/`cost_plus` → rates resolved from staff at billing time
- `services/projects.py`: `create_project`, `compute_contract_value(project)`.
- Endpoints `POST/GET/PATCH /v1/projects[/{id}]`; `GET /v1/projects?status=&client_id=`.

**Testing**:
- `Unit: percentage fee, cost=$5,000,000, pct=8.5 → contract_value=$425,000`
- `Unit: fixed fee with no contract_amount → ValidationError`
- `Integration: duplicate project number within firm → 409`
- `Integration: same project number across two firms → both succeed (firm-scoped uniqueness)`

#### 2.3 — Phases and phase templates

**What**: `project_phases` with AIA phase codes, phase budgets, and reusable templates; auto-generate phases when an opportunity is won.

**Design**:
- DDL from suggestion-1 "Projects & Phases" `project_phases` (phase_code CHECK with the 8 AIA codes incl. `custom`; `fee_cents`, `budget_hours`, `percent_complete`, `sort_order`; EVM columns `bcws_cents`/`bcwp_cents`/`acwp_cents`).
- Default AIA template (matches suggestion-2 firm settings example): SD 15%, DD 20%, CD 40%, B&N 5%, CA 20% of contract fee. Stored in `firms.settings->default_phase_template` (JSONB).
- `services/projects.py`:
  - `apply_phase_template(project, template) -> list[Phase]` distributes `fee_cents` by percentage.
  - `create_project_from_opportunity(opp)` — copies estimated fee and applies default template.
- Endpoints: `POST /v1/projects/{id}/phases`, `PATCH /v1/phases/{id}` (status, percent_complete, budget), `POST /v1/projects/{id}/phases:apply-template`.
- Phase status transitions emit `project.phase_started` / `project.phase_completed` audit events.

**Testing**:
- `Unit: apply default template to $500,000 fee → CD phase fee = $200,000 (40%), sum of phase fees = contract`
- `Unit: percent_complete=150 → ValidationError`
- `Integration: win opportunity → project created with 5 phases from template`
- `Integration: PATCH phase status not_started→completed → audit event written`

---

## Phase 3: Time Tracking & Budget Burn

### Purpose
Deliver the most-used daily workflow and the data foundation for billing and project health. After this phase, staff log time against project phases (manual or timer), the system computes billable/non-billable hours, budget burn per phase, and exposes a project health dashboard with budget-consumed metrics. This is where the product starts delivering value.

### Tasks

#### 3.1 — Time entries

**What**: Create, edit, list, and approve time entries with project/phase/role attribution and a source tag for later AI attribution.

**Design**:
- DDL from suggestion-1 "Time & Expenses" `time_entries` (`hours > 0` CHECK; `is_billable`; `source` CHECK manual/ai_draft/calendar_import/timer; `approved_by`/`approved_at`; indexes on staff, project, phase by date).
- Endpoints:
  - `POST /v1/time-entries` `{project_id, phase_id, entry_date, hours, is_billable, description}`
  - `GET /v1/time-entries?staff_id=&project_id=&from=&to=` (cursor paginated)
  - `PATCH /v1/time-entries/{id}` (retroactive edit — addresses Monograph's "hard to edit" gap)
  - `POST /v1/time-entries/{id}/approve` (role: project_manager+)
  - `GET /v1/timesheets/week?staff_id=&week_start=` → weekly grid (mirrors suggestion-3 `rm_timesheets` shape, computed on read)
- Validation: `phase_id` must belong to `project_id` and same firm.

**Testing**:
- `Unit: hours=0 → ValidationError`
- `Integration: phase_id from a different project → 422`
- `Integration: create entry then PATCH hours → updated, updated_at advances`
- `Integration: approve sets approved_by/approved_at; non-manager approve → 403`
- `Integration: weekly grid aggregates entries by day with billable totals`

#### 3.2 — Budget burn service

**What**: Compute phase- and project-level budget burn from approved time and staff cost rates.

**Design**:
- `services/budgets.py`:
```python
@dataclass
class PhaseBurn:
    phase_id: UUID
    budget_cents: int          # phase.fee_cents
    budget_hours: float
    actual_hours: float
    labor_cost_cents: int      # sum(hours * staff.hourly_cost_cents)
    expense_cents: int
    percent_complete: float
    budget_consumed_pct: float # (labor+expense) / budget * 100
```
  - `compute_phase_burn(phase_id) -> PhaseBurn`
  - `compute_project_burn(project_id) -> ProjectBurn` (aggregates phases)
- Uses the "Project financial summary" query from suggestion-1 as the basis.
- Endpoint `GET /v1/projects/{id}/burn`.

**Testing**:
- `Unit: phase budget $50,000, 145h logged at $80/h cost + $2,400 expenses → labor=$11,600, consumed=28%`
- `Unit: zero-budget phase → budget_consumed_pct = None (avoid div-by-zero)`
- `Integration: only approved entries count toward burn (draft entries excluded if approval required)`

#### 3.3 — Project health dashboard (MVP)

**What**: A dashboard endpoint returning budget consumed, time remaining, and phase status per project — the MVP "basic project health dashboard."

**Design**:
- `services/dashboards.py: project_health(project_id) -> ProjectHealth` returning fields modelled on suggestion-3 `rm_project_health` (status, fee_type, contract_cents, budget_consumed_pct, current_phase, per-phase metrics, simple `budget_health` classification: <80% on_track, 80–100% at_risk, >100% over_budget).
- `GET /v1/dashboards/project/{id}` and `GET /v1/dashboards/firm` (portfolio list with health flags).
- Computed live in MVP (no materialised read model yet; that arrives in Phase 8 if needed).

**Testing**:
- `Unit: consumed 85% → budget_health = 'at_risk'`
- `Unit: consumed 110% → budget_health = 'over_budget'`
- `Integration: firm dashboard lists only active/on_hold projects, sorted with at-risk first`

---

## Phase 4: Expenses, Sub-Consultants & Deliverables

### Purpose
Round out cost tracking with reimbursable/billable expenses, sub-consultant pass-through (a Factor A/E / BQE differentiator), and deliverable milestone tracking aligned to phases. After this phase the project record captures the full cost picture needed for accurate invoicing and ASC 606 revenue.

### Tasks

#### 4.1 — Expenses

**What**: Expense entry with category, billable/reimbursable/allowable flags (FAR Part 31), and receipt URL.

**Design**:
- DDL from suggestion-1 `expenses` (`category` CHECK; `is_reimbursable`, `is_billable`, `is_allowable`; `sub_consultant_id` FK; `receipt_url`).
- `is_allowable` defaults TRUE; meals/entertainment categories default FALSE per FAR Part 31 guidance.
- Endpoints `POST/GET/PATCH /v1/expenses`, `POST /v1/expenses/{id}/approve`.

**Testing**:
- `Unit: category='meals' → is_allowable defaults False`
- `Integration: expense linked to phase appears in that phase's burn (Phase 3 service)`
- `Integration: invalid category → 422`

#### 4.2 — Sub-consultants and pass-through

**What**: Firm-level sub-consultant directory and project/phase-level sub-consultant contracts with markup.

**Design**:
- DDL from suggestion-1 "Sub-Consultants" (`sub_consultants`, `project_sub_consultants` with `contract_amount_cents`, `markup_pct`).
- `services/projects.py: subconsultant_passthrough(project_id) -> int` = Σ(contract × (1+markup)).
- Endpoints `POST/GET /v1/sub-consultants`, `POST/GET /v1/projects/{id}/sub-consultants`.
- Sub-consultant invoices recorded as expenses with `category='sub_consultant'` and `sub_consultant_id` set.

**Testing**:
- `Unit: contract $50,000 markup 10% → billable pass-through $55,000`
- `Integration: add sub-consultant to project → appears in project detail; pass-through total correct`

#### 4.3 — Deliverables

**What**: Phase-scoped deliverable milestones (drawing sets, specs, submittals) with status and due dates; ISO 4157 / ISO 19650 naming notes.

**Design**:
- DDL from suggestion-1 "Deliverables & Milestones" (`deliverable_type` CHECK; `status` CHECK not_started→submitted).
- Optional `info_container_code TEXT` column (migration) for ISO 19650 CDE naming, nullable.
- Endpoints `POST/GET/PATCH /v1/projects/{id}/deliverables`.
- Deliverable completion can advance phase `percent_complete` (configurable, default off).

**Testing**:
- `Unit: invalid deliverable_type → 422`
- `Integration: list deliverables by phase; status transition not_started→review persisted`

---

## Phase 5: Billing & Invoicing

### Purpose
Turn tracked time, expenses, and sub-consultant costs into invoices across all three MVP billing modes, plus the AIA G702/G703-equivalent Schedule of Values that is a key differentiator. After this phase a firm can generate, approve, and export invoices as PDFs. This completes the MVP value loop (project → time → bill).

### Tasks

#### 5.1 — Invoice domain and generation engine

**What**: `invoices`, `invoice_line_items`, `payments` tables and a generation service that branches on billing method.

**Design**:
- DDL from suggestion-1 "Invoices & Payments" (`billing_method` CHECK incl. `aia_g702`; `status` CHECK full lifecycle; `subtotal/tax/total/amount_paid/retainer_applied`; `invoice_line_items.line_type` CHECK; `payments`).
- `services/billing.py`:
```python
def generate_invoice(project_id: UUID, method: str, period: DateRange) -> Invoice: ...
# branches:
#  time_and_materials -> lines from approved billable time (hours*bill_rate) + billable expenses + sub markup
#  fixed_fee          -> single line for agreed amount (or per-phase fixed)
#  percent_complete   -> per-phase (this_period_pct - previous_pct) * phase.fee_cents
#  aia_g702           -> Schedule of Values (see 5.2)
```
- Endpoints `POST /v1/invoices:generate`, `GET/PATCH /v1/invoices/{id}`, `POST /v1/invoices/{id}/approve|send`, `POST /v1/invoices/{id}/payments`.
- Status transitions emit invoice lifecycle audit events (suggestion-3 invoice timeline).

**Testing**:
- `Unit: T&M — 42h @ $150 bill + $450 expense → subtotal $6,750`
- `Unit: percent_complete — phase fee $50,000, prev 30%, now 45% → line $7,500`
- `Integration: payment applied → status partially_paid→paid; amount_paid updated`
- `Integration: send draft invoice → 422 unless status=approved`

#### 5.2 — AIA G702/G703-equivalent Schedule of Values

**What**: Generate an AIA-style application-for-payment with a continuation-sheet Schedule of Values — using an equivalent, non-copyrighted layout (per README legal note).

**Design**:
- `services/aia.py: build_schedule_of_values(project_id, period) -> list[SoVLine]` where each line carries `description, scheduled_value_cents, previous_pct/cents, this_period_pct/cents, total_completed_cents, balance_cents, retainage_cents` (structure mirrors suggestion-2 `aia_g702` line_items example).
- Summary (G702-equivalent): original contract sum, change orders, completed-to-date, retainage, current payment due.
- Stored as `invoice_line_items` with `line_type` mapping + a JSONB `aia_summary` on the invoice.
- **Does not** reproduce AIA-branded templates; PDF uses an original layout (Phase 6).

**Testing**:
- `Unit: SoV totals — Σ(this_period_cents) = invoice subtotal; balance = scheduled - total_completed`
- `Unit: retainage 5% applied → current payment due reduced accordingly`
- `Fixture: golden SoV for a 3-phase project matches expected line values`

#### 5.3 — Revenue recognition (ASC 606 / IFRS 15)

**What**: Compute recognised revenue per project using the project's `revenue_recognition` method.

**Design**:
- `services/revenue.py: recognise(project_id, as_of) -> RevenueSnapshot`:
  - `percent_complete` / `cost_to_cost` → contract × completion ratio − previously recognised
  - `milestones` → sum of completed milestone values
  - `time_and_materials` → billed-to-date
- Endpoint `GET /v1/projects/{id}/revenue?as_of=`.

**Testing**:
- `Unit: cost_to_cost — costs $200k of $500k budget, $1M contract → recognised $400k`
- `Unit: idempotent — recognising twice at same date yields same cumulative figure`

---

## Phase 6: PDF Output, Reporting & Client Portal

### Purpose
Make the firm's output client-ready and give clients self-service access — closing two incumbent gaps (weak PDF reporting; only Accelo has a real portal). After this phase, invoices and progress reports export as polished PDFs, and clients log into a portal to view status and pay invoices.

### Tasks

#### 6.1 — PDF rendering (invoices, AIA SoV, reports)

**What**: HTML/CSS → PDF rendering pipeline with Jinja2 templates and WeasyPrint.

**Design**:
- `pdf/render.py: render_pdf(template_name, context) -> bytes`.
- Templates: `invoice.html`, `aia_application.html` (original layout), `progress_report.html`.
- Endpoint `GET /v1/invoices/{id}/pdf` → `application/pdf`.
- Firm branding (logo, colours) pulled from `firms.settings`.

**Testing**:
- `Integration: render invoice → valid PDF bytes, non-empty, %PDF header`
- `Fixture: golden-PDF text extraction contains invoice number, total, line descriptions`

#### 6.2 — Customisable reporting

**What**: Parameterised reports (project financial summary, staff utilisation, AR aging) with PDF/CSV export.

**Design**:
- `services/reporting.py` backed by the three example queries in suggestion-1 (financial summary, utilisation, AR aging).
- `POST /v1/reports/{report_type}` `{filters, format: pdf|csv|json}`.
- Report types: `project_financials`, `staff_utilisation`, `ar_aging`, `phase_burn`.

**Testing**:
- `Integration: ar_aging report → rows with days_overdue computed; CSV export parseable`
- `Unit: utilisation — 769 billable of 1240 total → 62%`

#### 6.3 — Client portal

**What**: External, separately-authenticated portal where a client views their projects' status, invoices, and pays.

**Design**:
- Portal auth: magic-link or scoped portal JWT (`aud: "portal"`, `client_id` claim) — strictly read-scoped to that client's projects (OWASP API1 BOLA enforcement).
- Endpoints under `/v1/portal`: `GET /portal/projects`, `GET /portal/invoices`, `GET /portal/invoices/{id}/pdf`, `POST /portal/invoices/{id}/pay` (Phase 7 payment).
- Frontend `(portal)` route group: minimal, branded, mobile-first.

**Testing**:
- `Integration: portal token for client A → cannot read client B's invoices (403/empty)`
- `Integration: portal user sees only sent/paid invoices, never drafts`
- `E2E: client logs in, views project status, opens invoice PDF`

---

## Phase 7: Accounting Integrations & Async Infrastructure

### Purpose
Connect to QuickBooks Online and Xero for accounting handoff (table-stakes), and stand up the Celery/Redis async layer that integrations, webhooks, and later AI jobs depend on. After this phase, invoices and payments sync to the firm's accounting system and external systems receive webhooks.

### Tasks

#### 7.1 — Async task infrastructure

**What**: Celery app, Redis broker, retry policy, and a dead-letter pattern.

**Design**:
- `tasks/celery_app.py` with Redis broker/result backend; default retry `max_retries=5`, exponential backoff.
- Task base class logs to `audit_log` on terminal failure.
- Compose `worker` service runs `celery -A afm.tasks.celery_app worker`.

**Testing**:
- `Unit: task raising transient error → retried; permanent error → dead-lettered + audit row`
- `Integration: enqueue task, worker processes, result retrievable`

#### 7.2 — Accounting provider abstraction + QuickBooks Online

**What**: `AccountingProvider` protocol and a QBO implementation syncing invoices/payments via OAuth 2.0 (PKCE).

**Design**:
```python
class AccountingProvider(Protocol):
    def authorize_url(self, firm_id: UUID) -> str: ...
    def exchange_code(self, firm_id: UUID, code: str) -> Tokens: ...
    def push_invoice(self, invoice: Invoice) -> ExternalRef: ...
    def push_payment(self, payment: Payment) -> ExternalRef: ...
```
- OAuth tokens stored encrypted per firm (`integration_credentials` table, migration).
- `integrations/accounting/quickbooks.py` maps AFM invoice → QBO Invoice; respects 500 req/min limit with backoff.
- Endpoints `GET /v1/integrations/quickbooks/connect`, `GET /v1/integrations/quickbooks/callback`, `POST /v1/integrations/quickbooks/sync`.
- Sync runs as a Celery task.

**Testing**:
- `Integration (mocked QBO API): push_invoice → external ref stored on invoice`
- `Integration (mocked): 429 from QBO → task retries with backoff`
- `Unit: AFM invoice → QBO payload mapping (line items, customer, amounts)`

#### 7.3 — Xero integration

**What**: Xero implementation of the same protocol (granular scopes, 60 req/min/tenant).

**Design**:
- `integrations/accounting/xero.py` using `xero-python`; OAuth 2.0 PKCE; requests the minimal granular scopes (accounting.transactions, accounting.contacts).
- Same connect/callback/sync endpoints under `/v1/integrations/xero`.

**Testing**:
- `Integration (mocked Xero): push_invoice → contact + invoice created`
- `Unit: scope set is minimal (no broad legacy scopes)`

#### 7.4 — Outbound webhooks

**What**: Firm-configurable webhooks for domain events (budget threshold crossed, invoice issued, phase completed).

**Design**:
- `webhook_endpoints` table (migration): `firm_id, url, secret, events[]`.
- Dispatch via Celery; HMAC-SHA256 signature header (WebSub-style); retry with backoff; delivery log.
- Event triggers wired from `events.py`.

**Testing**:
- `Integration (mock receiver): phase completed → POST delivered with valid HMAC signature`
- `Integration: receiver returns 500 → retried; permanent failure logged`

---

## Phase 8: AI-Native Capabilities

### Purpose
Deliver the differentiating AI features that no incumbent treats as core: timesheet drafting from activity, fee estimation from historical data, predictive project-health alerts, and AI-generated client reports. This is the project's primary competitive advantage and depends on all prior phases (data to learn from, async infra to run jobs).

### Tasks

#### 8.1 — LLM client abstraction and prompt management

**What**: Provider-agnostic `LLMClient` with structured-output support and versioned prompts; persist outputs to `ai_analyses`.

**Design**:
- DDL from suggestion-1 `ai_analyses` (`analysis_type` CHECK: timesheet_draft/fee_estimate/project_health_alert/client_report_draft/zoning_intelligence/burn_rate_forecast; `content`, `score`, `details` JSONB, `model_version`).
- `ai/client.py`: `complete(system, user, response_model: type[BaseModel]) -> BaseModel` using litellm; provider/model from config.
- `ai/prompts.py`: each prompt is a versioned template `(name, version, system, user_template)`; `model_version` recorded on every analysis for auditability.

**Testing**:
- `Unit (mocked LLM): structured output parsed into Pydantic model`
- `Unit: malformed LLM JSON → one repair retry, then error`
- `Integration: analysis persisted to ai_analyses with model_version + details`

#### 8.2 — AI timesheet drafting

**What**: Generate draft time entries (`source='ai_draft'`) from calendar/email activity for staff review — addressing chronic under-recording.

**Design**:
- `integrations/calendar/` pulls events for a staff member/day (Google/Microsoft via OAuth; pluggable).
- `ai/timesheet_draft.py: draft_entries(staff_id, date) -> list[TimeEntryDraft]` — LLM maps calendar/email items to likely project/phase using the staff member's recent project assignments as context.
- System prompt structure: "You draft timesheet entries for an architect. Given calendar events and the staff member's active projects/phases, propose entries with project_id, phase_id, hours, and a short description. Never exceed total available hours. Output JSON matching the schema."
- Drafts created with `source='ai_draft'`, unapproved; staff reviews/edits/approves (full lifecycle traceable per suggestion-3).
- Triggered on-demand (`POST /v1/ai/timesheet-draft`) and via nightly Celery job.

**Testing**:
- `Unit (mocked LLM + fixture calendar): 3 events → 3 drafts mapped to active phases, hours sum ≤ working day`
- `Integration: drafts persisted with source='ai_draft', approved_at NULL`
- `Integration: full lifecycle — draft → edit → approve emits audit events`

#### 8.3 — AI fee estimation

**What**: Evidence-based fee proposal from historical scope-to-fee ratios by typology.

**Design**:
- `ai/fee_estimate.py: estimate_fee(typology, construction_cost, jurisdiction, scope_notes) -> FeeEstimate` — retrieves comparable completed projects (same client type/typology) from the DB, feeds their fee/cost ratios and phase splits to the LLM, returns a recommended fee range, per-phase split, and flagged commonly-missed scope items.
- Endpoint `POST /v1/ai/fee-estimate`; result stored in `ai_analyses` and attachable to an opportunity.

**Testing**:
- `Unit (mocked LLM + fixture comparables): returns fee range, phase split summing to 100%, ≥1 flagged scope item`
- `Integration: estimate attached to opportunity; retrievable`

#### 8.4 — Predictive project-health alerts

**What**: Detect at-risk projects from phase burn velocity before overruns materialise; compute EVM SPI/CPI.

**Design**:
- `ai/health_alerts.py` + `services/budgets.py` EVM:
  - BCWS = planned value to date; BCWP = budget × percent_complete; ACWP = actual cost.
  - `SPI = BCWP/BCWS`, `CPI = BCWP/ACWP` (ANSI/EIA-748).
  - Burn velocity = labor_cost / elapsed_weeks; projected completion vs. budget.
- LLM layer summarises numeric signals into a plain-language alert + recommended action; stored as `project_health_alert`.
- Nightly Celery job `health_recompute` scans active projects; crossing 80%/100% or SPI/CPI < 0.9 raises an alert + webhook.

**Testing**:
- `Unit: BCWP=$22.5k, BCWS=$25k → SPI=0.9 (behind schedule)`
- `Unit: CPI < 0.9 → flagged at_risk`
- `Integration (mocked LLM): at-risk project → alert row + webhook dispatched`

#### 8.5 — AI client progress reports

**What**: Draft a formatted client progress report from project state.

**Design**:
- `ai/client_report.py: draft_report(project_id, period) -> str` — assembles schedule status, deliverable completion, budget consumption, then LLM drafts a client-appropriate narrative; renders via Phase 6 PDF.
- Endpoint `POST /v1/ai/client-report`; output stored as `client_report_draft`.

**Testing**:
- `Unit (mocked LLM + fixture project): report mentions current phase, % complete, upcoming deliverables`
- `Integration: report draft → PDF rendered`

---

## Phase 9: Government Compliance (DCAA/FAR), MCP Server & Hardening

### Purpose
Add the high-value, harder-to-replicate capabilities and production hardening: DCAA/FAR indirect-cost accounting (a market gap Deltek dominates), an MCP server for AI-agent access (Scoro-parity, 2026), OIDC SSO, RLS, and security/compliance posture (OWASP, NIST, GDPR, SOC 2 readiness).

### Tasks

#### 9.1 — Indirect cost accounting (DCAA/FAR)

**What**: Indirect cost pools and indirect-rate calculation for government-contract projects.

**Design**:
- DDL from suggestion-1 "Indirect Cost Accounting" `indirect_cost_pools` (`pool_type` CHECK overhead/fringe/general_admin/facilities; `allocation_base` CHECK direct_labor/total_direct_cost/direct_labor_hours; `rate_pct`).
- `services/indirect_cost.py: compute_rates(firm_id, fiscal_year) -> dict[pool, rate]` = pool_actual / allocation_base_total. Applies allowable-only expenses (FAR Part 31; `expenses.is_allowable`).
- Burdened cost on government projects = direct labor × (1 + Σ applicable pool rates).
- Endpoints `POST/GET /v1/indirect-cost-pools`, `GET /v1/firms/{id}/indirect-rates?fiscal_year=`.

**Testing**:
- `Unit: overhead pool $1.65M actual, direct labor base $1M → rate 165%`
- `Unit: unallowable (meals) expense excluded from pool actual`
- `Integration: burdened cost on gov project applies fringe+overhead+G&A rates`

#### 9.2 — MCP server

**What**: Expose project/resource/financial data over Model Context Protocol for AI agents.

**Design**:
- `afm/mcp/` server using the `mcp` Python SDK; tools: `list_projects`, `project_health`, `staff_utilisation`, `ar_aging`, `draft_timesheet`. Auth via firm-scoped token; read-mostly.
- Reuses Phase 3/8 services; enforces the same firm-scoped authorisation.

**Testing**:
- `Integration: MCP list_projects returns firm-scoped projects only`
- `Integration: MCP tool with token for firm A cannot read firm B`

#### 9.3 — OIDC SSO, RLS, and security hardening

**What**: Enterprise SSO, PostgreSQL row-level security for hard tenant isolation, and security posture per standards.md.

**Design**:
- `security/oidc.py`: OIDC code flow (Entra ID, Google, Okta) issuing internal JWTs.
- Enable PostgreSQL RLS on all firm-scoped tables: policy `firm_id = current_setting('app.firm_id')::uuid`; session sets `app.firm_id` from JWT — defence-in-depth for OWASP API1 (BOLA).
- Rate limiting (per-token), audit_log on all financial-data exports (suggestion-3 access events), GDPR right-to-erasure endpoint (`DELETE /v1/firms/{id}/data-subject/{email}`), data-residency note in config.
- Security headers, dependency scanning in CI.

**Testing**:
- `Integration: OIDC callback issues valid internal JWT`
- `Integration (RLS): query without app.firm_id set returns zero rows`
- `Integration (RLS): cross-firm query blocked at DB layer even if app filter omitted`
- `Integration: financial export writes access.financial_data_exported audit row`
- `Integration: erasure request anonymises/removes subject data`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Core Domain (auth, firm/staff)   ── required by everything
    │
Phase 2: Clients, Projects & AIA Phases                ── requires Phase 1
    │
Phase 3: Time Tracking & Budget Burn                   ── requires Phase 2   ◄── MVP value loop begins
    │
Phase 4: Expenses, Sub-Consultants & Deliverables      ── requires Phase 2/3
    │
Phase 5: Billing & Invoicing                           ── requires Phase 3/4 ◄── completes MVP
    │
    ├── Phase 6: PDF, Reporting & Client Portal         ── requires Phase 5 ── can parallel with Phase 7
    └── Phase 7: Accounting Integrations & Async Infra  ── requires Phase 5 ── can parallel with Phase 6
             │
Phase 8: AI-Native Capabilities                        ── requires Phase 7 (async) + Phase 3/5 (data)
    │
Phase 9: Gov Compliance, MCP & Hardening               ── requires Phase 5/8
```

**MVP boundary**: Phases 1–5 deliver the features.md "Must-have (MVP)" set (phase-based projects, time tracking with attribution, budget burn, invoicing in all three modes, project health dashboard). QuickBooks/Xero handoff (also MVP) lands in Phase 7.

**v1.1 (Should-have)**: Phase 5.2 (AIA G702/G703-equivalent), Phase 4.2 (sub-consultants), Phase 6.3 (client portal), Phase 6.2 (reporting + PDF), Phase 8.2 (AI timesheet drafting). Resource scheduling/capacity planning is a v1.1 add to Phase 3 (backlog task: `scheduling` service).

**Backlog (Nice-to-have)**: Phase 9.1 (DCAA/FAR), Phase 8.3 (fee estimation), Phase 8.4 (predictive health), Phase 9.2 (MCP), zoning/regulatory intelligence (new `ai/zoning.py` task), BIM integration (Autodesk APS — new `integrations/bim/` module, the largest underserved gap per features.md), full-capability mobile app.

**Parallelism opportunities**:
- Phases 6 and 7 can be developed concurrently after Phase 5.
- Within Phase 7, QuickBooks (7.2) and Xero (7.3) are independent after the provider protocol (7.1 infra + base).
- Within Phase 8, tasks 8.2–8.5 are independent after 8.1.
- Frontend dashboard work can proceed in parallel with backend from Phase 3 onward against the generated OpenAPI spec.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and mocked-integration tests pass; real-DB integration tests pass under testcontainers.
3. `ruff check` and `ruff format --check` pass; `mypy afm/` passes; frontend `eslint`/`prettier` clean.
4. `docker compose build` succeeds and the `api` + `worker` containers pass healthchecks.
5. The phase's primary workflow works end-to-end (manually verified and covered by at least one E2E or integration test).
6. New configuration options are added to `.env.example` and documented.
7. New/changed API endpoints appear correctly in the auto-generated OpenAPI spec, and `docs/openapi.json` is regenerated.
8. An Alembic migration exists for every schema change and applies cleanly on a fresh database and as an upgrade from the previous migration.
9. Firm-scoped object-level authorisation is enforced on every new resource (no BOLA regressions; covered by a cross-firm test).
10. Any new financial computation has a unit test asserting exact cent-level values.
```
