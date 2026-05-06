# Standards & API Reference

> Project: Architecture Firm Management · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 19650 (Parts 1–6) — Organization and Digitization of Information about Buildings and Civil Engineering Works (BIM)**
- URL: https://www.iso.org/standard/68078.html
- Defines the information management framework for building information modelling (BIM) across the full asset lifecycle: concepts and principles (Part 1), delivery phase (Part 2), operational phase (Part 3), information exchange (Part 4), security (Part 5), and health and safety (Part 6). Architecture firm management software must accommodate ISO 19650-aligned deliverable structures, Common Data Environment (CDE) integration, and information container naming conventions. Draft amendments for Parts 1 and 2 are due mid-2026.

**ISO 9001:2015 — Quality Management Systems**
- URL: https://www.iso.org/standard/62085.html
- Provides the framework for quality assurance in professional services firms; architecture practices seeking certification need their project management systems to support documented procedures, non-conformance tracking, and audit trails. Practice management software that supports configurable workflows and audit logs facilitates ISO 9001 compliance.

**ISO 21500:2021 — Project, Programme and Portfolio Management**
- URL: https://www.iso.org/standard/75704.html
- International guidance on project management principles and practices; provides the conceptual underpinning for phase-gated project structures, change management, and earned value reporting that architecture firm management platforms should implement.

**ISO 4157 — Construction Drawings — Designation Systems**
- URL: https://www.iso.org/standard/13556.html
- Defines numbering and labelling conventions for architectural drawings, rooms, and building elements; relevant for deliverable management modules that track drawing packages and submittal registers.

---

### W3C & IETF Standards

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- De-facto standard for API authentication in SaaS applications; all major architecture firm management platforms (BQE CORE, Deltek, Accelo, Productive) use OAuth 2.0 for third-party API access. Any new platform must implement OAuth 2.0 to support accounting, calendar, and design-tool integrations.

**RFC 6750 — The OAuth 2.0 Authorization Framework: Bearer Token Usage**
- URL: https://datatracker.ietf.org/doc/html/rfc6750
- Companion to RFC 6749; defines how bearer tokens are transmitted in API requests. Required reading for any API integration layer.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Standard for encoding claims between parties as a JSON object signed with a digital signature; widely used for session tokens and API authentication in professional services SaaS. Productive and Scoro use JWT-based API tokens.

**RFC 7231 — HTTP/1.1 Semantics and Content**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Foundational HTTP standard governing request methods, status codes, and headers used by all RESTful APIs in this space.

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Authentication layer on top of OAuth 2.0; required for SSO integration with enterprise identity providers (Microsoft Entra ID, Google Workspace, Okta). Large architecture firms expect SSO as table stakes for enterprise-tier software.

**W3C Web Hooks / HTTP Callbacks (Unofficial but widely adopted)**
- URL: https://www.w3.org/TR/websub/
- Webhook patterns (HTTP POST callbacks) are used by Productive, Accelo, QuickBooks, and Xero to push real-time event notifications to integrating applications. Any new platform should provide outbound webhooks for project events (budget threshold crossed, invoice issued, phase completed).

---

### Data Model & API Specifications

**OpenAPI Specification 3.2 (OAS 3.2)**
- URL: https://www.openapis.org/ · https://swagger.io/specification/
- The industry standard for describing RESTful APIs in a machine-readable YAML/JSON document; enables auto-generated SDK client code, interactive documentation, and contract testing. Version 3.2.0 (September 2025) adds streaming media type support and OAuth 2.0 Device Authorization Flow. All new architecture firm management APIs should publish an OAS 3.2 specification.

**JSON:API Specification**
- URL: https://jsonapi.org/
- Standard for structuring JSON API requests and responses including relationships, pagination, filtering, and sorting; adopted by Productive.io. Reduces API client complexity by standardising how related resources (projects → phases → time entries) are included and traversed.

**iCalendar (RFC 5545) — Internet Calendaring and Scheduling**
- URL: https://datatracker.ietf.org/doc/html/rfc5545
- Standard format for calendar data exchange; relevant for exporting project milestones and deadlines to Google Calendar, Microsoft Outlook, and Apple Calendar — a frequently requested feature in architecture firm management tools.

**vCard (RFC 6350) — vCard Format Specification**
- URL: https://datatracker.ietf.org/doc/html/rfc6350
- Standard for contact data exchange; relevant for CRM and client management modules that need to import/export contacts from address book applications and email clients.

---

### Security & Authentication Standards

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/API-Security/editions/2023/en/0x00-header/
- The authoritative reference for API security risks; architecture firm management platforms handle sensitive financial, contractual, and project data, making API security a critical concern. Key risks include broken object-level authorisation (accessing another firm's project data), excessive data exposure, and injection attacks.

**NIST Cybersecurity Framework (CSF) 2.0**
- URL: https://www.nist.gov/cyberframework
- Provides a voluntary framework for managing cybersecurity risk; increasingly required by enterprise architecture clients and government contractors as a contractual condition. DCAA-compliant platforms must demonstrate controls aligned with NIST guidelines.

**GDPR — General Data Protection Regulation (EU) 2016/679**
- URL: https://gdpr-info.eu/
- Applies to architecture firms with European clients or EU-based staff; practice management software must support data residency controls, right-to-erasure workflows, and data processing agreements. Cloud-hosted platforms must clearly document data storage regions and subprocessors.

**SOC 2 Type II**
- URL: https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-greater-than-soc-2
- The de-facto security audit standard for B2B SaaS; enterprise architecture clients and government contractors typically require SOC 2 Type II reports from their software vendors. Deltek and BQE CORE publish SOC 2 reports; new entrants should target certification to win enterprise accounts.

---

### Industry-Specific Standards & Frameworks

**AIA Contract Documents — B-Series (Owner-Architect Agreements)**
- URL: https://www.aiacontracts.com/
- The AIA B-series contracts (B101, B201, etc.) define the legal structure of architect-client engagements, including fee types (hourly, lump sum, percentage of construction cost), phase deliverables, and scope of services. Practice management software must support fee structures and billing milestones that map to these contract forms.

**AIA G702/G703 — Application and Certificate for Payment**
- URL: https://aiacontracts.com/documents/g702-1992 · https://learn.aiacontracts.com/articles/completing-g702-and-g703-forms/
- G702 is the standard payment application form (summary of amounts due), and G703 is the continuation sheet (Schedule of Values line-item breakdown). These are the industry-standard billing documents for architect-client invoicing on construction projects. AIA billing support is a key differentiator among competing platforms.

**Federal Acquisition Regulation (FAR) — 48 CFR**
- URL: https://www.acquisition.gov/far/
- Governs all federal government procurement; architecture firms with federal contracts must comply with FAR cost principles (Part 31) for allowable/unallowable costs, indirect rate structures, and cost accounting. Practice management accounting modules must support FAR-compliant indirect cost segregation to qualify for government contract work.

**Defense Contract Audit Agency (DCAA) Accounting System Requirements**
- URL: https://www.dcaa.mil/Portals/88/Accounting_System.pdf
- DCAA specifies requirements for accounting systems used by government contractors: segregation of direct/indirect costs, timekeeping accuracy, cost accumulation by contract, and indirect rate calculation. Deltek Vantagepoint is the primary platform certified for DCAA compliance in the A/E sector.

**Cost Accounting Standards (CAS) — 48 CFR Chapter 99**
- URL: https://www.acquisition.gov/content/part-9903-contract-coverage
- Mandatory for larger government contracts; defines how contractors must consistently account for and allocate costs. Architecture firms with significant federal work require CAS-compliant accounting systems.

**ASC 606 (FASB) / IFRS 15 — Revenue from Contracts with Customers**
- URL: https://www.fasb.org/page/PageContent?pageId=/standards/asc606.html · https://www.ifrs.org/issued-standards/list-of-standards/ifrs-15-revenue-from-contracts-with-customers/
- US GAAP (ASC 606) and international (IFRS 15) revenue recognition standards applicable to architecture firms recognising project revenue over time using percentage-of-completion or cost-to-cost input methods. Practice management platforms must support these methods for accurate financial reporting and auditor compliance.

**Earned Value Management (ANSI/EIA-748)**
- URL: https://www.pmi.org/learning/library/earned-value-management-evms-standard-practice-7113
- Standard for measuring project performance and progress; increasingly required by sophisticated clients and government contracts. Earned value compares budgeted cost of work scheduled (BCWS) against actual cost of work performed (ACWP). Architecture firm management tools that support EVM provide a significant competitive advantage for enterprise and government-facing firms.

---

## Similar Products — Developer Documentation & APIs

### Deltek Vantagepoint

- **Description:** Enterprise ERP for architecture and engineering firms; covers project management, accounting, CRM, and government contract compliance for 12,000+ A/E firms.
- **API Documentation:** https://vantagepointapi.deltek.com/ (version 2026.2)
- **API Reference:** https://help.deltek.com/Product/Vantagepoint/7.2/dps_api_and_web_services.html
- **Developer Resources:** https://www.deltek.com/en/learn/developer-resources
- **SDKs/Libraries:** REST/JSON; code samples via Deltek learning portal
- **Developer Guide:** https://learning.deltek.com/category/vantagepoint_api_guides
- **Standards:** REST/JSON, OpenAPI
- **Authentication:** OAuth 2.0 / service accounts

---

### BQE CORE

- **Description:** All-in-one firm management platform for architecture, engineering, and professional services; covers time tracking, billing, project management, and accounting.
- **API Documentation:** https://api-explorer.bqecore.com/
- **Developer Portal:** https://api-developer.bqecore.com/
- **SDKs/Libraries:** GitHub samples at https://github.com/BQEDeveloper; PHP, .NET examples available
- **Developer Guide:** https://api-explorer.bqecore.com/docs/getting-started
- **Standards:** REST/JSON, OpenAPI; API version 2026.03.1.0
- **Authentication:** OAuth 2.0

---

### Productive

- **Description:** Professional services automation platform for agencies and consultancies; strong resource planning, time tracking, budgeting, and revenue forecasting.
- **API Documentation:** https://developer.productive.io/reference
- **Developer Portal:** https://developer.productive.io/
- **SDKs/Libraries:** No official SDK; well-documented REST API; community integrations on Pipedream
- **Developer Guide:** https://developer.productive.io/integrations.html
- **Standards:** JSON:API specification (jsonapi.org); REST/JSON; API versioned at /api/v2
- **Authentication:** API token via X-Auth-Token header

---

### Scoro

- **Description:** Professional services automation software for consultancies, agencies, and engineering firms; quote-to-cash lifecycle with portfolio dashboards and AI-powered insights.
- **API Documentation:** https://support.scoro.com/hc/en-us/articles/12805005214477
- **SDKs/Libraries:** Community Go library (github.com/packform/go-scoro); Pipedream integration
- **Developer Guide:** REST API v2 base URL: https://{subdomain}.scoro.com/api/v2
- **Standards:** REST/JSON; MCP server support (2026)
- **Authentication:** API key in request body or user_token

---

### Accelo

- **Description:** Service operations platform for professional services firms; strong client portal, retainer management, and quote-to-cash workflows.
- **API Documentation:** https://api.accelo.com/docs/
- **SDKs/Libraries:** GitHub documentation repo at https://github.com/Accelo/docs; community integrations via Pipedream
- **Developer Guide:** https://www.accelo.com/resources/apis/
- **Standards:** REST/JSON, XML, YAML; supports GET, POST, PUT, DELETE
- **Authentication:** OAuth 2.0 (user grants and service accounts)

---

### QuickBooks Online (Intuit)

- **Description:** Cloud accounting platform integrated by Monograph, BQE CORE, Factor A/E, and most architecture firm management tools for general ledger, AP/AR, and payroll.
- **API Documentation:** https://developer.intuit.com/app/developer/qbo/docs/develop
- **SDKs/Libraries:** PHP SDK (github.com/intuit/QuickBooks-V3-PHP-SDK); Java, Python community SDKs
- **Developer Guide:** https://developer.intuit.com/app/developer/qbo/docs/learn/explore-the-quickbooks-online-api
- **Standards:** REST/JSON; OpenAPI Swagger docs available; 500 req/min per company rate limit
- **Authentication:** OAuth 2.0 with PKCE; webhooks for real-time event notifications

---

### Xero

- **Description:** Cloud accounting platform with a developer-first API; popular integration target for mid-market architecture firm management tools including Productive and Scoro.
- **API Documentation:** https://developer.xero.com/documentation/api/accounting/overview
- **Developer Platform:** https://developer.xero.com/
- **SDKs/Libraries:** Official Python SDK (github.com/XeroAPI/xero-python); .NET SDK (github.com/XeroAPI/Xero-NetStandard); Node.js and Ruby SDKs
- **Developer Guide:** https://developer.xero.com/documentation/guides/oauth2/overview/
- **Standards:** REST/JSON; 6 APIs sharing OAuth 2.0; granular scopes (10 scopes as of March 2026, replacing 2 broad scopes); 60 req/min per tenant; 5,000 req/day per tenant
- **Authentication:** OAuth 2.0 with PKCE; granular scope model

---

### Autodesk Construction Cloud (ACC) / BIM 360

- **Description:** The leading AEC (architecture, engineering, construction) cloud platform; manages design files, BIM models, RFIs, submittals, and construction cost management. Architecture firm management tools seeking BIM integration must interface with ACC/APS APIs.
- **API Documentation:** https://aps.autodesk.com/en/docs/bim360/v1
- **Developer Platform:** https://aps.autodesk.com/ (Autodesk Platform Services)
- **SDKs/Libraries:** REST/JavaScript SDK; BIM 360 API GitHub: https://bim360api.github.io/
- **Developer Guide:** https://aps.autodesk.com/developer/overview/bim-360-api
- **Standards:** REST/JSON; OAuth 2.0 (APS authentication); forward-compatible with ACC project structure
- **Authentication:** OAuth 2.0 (3-legged for user context; 2-legged for server-to-server)

---

## Notes

**Emerging standards gap — practice management data portability**: There is no industry standard for exporting or migrating project data between architecture firm management platforms (unlike accounting data, which QuickBooks IIF/QBO formats partially address). Firms are effectively locked into their chosen platform. An open data model specification for A/E project records (similar to IFC for BIM) would be a meaningful community contribution from an open-source project in this space.

**AIA digital forms API**: AIA Contract Documents are available in digital form via the AIA Contract Documents platform but do not expose a public API for programmatic form generation. Software vendors must either reproduce the G702/G703 structure under licence or provide equivalent outputs without using the actual AIA-branded template.

**Model Context Protocol (MCP)**: Scoro became the first architecture firm management platform to ship an MCP server in early 2026, enabling AI agents to query project and resource data through the standardised MCP interface. This represents a forward-looking integration point for AI-native tools in this space. See https://modelcontextprotocol.io/ for specification.

**DCAA compliance tooling**: There is no open-source or low-cost implementation of DCAA-compliant indirect cost accounting for A/E firms. This represents both a technical complexity and a market gap; firms with federal contracts are effectively forced to use Deltek or expensive alternatives.
