# IP Management Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source platform for managing patent and trademark portfolios — covering docketing, multi-jurisdictional deadlines, annuity decisions, and preliminary freedom-to-operate analysis.

The IP Management Platform is a candidate open-source alternative to the commercial IPMS oligopoly (Anaqua, Clarivate/CPA Global, Questel, Dennemeyer). It targets corporate IP teams, prosecution boutiques, university technology transfer offices, and startups who currently choose between $100K+/year enterprise suites and ad-hoc spreadsheets. Built on open patent data (USPTO PatentsView, EPO OPS, WIPO ST.96), it brings AI-assisted decision support to workflows historically reserved for organisations that can afford enterprise licensing.

---

## Why IP Management Platform?

- **No mature open-source IPMS exists.** The entire market is served by commercial vendors charging $95/user/month at the low end and $500K+/year at the high end. Universities, startups, and developing-country IP offices have no viable free option.
- **Enterprise tools are priced out of reach for most teams.** Anaqua AQX and CPA Global typically run $100K–$500K+/year with long implementation cycles, while mid-market platforms (Questel, Dennemeyer DIAMS iQ) still demand $30K–$150K/year.
- **Preliminary FTO analysis is gated by attorney fees.** Freedom-to-operate searches cost $5,000–$50,000 per product, blocking startups and university spinouts from basic IP clearance.
- **The data layer is already open; the application layer is missing.** USPTO PatentsView, EPO OPS, and WIPO ST.96 are freely accessible, but no actively maintained OSS application stitches them into a usable IPMS workflow.
- **Incumbents suffer post-M&A fragmentation.** Clarivate's $6.8B acquisition of CPA Global and absorption of IPfolio created integration friction and uncertain product roadmaps that an OSS alternative can sidestep.

---

## Key Features

### Docketing & Deadline Management

- Structured records for patents and trademarks with filing dates, jurisdiction, status, and associated deadlines
- Automatic calculation of US prosecution deadlines, Paris Convention 12-month priority windows, and PCT national phase entry deadlines (30/31 months) across configurable jurisdiction lists
- Deadline cascade computation from a PCT filing date across 150+ jurisdictions with rule-specific variations and conflict detection
- Office action response tracking, RCE/appeal workflow, and foreign filing deadline management
- Madrid Protocol trademark renewal tracking and NICE classification management

### Annuity & Renewal Intelligence

- Fee schedules by jurisdiction with payment status tracking and configurable reminder alerts
- AI-assisted annuity decision support that displays citation count, forward citation trends, and family coverage alongside each renewal decision
- Pay/abandon recommendations with confidence scores incorporating citation strength, business alignment, licensing potential, and remaining portfolio coverage
- Batch review workflow for scheduled renewal decisions

### Patent Office Integration & Document Processing

- USPTO and EPO OPS API integration for automated status updates and correspondence imports
- WIPO ST.96 XML support for structured exchange with patent offices
- AI-assisted docketing from uploaded patent office correspondence: proposed docketing entries and deadline calculations surfaced for paralegal review
- Prosecution file history and correspondence storage linked to each docketed case

### Portfolio Analytics & FTO

- Filterable inventory by status, jurisdiction, technology area, owner, and cost-to-date
- Prior art search via USPTO PatentsView and EPO OPS, embedded in the docketing interface
- Preliminary FTO analysis: AI maps product features to claim language across retrieved patents and produces a colour-coded risk landscape
- Portfolio optimisation scoring on business alignment, citation strength, and cost-per-protected-revenue to surface candidates for abandonment or licensing

### Counsel, Inventors & Licensing

- Outside counsel assignment, work authorisation, and invoice logging for cost tracking
- Invention disclosure tracking from creation through prosecution to licensing, with inventor compensation and royalty records
- Licensing agreement records with royalty payment schedules and field-of-use restrictions linked to portfolio assets

---

## AI-Native Advantage

AI is applied to the most expensive and error-prone IP workflows. Annuity pay/abandon decisions — historically reliant on costly counsel judgement — are scored against citation, licensing, and competitive-coverage signals. Office action correspondence is read automatically and proposed as docketing entries, replacing manual paralegal data entry of the kind Anaqua's Document Auto-Processing pioneered commercially. Preliminary FTO analysis maps product features to claim language across open patent databases, democratising a capability currently reserved for organisations with $5K–$50K per-search budgets. Portfolio scoring continuously identifies the 15–25% of large portfolios that deliver minimal business value.

---

## Tech Stack & Deployment

The platform is designed for self-hosted and cloud deployment, with a plugin architecture so paid annuity services and patent office connectivity can be attached when needed. It builds on open standards: WIPO ST.96 XML for patent office data exchange, EPO OPS for European patent retrieval, USPTO PatentsView for US data, and WIPO TMview for trademark data. REST APIs and standard authentication patterns are expected to support integration with billing systems, document management tools, and external analytics. Trade-secret-grade encryption, access control, and audit logging are required given the confidential R&D data IPMS platforms hold.

---

## Market Context

The IP Management Software market was approximately $9.045 billion in 2025 and is forecast to reach $17.649 billion by 2031 at an 11.79% CAGR (GlobeNewswire/Research and Markets); the enterprise segment alone is growing at roughly 13.94% CAGR (Market.us). Pricing spans $95–$495/user/month for prosecution boutiques (AppColl), $20K–$150K/year for mid-market analytics and IPMS tools (PatSnap, Questel, Dennemeyer), and $100K–$500K+/year for Fortune 500 platforms (Anaqua, CPA Global). Primary buyers include corporate IP counsel managing 1,000–50,000+ patent portfolios, mid-size technology companies, prosecution law firms, and budget-constrained university technology transfer offices.

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

---

## Licence

Licence to be determined. See [discussion](#) for context.
