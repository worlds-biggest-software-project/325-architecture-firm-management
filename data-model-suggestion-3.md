# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Architecture Firm Management · Created: 2026-05-25

## Philosophy

Architecture firm management involves a project lifecycle with many state transitions that benefit from full traceability: opportunities progress through a sales pipeline, contracts are negotiated and signed, phases start and complete, time is entered and approved, invoices are drafted and sent, payments arrive, and change orders modify scope and budget. For firms with government contracts, DCAA audits require demonstrable cost tracking with an unbroken chain from time entry to invoice to payment. An event-sourced architecture stores every state change as an immutable event.

This approach solves several problems specific to A/E firms. Budget overruns are the most common project failure — event-sourced burn tracking means you can replay the exact sequence of time entries, expense approvals, and scope changes that led to a budget breach. Fee disputes with clients can be resolved by replaying the invoice lifecycle: what was billed, when it was sent, what was paid, and when change orders adjusted the contract. DCAA auditors need traceable time entry audit trails — the event store provides immutable proof that time was entered, approved, and invoiced in the correct sequence.

For multi-office firms, the event stream enables real-time dashboards: firm-wide utilisation, project health across offices, revenue forecasting, and pipeline analytics — all computed by projecting events into materialised read models.

**Best for:** Multi-office firms and government contractors requiring DCAA-auditable time entry trails, change order tracking, fee dispute resolution from invoice history, and real-time firm-wide KPI dashboards derived from event projections.

**Trade-offs:**
- **Pro:** Complete, immutable audit trail for DCAA compliance
- **Pro:** Budget overrun forensics from time and expense event replay
- **Pro:** Fee disputes resolvable from invoice lifecycle events
- **Pro:** Change orders tracked as events with scope/budget impact
- **Pro:** Firm-wide KPI dashboards from event projections
- **Pro:** AI timesheet drafting traceable (draft → reviewed → approved)
- **Con:** Read models must be maintained and rebuilt
- **Con:** Higher write amplification per time entry
- **Con:** Event schema evolution requires versioning
- **Con:** CQRS learning curve for the team

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| AIA B-Series | Contract events carry fee structure details |
| AIA G702/G703 | Invoice events carry Schedule of Values data |
| ISO 19650 | Deliverable events include BIM references |
| ANSI/EIA-748 (EVM) | EVM metrics computed from budget and time events |
| ASC 606 / IFRS 15 | Revenue recognition events track method and amounts |
| FAR Part 31 | Expense events flag allowable/unallowable |
| DCAA | Immutable event store satisfies audit trail requirement |
| CloudEvents v1.0 | Event envelope spec |
| ISO 8601 | All timestamps as TIMESTAMPTZ |

---

## Event Store

```sql
CREATE TABLE event_store (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type TEXT NOT NULL,
    stream_id UUID NOT NULL,
    event_type TEXT NOT NULL,
    event_data JSONB NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}',
    sequence_number BIGINT NOT NULL,
    ce_source TEXT NOT NULL DEFAULT '/architecture-firm',
    ce_specversion TEXT NOT NULL DEFAULT '1.0',
    created_by UUID NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_type, stream_id, sequence_number)
) PARTITION BY RANGE (created_at);

CREATE TABLE event_store_2026_h1 PARTITION OF event_store
    FOR VALUES FROM ('2026-01-01') TO ('2026-07-01');
CREATE TABLE event_store_2026_h2 PARTITION OF event_store
    FOR VALUES FROM ('2026-07-01') TO ('2027-01-01');

CREATE INDEX idx_events_stream ON event_store(stream_type, stream_id, sequence_number);
CREATE INDEX idx_events_type ON event_store(event_type, created_at DESC);
CREATE INDEX idx_events_creator ON event_store(created_by, created_at DESC);
CREATE INDEX idx_events_data ON event_store USING GIN (event_data jsonb_path_ops);
```

---

## Event Type Registry

```sql
CREATE TABLE event_types (
    event_type TEXT PRIMARY KEY,
    stream_type TEXT NOT NULL,
    description TEXT NOT NULL,
    schema_version INT NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

INSERT INTO event_types (event_type, stream_type, description) VALUES
-- Opportunity stream
('opportunity.created', 'opportunity', 'New opportunity in pipeline'),
('opportunity.stage_changed', 'opportunity', 'Pipeline stage advanced/regressed'),
('opportunity.won', 'opportunity', 'Opportunity won — project created'),
('opportunity.lost', 'opportunity', 'Opportunity lost with reason'),

-- Project stream
('project.created', 'project', 'Project set up with fee structure'),
('project.phase_added', 'project', 'Phase added with budget'),
('project.phase_started', 'project', 'Phase status changed to in_progress'),
('project.phase_completed', 'project', 'Phase marked complete'),
('project.phase_budget_updated', 'project', 'Phase budget or hours changed'),
('project.percent_complete_updated', 'project', 'Phase completion percentage updated'),
('project.sub_consultant_added', 'project', 'Sub-consultant engaged'),
('project.sub_consultant_invoiced', 'project', 'Sub-consultant invoice received'),
('project.change_order_issued', 'project', 'Scope/fee change order documented'),
('project.deliverable_submitted', 'project', 'Deliverable milestone submitted'),
('project.deliverable_approved', 'project', 'Deliverable accepted by client'),
('project.status_changed', 'project', 'Project status changed'),
('project.completed', 'project', 'Project marked complete'),
('project.archived', 'project', 'Project archived'),

-- Time stream
('time.entered', 'time', 'Time entry recorded'),
('time.ai_drafted', 'time', 'AI generated timesheet draft'),
('time.reviewed', 'time', 'Staff reviewed AI draft'),
('time.modified', 'time', 'Time entry modified'),
('time.approved', 'time', 'Time entry approved by manager'),
('time.rejected', 'time', 'Time entry rejected with reason'),
('time.deleted', 'time', 'Time entry deleted with reason'),

-- Expense stream
('expense.submitted', 'expense', 'Expense submitted'),
('expense.approved', 'expense', 'Expense approved'),
('expense.rejected', 'expense', 'Expense rejected'),
('expense.classified', 'expense', 'Expense classified as allowable/unallowable'),

-- Invoice stream
('invoice.drafted', 'invoice', 'Invoice draft created'),
('invoice.line_added', 'invoice', 'Line item added'),
('invoice.approved', 'invoice', 'Invoice approved for sending'),
('invoice.sent', 'invoice', 'Invoice sent to client'),
('invoice.viewed', 'invoice', 'Client viewed invoice'),
('invoice.disputed', 'invoice', 'Client disputed invoice'),
('invoice.paid', 'invoice', 'Full payment received'),
('invoice.partially_paid', 'invoice', 'Partial payment received'),
('invoice.voided', 'invoice', 'Invoice voided'),
('invoice.overdue', 'invoice', 'Invoice passed due date'),

-- Revenue stream
('revenue.recognised', 'revenue', 'Revenue recognised per ASC 606'),
('revenue.deferred', 'revenue', 'Revenue deferred'),
('revenue.adjusted', 'revenue', 'Revenue adjustment'),

-- Staff stream
('staff.joined', 'staff', 'Staff member added'),
('staff.role_changed', 'staff', 'Role updated'),
('staff.rate_changed', 'staff', 'Bill rate or cost rate changed'),
('staff.deactivated', 'staff', 'Staff deactivated'),

-- Access stream
('access.project_viewed', 'access', 'User viewed project data'),
('access.financial_data_exported', 'access', 'Financial data exported'),
('access.report_generated', 'access', 'Report generated');
```

---

## Stream Snapshots & Projection Checkpoints

```sql
CREATE TABLE stream_snapshots (
    stream_type TEXT NOT NULL,
    stream_id UUID NOT NULL,
    snapshot_data JSONB NOT NULL,
    last_sequence_number BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_type, stream_id)
);

CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id UUID NOT NULL,
    last_event_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Read Model: Projects

```sql
CREATE TABLE rm_projects (
    id UUID PRIMARY KEY,
    firm_id UUID NOT NULL,
    client JSONB NOT NULL DEFAULT '{}',
    name TEXT NOT NULL,
    number TEXT NOT NULL,
    status TEXT NOT NULL,
    fee_structure JSONB NOT NULL DEFAULT '{}',
    phases JSONB NOT NULL DEFAULT '[]',
    sub_consultants JSONB NOT NULL DEFAULT '[]',
    change_orders JSONB NOT NULL DEFAULT '[]',
    -- change_orders example:
    -- [
    --   {"number": 1, "date": "2026-05-15", "description": "Added parking structure scope",
    --    "fee_impact_cents": 5000000, "approved_by": "Jane Smith", "approved_at": "2026-05-18"}
    -- ]
    deliverables JSONB NOT NULL DEFAULT '[]',
    total_budget_cents BIGINT NOT NULL DEFAULT 0,
    total_spent_cents BIGINT NOT NULL DEFAULT 0,
    total_invoiced_cents BIGINT NOT NULL DEFAULT 0,
    total_collected_cents BIGINT NOT NULL DEFAULT 0,
    project_manager_id UUID,
    office_id UUID,
    start_date DATE,
    target_end_date DATE,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (firm_id, number)
);

CREATE INDEX idx_rm_projects_firm ON rm_projects(firm_id);
CREATE INDEX idx_rm_projects_status ON rm_projects(firm_id, status)
    WHERE status IN ('active', 'on_hold');
```

---

## Read Model: Timesheets

```sql
CREATE TABLE rm_timesheets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL,
    staff_id UUID NOT NULL,
    staff_name TEXT NOT NULL,
    week_start DATE NOT NULL,
    entries JSONB NOT NULL DEFAULT '[]',
    -- entries example:
    -- [
    --   {"id": "entry-uuid", "project_id": "proj-uuid", "project_name": "City Hall Renovation",
    --    "phase": "Construction Documents", "date": "2026-05-19",
    --    "hours": 6.5, "billable": true, "description": "Floor plan revisions",
    --    "source": "ai_draft", "status": "approved"}
    -- ]
    total_hours REAL NOT NULL DEFAULT 0,
    billable_hours REAL NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'in_progress' CHECK (status IN (
        'in_progress', 'submitted', 'approved', 'rejected'
    )),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (staff_id, week_start)
);

CREATE INDEX idx_rm_timesheets_staff ON rm_timesheets(staff_id, week_start DESC);
```

---

## Read Model: Invoices

```sql
CREATE TABLE rm_invoices (
    id UUID PRIMARY KEY,
    project_id UUID NOT NULL,
    project_name TEXT NOT NULL,
    client_name TEXT NOT NULL,
    invoice_number TEXT NOT NULL,
    billing_method TEXT NOT NULL,
    status TEXT NOT NULL,
    line_items JSONB NOT NULL DEFAULT '[]',
    subtotal_cents BIGINT NOT NULL DEFAULT 0,
    total_cents BIGINT NOT NULL DEFAULT 0,
    amount_paid_cents BIGINT NOT NULL DEFAULT 0,
    outstanding_cents BIGINT NOT NULL DEFAULT 0,
    due_date DATE,
    timeline JSONB NOT NULL DEFAULT '[]',
    -- timeline example:
    -- [
    --   {"event": "invoice.drafted", "at": "2026-05-20", "by": "staff-uuid"},
    --   {"event": "invoice.approved", "at": "2026-05-21", "by": "staff-uuid"},
    --   {"event": "invoice.sent", "at": "2026-05-22"},
    --   {"event": "invoice.viewed", "at": "2026-05-23"},
    --   {"event": "invoice.partially_paid", "at": "2026-06-01", "amount_cents": 500000}
    -- ]
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_inv_project ON rm_invoices(project_id, created_at DESC);
CREATE INDEX idx_rm_inv_status ON rm_invoices(status)
    WHERE status NOT IN ('paid', 'voided');
```

---

## Read Model: Firm Dashboard

```sql
CREATE TABLE rm_firm_dashboard (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL,
    dashboard_date DATE NOT NULL,
    metrics JSONB NOT NULL DEFAULT '{}',
    -- metrics example:
    -- {
    --   "utilisation": {"target_pct": 65, "actual_pct": 62, "total_hours": 1240, "billable_hours": 769},
    --   "pipeline": {"total_opportunities": 12, "weighted_value_cents": 15000000,
    --     "stages": {"lead": 3, "qualified": 4, "proposal": 3, "shortlisted": 2}},
    --   "revenue": {"mtd_invoiced_cents": 8500000, "mtd_collected_cents": 6200000,
    --     "ar_outstanding_cents": 12300000, "ar_overdue_cents": 4500000},
    --   "projects": {"active": 18, "on_hold": 3, "at_risk": 2},
    --   "staffing": {"total": 45, "billable": 32, "under_target": 8}
    -- }
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (firm_id, dashboard_date)
);

CREATE INDEX idx_rm_dashboard ON rm_firm_dashboard(firm_id, dashboard_date DESC);
```

---

## Read Model: Project Health

```sql
CREATE TABLE rm_project_health (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL UNIQUE,
    project_name TEXT NOT NULL,
    project_number TEXT NOT NULL,
    firm_id UUID NOT NULL,
    client_name TEXT NOT NULL,
    status TEXT NOT NULL,
    fee_type TEXT NOT NULL,
    contract_cents BIGINT,
    budget_consumed_pct REAL,
    schedule_health TEXT CHECK (schedule_health IN ('on_track', 'at_risk', 'behind')),
    budget_health TEXT CHECK (budget_health IN ('on_track', 'at_risk', 'over_budget')),
    current_phase TEXT,
    phase_metrics JSONB NOT NULL DEFAULT '[]',
    -- phase_metrics example:
    -- [
    --   {"phase": "Design Development", "budget_hours": 300, "actual_hours": 145,
    --    "budget_cents": 5000000, "actual_cents": 2400000, "pct_complete": 45,
    --    "spi": 0.94, "cpi": 1.04}
    -- ]
    burn_rate_weekly_cents BIGINT,
    estimated_completion_date DATE,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_health_firm ON rm_project_health(firm_id);
CREATE INDEX idx_rm_health_risk ON rm_project_health(budget_health)
    WHERE budget_health IN ('at_risk', 'over_budget');
```

---

## Example Queries

### Budget overrun forensics

```sql
SELECT event_type, event_data, created_by, created_at
FROM event_store
WHERE stream_type = 'time'
  AND stream_id IN (
      SELECT id FROM rm_timesheets
      WHERE EXISTS (
          SELECT 1 FROM jsonb_array_elements(entries) e
          WHERE (e->>'project_id') = 'project-uuid'
      )
  )
ORDER BY created_at;
```

### Change order history for project

```sql
SELECT event_data, created_by, created_at
FROM event_store
WHERE stream_type = 'project'
  AND stream_id = 'project-uuid'
  AND event_type = 'project.change_order_issued'
ORDER BY sequence_number;
```

### Invoice lifecycle replay

```sql
SELECT event_type, event_data, created_at
FROM event_store
WHERE stream_type = 'invoice'
  AND stream_id = 'invoice-uuid'
ORDER BY sequence_number;
```

### AI timesheet draft audit trail

```sql
SELECT event_type, event_data, created_at
FROM event_store
WHERE stream_type = 'time'
  AND stream_id = 'time-entry-uuid'
ORDER BY sequence_number;
-- Shows: time.ai_drafted → time.reviewed → time.modified → time.approved
```

### Firm-wide project health

```sql
SELECT project_name, project_number, client_name,
       budget_health, schedule_health,
       budget_consumed_pct, burn_rate_weekly_cents
FROM rm_project_health
WHERE firm_id = 'firm-uuid'
  AND status = 'active'
ORDER BY
    CASE budget_health WHEN 'over_budget' THEN 1 WHEN 'at_risk' THEN 2 ELSE 3 END;
```

---

## Event-Driven Automation Patterns

### Budget Alert

When `time.approved` or `expense.approved` fires, the project health projection recalculates `budget_consumed_pct`. When it crosses 80% or 100%, a `project.budget_alert` notification is emitted.

### EVM Auto-Compute

When `project.percent_complete_updated` fires, the EVM projection recalculates BCWP and computes SPI (Schedule Performance Index) and CPI (Cost Performance Index) from the latest time and expense events.

### Revenue Recognition

When `invoice.paid` fires, the revenue projection checks the project's revenue recognition method. For percent-complete, it computes recognisable revenue from the latest completion percentage. Revenue events are emitted for accounting integration.

### AI Timesheet Lifecycle

When `time.ai_drafted` fires from calendar/email analysis, the draft appears in the staff member's timesheet. The `time.reviewed` event records that the staff member saw the draft. `time.modified` captures any changes. `time.approved` locks the entry. The full lifecycle is auditable for DCAA compliance.

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 3 | event_store (partitioned), event_types, stream_snapshots |
| Projection Infrastructure | 1 | projection_checkpoints |
| Read Model: Projects | 1 | rm_projects (phases, subs, change orders, deliverables) |
| Read Model: Timesheets | 1 | rm_timesheets (weekly per staff member) |
| Read Model: Invoices | 1 | rm_invoices (timeline) |
| Read Model: Firm Dashboard | 1 | rm_firm_dashboard (daily aggregate KPIs) |
| Read Model: Project Health | 1 | rm_project_health (EVM, burn rate, risk) |
| **Total** | **10** | 4 infrastructure + 6 read models |

---

## Key Design Decisions

1. **Change orders as project events** — `project.change_order_issued` events carry scope description, fee impact, and approval details. The `rm_projects` read model accumulates change orders in a JSONB array. This creates an irrefutable record for fee disputes: the client agreed to the scope change, the fee was adjusted, and the budget was updated — all traceable.

2. **AI timesheet lifecycle as events** — AI-drafted timesheet entries go through a full lifecycle: `time.ai_drafted` → `time.reviewed` → `time.modified` → `time.approved`. Each step is an event with who did what and when. DCAA auditors can verify that time entries were not fabricated or modified without review.

3. **EVM as computed from events** — Rather than manually updating earned value fields, the projection computes BCWS, BCWP, and ACWP from phase budget events, percent-complete events, and approved time/expense events. SPI and CPI are derived values on the `rm_project_health` read model.

4. **Invoice timeline for dispute resolution** — Every invoice state transition is an event: drafted, approved, sent, viewed by client, disputed, paid. The `rm_invoices` read model includes a `timeline` JSONB array. When a client disputes an invoice, the firm can show exactly what was sent, when it was viewed, and what the line items covered.

5. **Firm dashboard as a daily read model** — `rm_firm_dashboard` stores one row per firm per day with utilisation, pipeline, revenue, and project health metrics. This avoids expensive real-time aggregation across all event streams for the KPIs that firm principals check daily.

6. **Project health with risk classification** — `rm_project_health` computes `budget_health` and `schedule_health` from EVM metrics. Projects classified as `at_risk` or `over_budget` surface in executive dashboards without querying the event store.

7. **Expense classification events** — `expense.classified` events track whether an expense is flagged as allowable or unallowable per FAR Part 31. This separation is critical for government contract accounting where unallowable costs must be excluded from indirect rate calculations.

8. **Opportunity pipeline as events** — CRM stage transitions are events, enabling win-rate analytics and pipeline velocity dashboards from event projections.
