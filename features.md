# Architecture Firm Management — Feature & Functionality Survey

> Candidate #325 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Monograph | Cloud SaaS | Commercial — tiered subscription | https://monograph.com |
| BQE CORE | Cloud SaaS | Commercial — quote-based / tiered | https://www.bqe.com |
| Deltek Vantagepoint | Cloud + On-premise | Commercial — enterprise quote | https://www.deltek.com/en/erp/vantagepoint |
| Productive | Cloud SaaS | Commercial — tiered ($9–$28/user/mo) | https://productive.io |
| Scoro | Cloud SaaS | Commercial — from $26/user/month | https://www.scoro.com |
| Accelo | Cloud SaaS | Commercial — quote-based | https://www.accelo.com |
| Factor A/E | Cloud SaaS | Commercial — from $30/user/month | https://factorapp.com |
| ArchiOffice (BQE) | Cloud SaaS | Commercial — tiered (legacy product) | https://www.bqe.com |

---

## Feature Analysis by Solution

### Monograph

**Core features**
- Gantt-chart-based project planning with phase-level breakdown
- Real-time budget and fee tracking across projects
- Timesheet management with per-project and per-phase attribution
- Invoicing and payment collection with QuickBooks Online sync
- Resource allocation and workload dashboards
- Project templates with reusable phase structures
- Payroll export (Grow plan)
- Financial forecasting for project and firm-level cash flow
- Client and contact management

**Differentiating features**
- Purpose-built exclusively for architecture and engineering firms; UX reflects A/E workflows rather than generic project management
- Phase-level budget tracking aligned with AIA project phases
- Margin and cash flow dashboards designed around design-practice economics

**UX patterns**
- Clean, modern interface with strong onboarding; rated 90% user satisfaction
- Gantt chart as the primary navigation paradigm for projects
- Dashboards surface workload distribution and time-sheet completeness
- Progressive feature unlock across tiers (Grow plan adds forecasting)

**Integration points**
- QuickBooks Online (two-way invoice and expense sync)
- Payroll export (CSV)
- Limited native integrations; relies on Zapier/manual export for other connections

**Known gaps**
- Mobile app is weak or absent (frequently cited in user reviews)
- Time entry is difficult to edit retroactively; no calendar-view time log
- Task management is limited — no sub-tasks, requires many clicks
- Custom reporting is basic; PDF export of reports not available
- No all-in-one invoicing from estimate to billing — requires separate tools for some workflows
- No community forum or user peer-support channel
- No BIM/design tool integration

**Licence / IP notes**
- Proprietary commercial SaaS. No open-source components disclosed.

---

### BQE CORE

**Core features**
- Time and expense tracking with SmartSearch and mobile access
- Batch invoicing; billing by percent complete or time & expense
- Phase-level project management with milestones, tasks, and deliverables
- Resource planning and staffing optimisation
- CRM functionality for pipeline and client management
- Accounting module with general ledger, AP/AR
- Real-time profitability dashboards at client, project, and phase levels
- ePayments processing
- AIA-aligned invoicing support

**Differentiating features**
- One of few platforms with a full integrated accounting module (not just QuickBooks sync)
- AIA billing support native (G702/G703 style invoicing)
- Broad feature set spanning practice management, CRM, and accounting in a single product

**UX patterns**
- Comprehensive but complex; significant initial configuration required
- Desktop-first; standalone desktop time-tracking application available
- Automatic time entry reminders reduce under-recording
- Batch invoice workflow designed for high-volume billing periods

**Integration points**
- QuickBooks (import/export)
- Xero
- ePayment processors
- REST API (OAuth 2.0) — see api-explorer.bqecore.com
- GitHub SDK samples available

**Known gaps**
- Complex initial setup deters smaller firms
- Accounting depth is still lighter than dedicated accounting platforms
- Less modern UI compared to Monograph or Productive
- No native BIM integration

**Licence / IP notes**
- Proprietary commercial SaaS. REST API available to partners/customers.

---

### Deltek Vantagepoint

**Core features**
- Full project lifecycle management from pursuit through close-out
- Project-centric ERP: accounting, billing, payroll, and HR in one platform
- Resource planning with skills-based assignment and capacity forecasting
- CRM and business development pipeline
- Change order and baseline management with scenario modelling
- Government contract compliance: DCAA-ready indirect cost tracking and FAR compliance
- Multi-office and multi-currency support
- AI-powered project planning and multi-project dashboards (2026)
- AIA G702/G703 billing and construction-phase progress billing
- Detailed custom reporting and analytics

**Differentiating features**
- Only platform with a genuinely DCAA-compliant accounting system suitable for federal government A/E contracts
- 12,000+ A/E firm customer base; de-facto enterprise standard
- Full ERP replacing the need for separate accounting software

**UX patterns**
- Enterprise-grade complexity; steep learning curve for non-accountants
- Timesheets require horizontal scrolling on standard displays
- Rich drill-down reporting but requires training to configure
- Role-based dashboards; project manager vs. finance vs. executive views

**Integration points**
- REST API (vantagepointapi.deltek.com) — version 2026.2
- Deltek Marketplace ecosystem of certified partners and connectors
- Integration with Deltek Ajera, Deltek Vision (legacy migration path)
- Payroll and HR integrations

**Known gaps**
- High cost ($150–$300/user/month) and complex implementation ($50k+ year one)
- UI/UX described as "clunky" and built for accountants/engineers, not architects
- Report customisation requires significant technical knowledge
- Third-party integrations can require middleware
- Performance issues at scale
- No direct BIM/design tool integration

**Licence / IP notes**
- Proprietary enterprise software owned by Thoma Bravo. Certified partner ecosystem for extensions.

---

### Productive

**Core features**
- Project and resource planning with visual utilisation dashboards
- Time tracking with billable/non-billable hour separation
- Budget management with real-time margin visibility
- Overhead and expense management for accurate project profitability
- Revenue forecasting based on scheduled work
- Invoicing from tracked time and fixed-fee projects
- Reporting and custom dashboards
- Team capacity and scheduling across overlapping projects

**Differentiating features**
- Strongest resource and capacity planning UX in mid-market tier
- Revenue forecasting that projects future income from current resource schedules
- Open API (JSON:API spec) with comprehensive developer documentation

**UX patterns**
- Modern, clean interface; lower learning curve than enterprise alternatives
- Resource schedule board central to workflow (not Gantt-first)
- Financial widgets embedded in project views for real-time margin awareness

**Integration points**
- JSON:API REST API (developer.productive.io)
- QuickBooks, Xero, HubSpot, Slack, Zapier
- Webhooks for event-driven integrations

**Known gaps**
- Not architecture-specific; lacks AIA billing, phase-based contract structures
- No client portal
- Less depth in project accounting than BQE CORE or Deltek
- No BIM/design tool integration
- Government contract compliance (DCAA/FAR) not supported

**Licence / IP notes**
- Proprietary commercial SaaS. Open API under standard developer terms.

---

### Scoro

**Core features**
- Quote-to-cash project lifecycle management
- Portfolio-level dashboards with budget burn and resource utilisation
- Gantt and timeline views with task sequencing
- Time tracking and timesheet approval workflows
- Invoicing, recurring billing, and revenue recognition
- CRM and sales pipeline management
- Profitability tracking at role, service, and project level
- Reporting suite with customisable KPI dashboards
- AI-powered instant insights (2026)
- MCP server integration (2026)

**Differentiating features**
- Quote-to-cash in a single workflow — from initial scope to final invoice without context switching
- Role-level margin tracking enables pricing decisions based on profitability by staff role
- MCP server support positions it for AI-agent workflows

**UX patterns**
- Comprehensive but well-organised; multiple views per resource type
- Progressive disclosure with sub-tasks launching in 2026
- Strong executive-level reporting UX

**Integration points**
- REST API v2 (subdomain.scoro.com/api/v2)
- QuickBooks, Xero, Zapier, Slack
- MCP server (Model Context Protocol)

**Known gaps**
- Not architecture-specific; no AIA billing support
- No DCAA compliance
- No client portal
- No BIM integration
- Architecture-specific use cases require manual configuration of custom fields

**Licence / IP notes**
- Proprietary commercial SaaS. API available to customers.

---

### Accelo

**Core features**
- Quote-to-cash service operations platform
- Project management with milestone tracking
- Retainer management with smart task scheduling and budget tracking
- Client portal with real-time project and invoice visibility
- Ticket and request management (shared inbox)
- Time tracking with approval workflows for billing
- Invoicing and ePayments
- CRM and sales pipeline

**Differentiating features**
- Most capable client portal of any tool in this segment — clients can view project status, retainer balances, and pay invoices directly
- Retainer management is a standalone module with recurring task automation
- Architecture firms with ongoing master planning or retainer clients benefit from the retainer-centric model

**UX patterns**
- Quote-to-cash workflow as primary navigation paradigm
- Client portal is self-service and does not require separate login infrastructure to deploy
- Automation workflows for recurring retainer tasks (2026 updates)

**Integration points**
- RESTful API (api.accelo.com/docs/) — JSON, XML, and YAML
- OAuth 2.0 user grants
- Zapier, Xero, QuickBooks, Salesforce, Slack

**Known gaps**
- Not architecture-specific; lacks AIA billing and phase-based project structures
- No DCAA compliance
- Limited project accounting depth for complex multi-phase projects
- No BIM integration
- Pricing increases significantly for retainer management module (Business tier)

**Licence / IP notes**
- Proprietary commercial SaaS. Public REST API with OAuth 2.0.

---

### Factor A/E

**Core features**
- Phase and sub-phase project breakdowns aligned to A/E workflows
- Sub-consultant tracking and management
- Time entry and resource scheduling
- AIA-style invoicing (fixed fee, hourly, phased)
- Invoice customisation with client-specific views
- Deposit and progress payment tracking
- Architecture-specific KPI dashboards
- QuickBooks Online integration
- Fully assisted onboarding with no extra support fees

**Differentiating features**
- Most affordable purpose-built A/E tool at $30/user/month with full-service onboarding
- Sub-consultant management native to project structure
- AIA-aligned billing built into core, not as an add-on

**UX patterns**
- Simplified UX targeted at small to mid-size firms (1–50 staff)
- Guided onboarding process; no self-serve complexity
- Real-time dashboards focused on firm profitability and project health

**Integration points**
- QuickBooks Online
- Limited third-party integration ecosystem compared to larger platforms

**Known gaps**
- No native accounting module; fully dependent on QuickBooks for financials
- No client portal
- Limited reporting customisation
- Small integration ecosystem
- No DCAA compliance for government contracts
- No BIM integration
- Early-stage product; fewer enterprise-grade features

**Licence / IP notes**
- Proprietary commercial SaaS. Venture-backed early-stage company.

---

### ArchiOffice (BQE legacy)

**Core features**
- Project management for architecture practices
- Contact and client management
- Time and expense tracking with mobile app and offline sync
- Phase-based billing
- Reporting and document management
- Integration with BQE CORE (migration path)

**Differentiating features**
- Historically, one of the first purpose-built architecture office management tools
- Architecture-specific phase templates predating most competitors

**UX patterns**
- Older interface; BQE is actively migrating users to BQE CORE
- Mobile app with offline sync for field time entry

**Integration points**
- BQE CORE (migration/sync)
- Limited API surface

**Known gaps**
- Legacy product with no significant new feature development
- Fewer integrations than current-generation tools
- BQE is sunsetting ArchiOffice in favour of CORE
- No BIM integration
- Weak reporting

**Licence / IP notes**
- Proprietary commercial SaaS. Being phased out by BQE Software.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Time tracking with project and phase attribution (hourly, fixed-fee, percentage complete)
- Phase-based project budgeting aligned to design workflow stages (Schematic Design, Design Development, Construction Documents, etc.)
- Invoice generation from tracked time and expenses
- Basic project dashboards showing budget burn and schedule progress
- Resource allocation across concurrent projects
- QuickBooks or Xero accounting integration
- Mobile time entry

### Differentiating Features
- DCAA/FAR-compliant indirect cost accounting for government contracts (Deltek only)
- Native AIA G702/G703 billing support (BQE CORE, Factor A/E, Deltek)
- Full integrated accounting module (BQE CORE, Deltek — eliminates QuickBooks dependency)
- Client self-service portal with invoice payment (Accelo)
- Retainer management with automated recurring task scheduling (Accelo)
- Revenue forecasting from current resource schedules (Productive)
- Role-level margin tracking (Scoro)
- Sub-consultant and sub-prime tracking (Factor A/E, BQE CORE)
- MCP server / AI agent integration (Scoro 2026)

### Underserved Areas / Opportunities
- **BIM / design tool integration**: No tool integrates directly with Revit, ArchiCAD, or Rhino to pull deliverable milestones, drawing issue records, or design hours into the management system
- **Automated timesheet drafting**: All platforms rely on manual time entry; none analyse calendar, email, or design tool activity to suggest timesheet entries
- **AI-assisted fee estimation**: No tool leverages historical project data to benchmark proposed fees against comparable past projects
- **Client portal for design review**: Accelo provides a transactional portal; none provide a design-review collaboration portal (mark-up, approvals) within the PM tool
- **Predictive project health warnings**: No tool proactively flags at-risk projects based on phase burn velocity or historical overrun patterns before they materialise
- **Regulatory and zoning intelligence**: No tool surfaces jurisdiction-specific code, permit, or approval timeline information at project setup
- **Mobile experience**: Universally weak; mobile apps are limited to time entry; no meaningful project management via mobile
- **AI-generated client reporting**: Manual effort required to produce formatted client progress reports; no drafting assistance

### AI-Augmentation Candidates
- Time entry drafting from calendar/email/design tool activity logs (eliminating chronic under-recording)
- Fee estimation models trained on historical scope-to-fee ratios by project typology
- Early-warning project health monitoring using burn velocity and phase completion rates
- Natural language client report generation from project status data
- Scope change detection by comparing current hours against initial estimates at phase level
- Resource utilisation optimisation across project portfolio using predictive capacity modelling

---

## Legal & IP Summary

All tools surveyed are proprietary commercial SaaS products. No open-source or permissive-licence tools were identified in the core architecture firm management category. APIs are available for most platforms (BQE CORE, Deltek, Productive, Scoro, Accelo) under standard developer terms that permit integration but restrict redistribution. AIA contract document formats (G702/G703) are copyrighted by the American Institute of Architects; software may support the workflow and data structure but cannot reproduce the actual copyrighted form templates without licensing from AIA. No patent concerns were identified in the feature set surveyed, though Deltek's government-contract accounting methodology (indirect cost rate structures) represents deep domain expertise that would require substantial implementation work to replicate.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Phase-based project creation aligned to standard AIA design phases
- Time tracking with project, phase, and role attribution
- Budget tracking with real-time phase burn visibility
- Invoice generation: fixed-fee, hourly, and percentage-complete billing modes
- QuickBooks Online or Xero integration for accounting handoff
- Basic project health dashboard (budget consumed, time remaining, phase status)

**Should-have (v1.1)**
- AIA-style billing output (G702/G703 format or equivalent)
- Sub-consultant / sub-prime tracking with cost pass-through
- Resource scheduling and capacity planning
- Client portal for invoice delivery and payment
- AI-assisted timesheet drafting from calendar and email activity
- Customisable reporting with PDF export

**Nice-to-have (backlog)**
- DCAA/FAR-compliant indirect cost accounting module
- BIM tool integration (Revit, ArchiCAD) for deliverable milestone sync
- AI fee estimation trained on historical project data
- Predictive project health alerts with phase burn velocity analysis
- Regulatory intelligence layer (jurisdiction-specific codes and permit timelines at project setup)
- Retainer management with recurring task automation
- Mobile app with full project management capability (not just time entry)
