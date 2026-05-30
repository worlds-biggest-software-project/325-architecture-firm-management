# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Architecture Firm Management · Created: 2026-05-25

## Philosophy

Architecture firm management combines a structured financial pipeline (project → phase → time → invoice → payment) with highly variable project configurations. Fee structures vary per contract (fixed, hourly, percentage, cost-plus). AIA G702/G703 billing requires a Schedule of Values with line items that differ per project. Sub-consultant arrangements vary by discipline and engagement. Deliverable tracking varies by project type (residential, commercial, government). A hybrid model keeps the financial pipeline relational while absorbing project-specific configurations, CRM data, deliverable tracking, and indirect cost structures into JSONB columns.

This is well-suited to architecture firms because no two projects have the same structure: a residential renovation has 3 phases; a hospital design has 8 phases with specialty consultants; a government project adds DCAA compliance fields. Rather than normalizing every variation into separate tables, the hybrid model stores phase budgets and configurations on the project as JSONB, sub-consultant details inline, and deliverable milestones as phase-level JSONB. The relational backbone ensures time entries, invoices, and payments maintain strict integrity.

**Best for:** Small-to-mid-size firms wanting fast setup and fewer tables, where projects vary widely in structure, and where government contract accounting is not the dominant use case.

**Trade-offs:**
- **Pro:** 8 core tables vs. 18 in normalized — simpler setup and maintenance
- **Pro:** Project phases, sub-consultants, and deliverables as JSONB — infinite project variety
- **Pro:** Fee structures and Schedule of Values as project JSONB
- **Pro:** New project types require no schema migration
- **Con:** Phase-level budget queries require JSONB path extraction
- **Con:** Time entry validation against phase JSONB requires application logic
- **Con:** Indirect cost accounting in JSONB is less auditable for DCAA
- **Con:** Cross-project aggregation on JSONB fields is slower

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| AIA B-Series | Fee structures in project JSONB |
| AIA G702/G703 | Schedule of Values in invoice JSONB |
| ISO 19650 | Deliverable metadata in phase JSONB |
| ANSI/EIA-748 (EVM) | EVM fields in phase JSONB |
| ASC 606 / IFRS 15 | Revenue method on projects |
| FAR Part 31 | Expense allowability flag |
| OAuth 2.0 | API authentication |
| ISO 8601 | All timestamps as TIMESTAMPTZ |

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
    settings JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "offices": [
    --     {"id": "off-uuid", "name": "Portland HQ", "address": {...}}
    --   ],
    --   "fiscal_year_start_month": 1,
    --   "default_phase_template": [
    --     {"code": "sd", "name": "Schematic Design", "pct": 15},
    --     {"code": "dd", "name": "Design Development", "pct": 20},
    --     {"code": "cd", "name": "Construction Documents", "pct": 40},
    --     {"code": "bn", "name": "Bidding & Negotiation", "pct": 5},
    --     {"code": "ca", "name": "Construction Administration", "pct": 20}
    --   ],
    --   "indirect_cost_pools": [
    --     {"name": "Overhead", "type": "overhead", "fiscal_year": 2026, "rate_pct": 165, "base": "direct_labor"},
    --     {"name": "Fringe", "type": "fringe", "fiscal_year": 2026, "rate_pct": 35, "base": "direct_labor"}
    --   ],
    --   "bill_rate_schedule": [
    --     {"role": "principal", "rate_cents": 25000},
    --     {"role": "project_manager", "rate_cents": 18500},
    --     {"role": "designer", "rate_cents": 13000}
    --   ]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE staff (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id) ON DELETE CASCADE,
    email TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    role TEXT NOT NULL CHECK (role IN (
        'principal', 'associate', 'project_manager', 'project_architect',
        'designer', 'intern', 'admin', 'accounting', 'marketing'
    )),
    office_id UUID,
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

## Projects

```sql
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id) ON DELETE CASCADE,
    client JSONB NOT NULL DEFAULT '{}',
    -- client example:
    -- {
    --   "name": "Acme Development Corp", "type": "developer",
    --   "contact_name": "Jane Smith", "contact_email": "jane@acme.com",
    --   "contact_phone": "555-0200",
    --   "address": {"line1": "456 Market St", "city": "Portland", "state": "OR", "postal": "97201"}
    -- }
    name TEXT NOT NULL,
    number TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
        'proposal', 'active', 'on_hold', 'completed', 'archived'
    )),
    fee_structure JSONB NOT NULL DEFAULT '{}',
    -- fee_structure example for fixed fee:
    -- {"type": "fixed", "contract_amount_cents": 25000000, "retainer_cents": 2500000}
    --
    -- fee_structure for percentage of construction:
    -- {"type": "percentage", "construction_cost_cents": 500000000, "percentage": 8.5}
    --
    -- fee_structure for hourly / not-to-exceed:
    -- {"type": "not_to_exceed", "cap_cents": 30000000, "multiplier": 3.0}
    revenue_recognition TEXT NOT NULL DEFAULT 'percent_complete',
    is_government_contract BOOLEAN NOT NULL DEFAULT FALSE,
    contract_number TEXT,
    phases JSONB NOT NULL DEFAULT '[]',
    -- phases example:
    -- [
    --   {"id": "ph-uuid-1", "code": "sd", "name": "Schematic Design", "status": "completed",
    --    "fee_cents": 3750000, "budget_hours": 200, "start_date": "2026-01-15", "end_date": "2026-03-15",
    --    "percent_complete": 100, "evm": {"bcws": 3750000, "bcwp": 3750000, "acwp": 3500000}},
    --   {"id": "ph-uuid-2", "code": "dd", "name": "Design Development", "status": "in_progress",
    --    "fee_cents": 5000000, "budget_hours": 300, "start_date": "2026-03-16",
    --    "percent_complete": 45, "evm": {"bcws": 2500000, "bcwp": 2250000, "acwp": 2400000},
    --    "deliverables": [
    --      {"name": "DD Drawing Set", "type": "drawing_set", "status": "in_progress", "due_date": "2026-06-01"},
    --      {"name": "Outline Specifications", "type": "specification", "status": "not_started", "due_date": "2026-06-15"}
    --    ]},
    --   {"id": "ph-uuid-3", "code": "cd", "name": "Construction Documents", "status": "not_started",
    --    "fee_cents": 10000000, "budget_hours": 600, "percent_complete": 0}
    -- ]
    sub_consultants JSONB NOT NULL DEFAULT '[]',
    -- sub_consultants example:
    -- [
    --   {"name": "MEP Engineers Inc", "discipline": "mechanical_electrical_plumbing",
    --    "contract_cents": 5000000, "markup_pct": 10, "contact": {"name": "Bob MEP", "email": "bob@mep.com"}},
    --   {"name": "Structural Solutions", "discipline": "structural",
    --    "contract_cents": 3000000, "markup_pct": 10}
    -- ]
    opportunity JSONB,
    -- opportunity example:
    -- {"stage": "won", "estimated_fee_cents": 25000000, "probability_pct": 100,
    --  "owner_id": "staff-uuid", "won_date": "2025-12-15"}
    project_manager_id UUID REFERENCES staff(id),
    office_id UUID,
    start_date DATE,
    target_end_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (firm_id, number)
);

CREATE INDEX idx_projects_firm ON projects(firm_id);
CREATE INDEX idx_projects_status ON projects(firm_id, status)
    WHERE status IN ('active', 'on_hold');
CREATE INDEX idx_projects_phases ON projects USING GIN (phases jsonb_path_ops);
```

---

## Time Entries

```sql
CREATE TABLE time_entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    staff_id UUID NOT NULL REFERENCES staff(id),
    project_id UUID NOT NULL REFERENCES projects(id),
    phase_id UUID NOT NULL,
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
CREATE INDEX idx_time_phase ON time_entries(project_id, phase_id, entry_date DESC);
```

---

## Expenses

```sql
CREATE TABLE expenses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    staff_id UUID REFERENCES staff(id),
    project_id UUID NOT NULL REFERENCES projects(id),
    phase_id UUID,
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
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_expenses_project ON expenses(project_id, expense_date DESC);
```

---

## Invoices & Payments

```sql
CREATE TABLE invoices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id),
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
    line_items JSONB NOT NULL DEFAULT '[]',
    -- line_items for time_and_materials:
    -- [
    --   {"type": "labor", "phase": "Design Development", "staff": "Jane Doe", "role": "project_architect",
    --    "hours": 42, "rate_cents": 15000, "amount_cents": 630000},
    --   {"type": "labor", "phase": "Design Development", "staff": "Bob Smith", "role": "designer",
    --    "hours": 80, "rate_cents": 13000, "amount_cents": 1040000},
    --   {"type": "expense", "description": "Printing - DD set review", "amount_cents": 45000},
    --   {"type": "sub_consultant", "name": "MEP Engineers", "amount_cents": 250000, "markup_cents": 25000}
    -- ]
    --
    -- line_items for aia_g702 (Schedule of Values):
    -- [
    --   {"item": 1, "description": "Schematic Design", "scheduled_value_cents": 3750000,
    --    "previous_pct": 100, "previous_cents": 3750000, "this_period_pct": 0, "this_period_cents": 0,
    --    "total_completed_pct": 100, "total_completed_cents": 3750000, "balance_cents": 0},
    --   {"item": 2, "description": "Design Development", "scheduled_value_cents": 5000000,
    --    "previous_pct": 30, "previous_cents": 1500000, "this_period_pct": 15, "this_period_cents": 750000,
    --    "total_completed_pct": 45, "total_completed_cents": 2250000, "balance_cents": 2750000}
    -- ]
    subtotal_cents BIGINT NOT NULL DEFAULT 0,
    tax_cents BIGINT NOT NULL DEFAULT 0,
    total_cents BIGINT NOT NULL DEFAULT 0,
    amount_paid_cents BIGINT NOT NULL DEFAULT 0,
    retainer_applied_cents BIGINT NOT NULL DEFAULT 0,
    due_date DATE,
    sent_at TIMESTAMPTZ,
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, invoice_number)
);

CREATE INDEX idx_invoices_project ON invoices(project_id, created_at DESC);
CREATE INDEX idx_invoices_status ON invoices(status) WHERE status NOT IN ('paid', 'voided');

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

## Audit Log

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
```

---

## Example Queries

### Project budget burn by phase

```sql
SELECT phase->>'name' AS phase_name,
       (phase->>'fee_cents')::BIGINT AS budget_cents,
       (phase->>'percent_complete')::REAL AS pct_complete,
       COALESCE(SUM(te.hours * s.hourly_cost_cents / 100), 0) AS actual_cost_cents
FROM projects p,
     jsonb_array_elements(p.phases) AS phase
LEFT JOIN time_entries te ON te.project_id = p.id AND te.phase_id = (phase->>'id')::UUID
LEFT JOIN staff s ON s.id = te.staff_id
WHERE p.id = 'project-uuid'
GROUP BY phase->>'name', phase->>'fee_cents', phase->>'percent_complete';
```

### Staff utilisation

```sql
SELECT s.name, s.utilisation_target_pct,
       SUM(te.hours) AS total_hours,
       SUM(te.hours) FILTER (WHERE te.is_billable) AS billable_hours,
       ROUND(SUM(te.hours) FILTER (WHERE te.is_billable) * 100.0 / NULLIF(SUM(te.hours), 0), 1) AS util_pct
FROM staff s
LEFT JOIN time_entries te ON te.staff_id = s.id
    AND te.entry_date >= DATE_TRUNC('month', CURRENT_DATE)
WHERE s.firm_id = 'firm-uuid' AND s.is_active = TRUE
GROUP BY s.id;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Firm & Staff | 2 | firms (offices, phases template, cost pools, rates in JSONB), staff |
| Projects | 1 | projects (client, phases, sub-consultants, opportunity in JSONB) |
| Time & Expenses | 2 | time_entries, expenses |
| Invoices & Payments | 2 | invoices (line items in JSONB), payments |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **8** | |

---

## Key Design Decisions

1. **Project phases as JSONB** — Phases with budgets, EVM metrics, deliverables, and dates live as a JSONB array on the project. This handles the variable number of phases per project (3 for a small residential project, 8+ for a hospital) without a join table. Phase IDs are UUIDs for time entry cross-referencing.

2. **Client inline on project** — Client details are stored as JSONB on each project rather than in a separate clients table. This simplifies the schema for firms that don't need CRM functionality. For firms that do, a separate clients table can be added later.

3. **Fee structures as JSONB** — `fee_structure` absorbs fixed, hourly, percentage, and cost-plus fee arrangements as typed JSONB documents. Invoice generation reads the fee structure to determine billing logic.

4. **AIA G702 Schedule of Values as invoice JSONB** — Invoice `line_items` for AIA billing use a Schedule of Values format with previous/current/total completion percentages per phase. This matches the G702/G703 workflow without a dedicated table.

5. **Sub-consultants inline on projects** — Sub-consultant contracts, markup, and contact details live in the project's `sub_consultants` JSONB array. Expenses with `category = 'sub_consultant'` track actual costs against these contracts.

6. **Indirect cost pools on firm settings** — DCAA/FAR indirect cost pool definitions live in `firms.settings`. For firms that don't need government contract accounting, these fields are simply empty. For those that do, the JSONB structure is less auditable than relational tables but simpler to manage.

7. **Deliverables inline on phases** — Each phase in the JSONB array can include a `deliverables` sub-array tracking drawing sets, specs, and submittals with status and due dates. This avoids a separate deliverables table for what is essentially phase-level metadata.

8. **Time entry source tracking** — `time_entries.source` distinguishes manual vs. AI-drafted vs. calendar-imported entries, maintaining the audit trail for AI timesheet assistance.
