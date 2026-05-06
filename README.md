# Architecture Firm Management

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source practice management platform for architecture and engineering firms — covering project tracking, time and billing, design deliverables, and a client portal.

Architecture firms today choose between expensive enterprise ERP (Deltek), complex mid-market suites (BQE CORE), or lightweight purpose-built tools that lack accounting depth (Monograph, Factor A/E). This project aims to deliver a modern, open alternative that combines phase-based A/E workflows, AIA-aligned billing, and AI-assisted drafting of timesheets, fee estimates, and client reports — for independent practices, mid-size firms, and design-build hybrids.

---

## Why Architecture Firm Management?

- Enterprise platforms like Deltek Vantagepoint cost $150–$300/user/month with $50k+ year-one implementations and are widely described as clunky and built for accountants rather than architects.
- Mid-market and purpose-built tools (Monograph, Factor A/E, BQE CORE) each force trade-offs: limited accounting depth, weak mobile experience, basic custom reporting, or steep configuration cost.
- No surveyed tool integrates with BIM/design tools (Revit, ArchiCAD, Rhino) to pull deliverable milestones or design-hour data into the management system.
- Time capture is universally manual; chronic under-recording on project-based billing is unaddressed by current platforms.
- All incumbents are proprietary commercial SaaS — no open-source option exists in this category, leaving firms locked into vendor pricing and roadmaps.

---

## Key Features

### Project & Phase Management

- Phase-based project creation aligned to standard AIA design phases (Schematic Design, Design Development, Construction Documents, etc.)
- Sub-consultant and sub-prime tracking with cost pass-through
- Resource scheduling and capacity planning across concurrent projects
- Project templates with reusable phase structures
- Basic project health dashboard (budget consumed, time remaining, phase status)

### Time, Billing & Accounting

- Time tracking with project, phase, and role attribution
- Budget tracking with real-time phase burn visibility
- Invoice generation in fixed-fee, hourly, and percentage-complete billing modes
- AIA-style billing output (G702/G703 format or equivalent)
- QuickBooks Online and Xero integration for accounting handoff
- Optional DCAA/FAR-compliant indirect cost accounting for government contracts

### Client & Deliverable Workflow

- Client portal for invoice delivery and payment
- Retainer management with recurring task automation
- Customisable reporting with PDF export
- Deliverable milestone tracking aligned to project phases

### AI-Augmented Practice

- AI-assisted timesheet drafting from calendar and email activity
- AI fee estimation trained on historical project data
- Predictive project health alerts using phase burn velocity
- AI-generated client progress reports drawn from project state
- Regulatory and zoning intelligence at project setup (jurisdiction-specific codes, permit timelines)

---

## AI-Native Advantage

Existing tools rely entirely on manual time entry, manual fee proposals, and manual client reporting. This project applies AI to the workflows incumbents have left untouched: drafting timesheet entries from calendar, email, and design-file activity to address chronic under-recording; producing evidence-based fee proposals from historical scope-to-fee ratios; detecting early warning signals of budget overrun or schedule slip from phase burn velocity; and drafting formatted client progress reports directly from project data. Scoro has begun adding AI insights and an MCP server, but no incumbent treats AI as the core operating model.

---

## Tech Stack & Deployment

The project is intended as a cloud-deployable SaaS with self-hosting as a first-class option, integrating with industry-standard accounting (QuickBooks Online, Xero) and aligning with AIA Contract Documents (B-Series), AIA G702/G703 billing formats, Earned Value Management practices, ISO 19650 BIM standards, and FAR/DCAA compliance for firms with government contracts. Integration with BIM tools (Revit, ArchiCAD) is identified as an underserved area to address. APIs from incumbents (BQE CORE, Deltek Vantagepoint, Productive, Scoro, Accelo) provide reference patterns for REST/JSON:API design.

---

## Market Context

The professional services automation market that includes A/E firm tooling is valued at approximately USD 9.6–11.3 billion in 2026 with 13–15% CAGR. Enterprise platforms (Deltek) are quote-based at $50k+ year-one cost; mid-market platforms (BQE CORE, Productive) run $30–$70/user/month; purpose-built lighter tools (Monograph, Factor A/E) start at $15–$30/user/month. Primary buyers are small independent practices (1–10 staff), mid-size firms with multiple project managers, large multi-office firms needing government-contract compliance, and design-build hybrids.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

Note: AIA G702/G703 contract document forms are copyrighted by the American Institute of Architects. This project may support the underlying workflow and data structures but cannot reproduce the copyrighted form templates without licensing from AIA.

---

## Licence

Licence to be determined. See [discussion](#) for context.
