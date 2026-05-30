# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Architecture Firm Management · Created: 2026-05-25

## Philosophy

An architecture firm management platform orchestrates the full lifecycle of professional services engagements: firms win work through a CRM pipeline, contracts define fee structures (fixed, hourly, percentage of construction cost), projects progress through AIA phases (Schematic Design through Construction Administration), staff track time against project phases, expenses accumulate, invoices are generated in AIA G702/G703 format or standard hourly/fixed-fee modes, and revenue is recognised per ASC 606/IFRS 15. A normalized relational model gives each concept its own table with database-enforced referential integrity across the project → phase → time entry → invoice pipeline.

This maps directly to how A/E firms operate: a principal pursues opportunities, wins a contract, sets up phases with budgets, assigns staff, tracks hours and expenses, bills clients monthly, manages sub-consultants as pass-through costs, and monitors project health through earned value metrics. Each step is a table. Government contract work adds indirect cost accounting (DCAA/FAR compliance) requiring segregation of direct vs. indirect costs by pool — which is naturally modelled as separate tables for cost pools and indirect rate calculations.

**Best for:** Mid-to-large firms and government contractors requiring AIA-aligned phase structures, DCAA-compliant cost accounting, earned value tracking, and strict referential integrity across the project → billing → revenue recognition pipeline.

**Trade-offs:**
- **Pro:** Database-enforced project → phase → time → invoice pipeline
- **Pro:** AIA phase structure as first-class entities with phase-level budgets
- **Pro:** Sub-consultant costs tracked with pass-through markup
- **Pro:** DCAA-compliant direct/indirect cost segregation
- **Pro:** Earned value metrics computable from budget and time data
- **Pro:** Fee structures support fixed, hourly, and percentage-of-construction billing
- **Con:** 24+ tables — complex for a solo practice
- **Con:** High join count for project financial dashboards
- **Con:** Indirect cost accounting tables add overhead for non-government firms

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| AIA B-Series Contracts | Project fee structures and phase definitions |
| AIA G702/G703 | Invoice table supports Schedule of Values format |
| ISO 19650 | Deliverable naming and BIM integration references |
| ISO 21500 | Phase-gated project structure |
| ANSI/EIA-748 (EVM) | Earned value fields on phases (BCWS, BCWP, ACWP) |
| ASC 606 / IFRS 15 | Revenue recognition method on projects |
| FAR Part 31 | Expense categorisation (allowable/unallowable) |
| DCAA | Indirect cost pools and rate calculations |
| CAS (48 CFR 99) | Cost accounting consistency |
| OAuth 2.0 | API authentication |
| OpenAPI 3.2 | API specification |
| ISO 8601 | All timestamps as TIMESTAMPTZ |
| ISO 4217 | Currency codes |

---

## Firm & Staff

```sql
CREATE TABLE firms (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    tax_id TEXT,
    timezone TEXT NOT NULL DEFAULT 'UTC',
    currency TEXT NOT NULL DEFAULT 'USD',
    fiscal_year_start_month INT NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE offices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    address_line1 TEXT,
    city TEXT,
    state TEXT,
    postal_code TEXT,
    country TEXT NOT NULL DEFAULT 'US',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_offices_firm ON offices(firm_id);

CREATE TABLE staff (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id) ON DELETE CASCADE,
    email TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    role TEXT NOT NULL CHECK (role IN (
        'principal', 'associate', 'project_manager', 'project_architect',
        'designer', 'intern', 'admin', 'accounting', 'marketing'
    )),
    office_id UUID REFERENCES offices(id),
    hourly_cost_cents BIGINT,
    hourly_bill_rate_cents BIGINT,
    utilisation_target_pct REAL DEFAULT 65,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_staff_firm ON staff(firm_id);
```

---

## Clients & CRM

```sql
CREATE TABLE clients (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    type TEXT DEFAULT 'corporate' CHECK (type IN (
        'corporate', 'government', 'non_profit', 'individual', 'developer'
    )),
    industry TEXT,
    address_line1 TEXT,
    city TEXT,
    state TEXT,
    postal_code TEXT,
    country TEXT DEFAULT 'US',
    primary_contact_name TEXT,
    primary_contact_email TEXT,
    primary_contact_phone TEXT,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_clients_firm ON clients(firm_id);

CREATE TABLE opportunities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id) ON DELETE CASCADE,
    client_id UUID REFERENCES clients(id),
    name TEXT NOT NULL,
    stage TEXT NOT NULL DEFAULT 'lead' CHECK (stage IN (
        'lead', 'qualified', 'proposal', 'shortlisted',
        'won', 'lost', 'on_hold'
    )),
    estimated_fee_cents BIGINT,
    estimated_construction_cost_cents BIGINT,
    probability_pct REAL DEFAULT 0,
    expected_close_date DATE,
    owner_id UUID REFERENCES staff(id),
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_opps_firm ON opportunities(firm_id);
CREATE INDEX idx_opps_stage ON opportunities(stage) WHERE stage NOT IN ('won', 'lost');
```

---

## Projects & Phases

```sql
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id) ON DELETE CASCADE,
    client_id UUID NOT NULL REFERENCES clients(id),
    opportunity_id UUID REFERENCES opportunities(id),
    name TEXT NOT NULL,
    number TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
        'proposal', 'active', 'on_hold', 'completed', 'archived'
    )),
    fee_type TEXT NOT NULL DEFAULT 'fixed' CHECK (fee_type IN (
        'fixed', 'hourly', 'percentage', 'cost_plus', 'not_to_exceed'
    )),
    contract_amount_cents BIGINT,
    construction_cost_cents BIGINT,
    fee_percentage REAL,
    revenue_recognition TEXT NOT NULL DEFAULT 'percent_complete' CHECK (revenue_recognition IN (
        'percent_complete', 'cost_to_cost', 'milestones', 'time_and_materials'
    )),
    retainer_amount_cents BIGINT,
    is_government_contract BOOLEAN NOT NULL DEFAULT FALSE,
    contract_number TEXT,
    start_date DATE,
    target_end_date DATE,
    actual_end_date DATE,
    project_manager_id UUID REFERENCES staff(id),
    office_id UUID REFERENCES offices(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (firm_id, number)
);

CREATE INDEX idx_projects_firm ON projects(firm_id);
CREATE INDEX idx_projects_client ON projects(client_id);
CREATE INDEX idx_projects_status ON projects(firm_id, status)
    WHERE status IN ('active', 'on_hold');

CREATE TABLE project_phases (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    phase_code TEXT NOT NULL CHECK (phase_code IN (
        'pre_design', 'schematic_design', 'design_development',
        'construction_documents', 'bidding_negotiation',
        'construction_administration', 'post_occupancy',
        'custom'
    )),
    status TEXT NOT NULL DEFAULT 'not_started' CHECK (status IN (
        'not_started', 'in_progress', 'completed', 'on_hold'
    )),
    fee_cents BIGINT NOT NULL DEFAULT 0,
    budget_hours REAL NOT NULL DEFAULT 0,
    start_date DATE,
    end_date DATE,
    percent_complete REAL DEFAULT 0,
    sort_order INT NOT NULL DEFAULT 0,
    -- Earned Value fields
    bcws_cents BIGINT DEFAULT 0,
    bcwp_cents BIGINT DEFAULT 0,
    acwp_cents BIGINT DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_phases_project ON project_phases(project_id);
```

---

## Sub-Consultants

```sql
CREATE TABLE sub_consultants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    discipline TEXT NOT NULL,
    contact_name TEXT,
    contact_email TEXT,
    contact_phone TEXT,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_subs_firm ON sub_consultants(firm_id);

CREATE TABLE project_sub_consultants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    sub_consultant_id UUID NOT NULL REFERENCES sub_consultants(id),
    phase_id UUID REFERENCES project_phases(id),
    contract_amount_cents BIGINT NOT NULL DEFAULT 0,
    markup_pct REAL DEFAULT 0,
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_project_subs ON project_sub_consultants(project_id);
```

---

## Time & Expenses

```sql
CREATE TABLE time_entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    staff_id UUID NOT NULL REFERENCES staff(id),
    project_id UUID NOT NULL REFERENCES projects(id),
    phase_id UUID NOT NULL REFERENCES project_phases(id),
    entry_date DATE NOT NULL,
    hours REAL NOT NULL CHECK (hours > 0),
    is_billable BOOLEAN NOT NULL DEFAULT TRUE,
    description TEXT,
    source TEXT NOT NULL DEFAULT 'manual' CHECK (source IN (
        'manual', 'ai_draft', 'calendar_import', 'timer'
    )),
    approved_by UUID REFERENCES staff(id),
    approved_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_time_staff ON time_entries(staff_id, entry_date DESC);
CREATE INDEX idx_time_project ON time_entries(project_id, entry_date DESC);
CREATE INDEX idx_time_phase ON time_entries(phase_id, entry_date DESC);

CREATE TABLE expenses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    staff_id UUID REFERENCES staff(id),
    project_id UUID NOT NULL REFERENCES projects(id),
    phase_id UUID REFERENCES project_phases(id),
    sub_consultant_id UUID REFERENCES sub_consultants(id),
    category TEXT NOT NULL CHECK (category IN (
        'travel', 'printing', 'materials', 'sub_consultant',
        'equipment', 'software', 'meals', 'other'
    )),
    description TEXT NOT NULL,
    amount_cents BIGINT NOT NULL,
    is_reimbursable BOOLEAN NOT NULL DEFAULT TRUE,
    is_billable BOOLEAN NOT NULL DEFAULT TRUE,
    is_allowable BOOLEAN NOT NULL DEFAULT TRUE,
    receipt_url TEXT,
    expense_date DATE NOT NULL,
    approved_by UUID REFERENCES staff(id),
    approved_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_expenses_project ON expenses(project_id, expense_date DESC);
CREATE INDEX idx_expenses_staff ON expenses(staff_id, expense_date DESC);
```

---

## Invoices & Payments

```sql
CREATE TABLE invoices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id),
    client_id UUID NOT NULL REFERENCES clients(id),
    invoice_number TEXT NOT NULL,
    billing_method TEXT NOT NULL CHECK (billing_method IN (
        'time_and_materials', 'fixed_fee', 'percent_complete',
        'aia_g702', 'milestone'
    )),
    status TEXT NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'approved', 'sent', 'paid', 'partially_paid',
        'overdue', 'voided', 'disputed'
    )),
    period_start DATE,
    period_end DATE,
    subtotal_cents BIGINT NOT NULL DEFAULT 0,
    tax_cents BIGINT NOT NULL DEFAULT 0,
    total_cents BIGINT NOT NULL DEFAULT 0,
    amount_paid_cents BIGINT NOT NULL DEFAULT 0,
    retainer_applied_cents BIGINT NOT NULL DEFAULT 0,
    due_date DATE,
    sent_at TIMESTAMPTZ,
    paid_at TIMESTAMPTZ,
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, invoice_number)
);

CREATE INDEX idx_invoices_project ON invoices(project_id, created_at DESC);
CREATE INDEX idx_invoices_client ON invoices(client_id, created_at DESC);
CREATE INDEX idx_invoices_status ON invoices(status) WHERE status NOT IN ('paid', 'voided');

CREATE TABLE invoice_line_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id UUID NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    phase_id UUID REFERENCES project_phases(id),
    line_type TEXT NOT NULL CHECK (line_type IN (
        'labor', 'expense', 'sub_consultant', 'fixed_fee',
        'retainer', 'adjustment'
    )),
    description TEXT NOT NULL,
    quantity REAL,
    rate_cents BIGINT,
    amount_cents BIGINT NOT NULL,
    sort_order INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_invoice_lines ON invoice_line_items(invoice_id);

CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id UUID NOT NULL REFERENCES invoices(id),
    amount_cents BIGINT NOT NULL,
    payment_method TEXT CHECK (payment_method IN (
        'check', 'ach', 'wire', 'credit_card', 'other'
    )),
    reference_number TEXT,
    received_date DATE NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_invoice ON payments(invoice_id);
```

---

## Indirect Cost Accounting (DCAA/FAR)

```sql
CREATE TABLE indirect_cost_pools (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    pool_type TEXT NOT NULL CHECK (pool_type IN (
        'overhead', 'fringe', 'general_admin', 'facilities', 'custom'
    )),
    fiscal_year INT NOT NULL,
    budgeted_cents BIGINT NOT NULL DEFAULT 0,
    actual_cents BIGINT NOT NULL DEFAULT 0,
    allocation_base TEXT NOT NULL DEFAULT 'direct_labor' CHECK (allocation_base IN (
        'direct_labor', 'total_direct_cost', 'direct_labor_hours'
    )),
    rate_pct REAL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (firm_id, name, fiscal_year)
);

CREATE INDEX idx_cost_pools_firm ON indirect_cost_pools(firm_id, fiscal_year);
```

---

## Deliverables & Milestones

```sql
CREATE TABLE deliverables (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    phase_id UUID NOT NULL REFERENCES project_phases(id),
    name TEXT NOT NULL,
    deliverable_type TEXT NOT NULL CHECK (deliverable_type IN (
        'drawing_set', 'specification', 'report', 'model',
        'presentation', 'submittal', 'permit', 'other'
    )),
    status TEXT NOT NULL DEFAULT 'not_started' CHECK (status IN (
        'not_started', 'in_progress', 'review', 'completed', 'submitted'
    )),
    due_date DATE,
    completed_date DATE,
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deliverables_project ON deliverables(project_id);
CREATE INDEX idx_deliverables_phase ON deliverables(phase_id);
```

---

## Audit & AI

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL,
    user_id UUID NOT NULL,
    action TEXT NOT NULL,
    resource_type TEXT NOT NULL,
    resource_id UUID NOT NULL,
    details JSONB,
    ip_address INET,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE TABLE audit_log_2026_h1 PARTITION OF audit_log
    FOR VALUES FROM ('2026-01-01') TO ('2026-07-01');
CREATE TABLE audit_log_2026_h2 PARTITION OF audit_log
    FOR VALUES FROM ('2026-07-01') TO ('2027-01-01');

CREATE INDEX idx_audit_firm ON audit_log(firm_id, created_at DESC);

CREATE TABLE ai_analyses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID REFERENCES projects(id),
    staff_id UUID REFERENCES staff(id),
    analysis_type TEXT NOT NULL CHECK (analysis_type IN (
        'timesheet_draft', 'fee_estimate', 'project_health_alert',
        'client_report_draft', 'zoning_intelligence', 'burn_rate_forecast'
    )),
    content TEXT NOT NULL,
    score REAL,
    details JSONB,
    model_version TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_project ON ai_analyses(project_id);
```

---

## Example Queries

### Project financial summary

```sql
SELECT p.name, p.number, p.contract_amount_cents,
       COALESCE(SUM(te.hours * s.hourly_cost_cents / 100), 0) AS labor_cost_cents,
       COALESCE(SUM(e.amount_cents), 0) AS expense_total_cents,
       pp.fee_cents AS phase_budget_cents,
       pp.percent_complete
FROM projects p
JOIN project_phases pp ON pp.project_id = p.id
LEFT JOIN time_entries te ON te.phase_id = pp.id
LEFT JOIN staff s ON s.id = te.staff_id
LEFT JOIN expenses e ON e.phase_id = pp.id
WHERE p.id = 'project-uuid'
GROUP BY p.id, pp.id;
```

### Staff utilisation for month

```sql
SELECT s.name, s.utilisation_target_pct,
       SUM(te.hours) AS total_hours,
       SUM(te.hours) FILTER (WHERE te.is_billable) AS billable_hours,
       ROUND(SUM(te.hours) FILTER (WHERE te.is_billable) * 100.0 / NULLIF(SUM(te.hours), 0), 1) AS actual_util_pct
FROM staff s
LEFT JOIN time_entries te ON te.staff_id = s.id
    AND te.entry_date >= DATE_TRUNC('month', CURRENT_DATE)
WHERE s.firm_id = 'firm-uuid' AND s.is_active = TRUE
GROUP BY s.id;
```

### Accounts receivable aging

```sql
SELECT c.name AS client,
       i.invoice_number, i.total_cents, i.amount_paid_cents,
       i.total_cents - i.amount_paid_cents AS outstanding_cents,
       CURRENT_DATE - i.due_date AS days_overdue
FROM invoices i
JOIN clients c ON c.id = i.client_id
WHERE i.status IN ('sent', 'overdue', 'partially_paid')
  AND i.firm_id = 'firm-uuid'
ORDER BY days_overdue DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Firm & Staff | 3 | firms, offices, staff |
| Clients & CRM | 2 | clients, opportunities |
| Projects & Phases | 2 | projects, project_phases (with EVM fields) |
| Sub-Consultants | 2 | sub_consultants, project_sub_consultants |
| Time & Expenses | 2 | time_entries, expenses |
| Invoices & Payments | 3 | invoices, invoice_line_items, payments |
| DCAA/FAR | 1 | indirect_cost_pools |
| Deliverables | 1 | deliverables |
| Audit & AI | 2 | audit_log (partitioned), ai_analyses |
| **Total** | **18** | |

---

## Key Design Decisions

1. **AIA phase codes as check constraints** — `project_phases.phase_code` uses standard AIA phase names (schematic design, design development, construction documents, etc.) as enumerated values with a `custom` option. This aligns project structures with AIA B-Series contracts while allowing flexibility.

2. **Fee type on projects** — `projects.fee_type` supports fixed, hourly, percentage-of-construction, cost-plus, and not-to-exceed billing — the standard fee structures in AIA owner-architect agreements. Invoice generation logic branches on this field.

3. **Earned value on phases** — `project_phases` includes BCWS (budgeted cost of work scheduled), BCWP (budgeted cost of work performed), and ACWP (actual cost of work performed) columns for EVM analysis per ANSI/EIA-748. These are computed from budget, percent-complete, and actual cost data.

4. **Time entry source tracking** — `time_entries.source` distinguishes manual entry from AI-drafted, calendar-imported, and timer-based entries. This supports the AI timesheet drafting feature while maintaining an audit trail of how each entry was created.

5. **Expense allowability for government contracts** — `expenses.is_allowable` flags whether an expense is allowable under FAR Part 31 (e.g., alcohol and entertainment are unallowable). This is essential for DCAA-compliant cost accounting.

6. **Indirect cost pools** — `indirect_cost_pools` stores per-fiscal-year overhead, fringe, and G&A budgets with allocation bases. This enables indirect rate calculation for government contract billing.

7. **Sub-consultant pass-through** — `project_sub_consultants` tracks sub-consultant contracts by project and phase with markup percentage. Expenses with `category = 'sub_consultant'` link back for cost tracking.

8. **Revenue recognition method** — `projects.revenue_recognition` captures whether revenue is recognised by percentage-complete, cost-to-cost, milestones, or time-and-materials — supporting ASC 606 / IFRS 15 compliance.
