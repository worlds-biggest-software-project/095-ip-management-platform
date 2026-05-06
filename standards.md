# Standards & API Reference

> Project: IP Management Platform · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

**ISO 56005:2020 — Innovation Management: Tools and Methods for Intellectual Property Management — Guidance**
- URL: https://www.iso.org/standard/72761.html
- Defines guidelines for organisations to create an IP strategy, establish systematic IP management within innovation processes, and apply consistent tools and methods. Annexes cover IP generation, acquisition, maintenance, search, evaluation, and risk management. An IP management platform implementing workflows aligned to ISO 56005 can be marketed to enterprises seeking certified innovation management practice.

**ISO 27001 / ISO 27002 Control 5.32 — Intellectual Property Rights**
- URL: https://www.isms.online/iso-27002/control-5-32-intellectual-property-rights/
- ISO 27002 Control 5.32 requires organisations to implement procedures to protect their own IP and respect third-party IP rights within an information security management system. An IP management platform storing confidential R&D data, trade secrets, and patent prosecution information must comply with these controls, including encryption, access control, and audit logging.

**ISO 27001 — Information Security Management System**
- URL: https://www.iso.org/standard/27001
- General ISMS standard governing encryption, access control, incident response, and audit trails. Mandatory for any SaaS platform storing confidential legal and IP data. An open-source IPMS should document ISO 27001-aligned controls for enterprise buyers to self-certify or seek certification.

---

### W3C & IETF Standards

**WIPO ST.96 — XML Standard for Industrial Property Information and Documentation**
- URL: https://www.wipo.int/standards/en/st96/
- XML Schema (Annex III): https://www.wipo.int/standards/XMLSchema/AFPatent/
- The primary XML standard for structured data exchange between patent offices, IPMS platforms, and IP service providers. Covers patents, trademarks, industrial designs, and geographical indications. ST.96 schemas are available at component level (modular) and document level (flattened). Essential for implementing automated docketing from USPTO, EPO, JPO, CNIPA, and other patent office data feeds. All major IPMS vendors integrate ST.96 for automated correspondence ingestion and status updates.

**IETF RFC 7519 — JSON Web Token (JWT)**
- URL: https://www.rfc-editor.org/rfc/rfc7519
- Standard data format for authentication tokens used in OAuth 2.0 / OpenID Connect flows. All patent office APIs and enterprise IPMS integrations require JWT-based authentication; implementing RFC 7519 correctly is a foundational security requirement for any IPMS API layer.

**IETF RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://www.rfc-editor.org/rfc/rfc6749
- The standard for delegated authorisation enabling users to grant third-party applications access to patent office APIs and connected services without sharing credentials. Required for USPTO, EPO OPS, and EUIPO API integrations.

**IETF RFC 6750 — OAuth 2.0 Bearer Token Usage**
- URL: https://www.rfc-editor.org/rfc/rfc6750
- Defines how bearer tokens are transmitted in OAuth 2.0 flows; required alongside RFC 6749 for all patent office and third-party API authentication.

**W3C DCAT — Data Catalog Vocabulary (Version 3)**
- URL: https://www.w3.org/TR/vocab-dcat-3/
- Standard metadata vocabulary for describing datasets and data services. Relevant for describing the open patent datasets (USPTO PatentsView, EPO OPS, WIPO PATENTSCOPE) that underpin an open-source IPMS. Enables interoperability with government open data portals and research data repositories.

---

### IP Treaty & Regulatory Frameworks

**Patent Cooperation Treaty (PCT) — WIPO**
- URL: https://www.wipo.int/en/web/pct-system
- Time Limits Table: https://www.wipo.int/en/web/pct-system/texts/time_limits
- Governs international patent applications filed under PCT. National phase entry deadlines are 30 months (most jurisdictions), 31 months (EPO and some others), or 34 months (Bosnia and Herzegovina) from the priority date. PCT national phase cascade computation — automatically deriving all downstream national entry deadlines for a given PCT filing date — is one of the highest-value automation features for any IPMS. Implementors must reference WIPO's official time limits table as it is updated periodically.

**Paris Convention for the Protection of Industrial Property**
- URL: https://www.wipo.int/en/web/paris-convention
- Establishes the 12-month priority window for patents (6 months for designs and trademarks) from the date of first filing. Priority claim management is table-stakes for any patent docketing system. The Convention text and member state information are freely available from WIPO.

**Madrid Protocol — International Trademark Registration**
- URL: https://www.wipo.int/en/web/madrid-system
- eMadrid Portal: https://madrid.wipo.int/manage
- Governs international trademark registration through the Madrid System. IPMS platforms must manage Madrid filing, renewal cycles (every 10 years), and national designation deadlines. WIPO's eMadrid platform provides a web interface for managing Madrid registrations; a dedicated public API for the Madrid Monitor database is not officially supported as of 2026 — integration typically relies on WIPO ST.96 data exchange or screen-scraping workarounds.

**TRIPS Agreement (Trade-Related Aspects of Intellectual Property Rights)**
- URL: https://www.wto.org/english/tratop_e/trips_e/trips_e.htm
- WTO agreement setting minimum IP protection standards across 164 member countries. Shapes jurisdiction coverage requirements for any global IPMS: the patent term (minimum 20 years from filing date), enforcement rights, and compulsory licensing exceptions all vary by jurisdiction within TRIPS' floor.

**EU Trade Secrets Directive (2016/943)**
- URL: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32016L0943
- EU Directive harmonising protection of undisclosed know-how and trade secrets across member states. An IPMS storing confidential R&D data, invention disclosures prior to filing, and competitive intelligence must implement controls meeting the Directive's "reasonable steps" threshold for trade secret protection.

**US Defend Trade Secrets Act (DTSA) 2016**
- URL: https://www.congress.gov/bill/114th-congress/senate-bill/1890
- US federal statute providing civil remedies for trade secret misappropriation. Complements GDPR in defining the security and access control requirements for confidential invention disclosure and prosecution data stored in an IPMS.

**GDPR — General Data Protection Regulation (EU) 2016/679**
- URL: https://gdpr.eu/
- Governs handling of personal data (inventor records, attorney contacts, licensing counterparty information) stored within an IPMS. Requires data minimisation, retention limits, right-of-erasure workflows, and Data Protection Impact Assessments (DPIAs) for processing at scale. EU-deployed IPMS platforms must appoint a Data Protection Officer if processing personal data at large scale.

---

### Data Model & API Specifications

**OpenAPI Specification (OAS) 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0.html
- GitHub: https://github.com/OAI/OpenAPI-Specification
- The industry-standard specification for describing RESTful APIs. An open-source IPMS should publish its API surface as an OAS 3.1 document to enable automated documentation generation, client SDK generation, and integration testing. WIPO's IP API Catalog (apicatalog.wipo.int) automatically indexes APIs described using OAS files — publishing an OAS-compliant API makes an OSS IPMS discoverable in the WIPO catalog.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/specification
- Foundation for the data model definitions within OAS 3.1. Useful for defining the patent, trademark, application, deadline, and portfolio data models that an IPMS API exposes.

---

### Security & Authentication Standards

**OpenID Connect 1.0 (OIDC)**
- URL: https://openid.net/developers/how-connect-works/
- Authentication layer built on top of OAuth 2.0. The standard for user identity in modern SaaS and legal tech platforms. Required for enterprise SSO integration (Microsoft Entra ID, Okta, Google Workspace). An IPMS must implement OIDC to be acceptable to corporate IP counsel teams operating in enterprise identity environments.

**NIST Cybersecurity Framework (CSF) 2.0**
- URL: https://www.nist.gov/cyberframework
- Six-function framework (Govern, Identify, Protect, Detect, Respond, Recover) that enterprise buyers use to assess supplier security posture. Of particular relevance: audit log retention (SP 800-92), encryption of data at rest and in transit (FIPS-approved ciphers), and access control policies. An IPMS handling patent prosecution data should document CSF alignment to support enterprise procurement.

**NIST SP 800-92 — Guide to Computer Security Log Management**
- URL: https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-92.pdf
- Defines requirements for audit log collection, retention, analysis, and protection. Patent prosecution data is legally sensitive; loss or tampering of deadline records can result in irreversible loss of patent rights. Audit logging per SP 800-92 is a baseline requirement for any production IPMS.

---

### MCP Server Specifications

**Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/
- PatSnap MCP integration: https://open.patsnap.com/
- An emerging standard for exposing data and tools to LLM agents. PatSnap has implemented MCP support to connect its domain-specific AI patent analytics agents to external LLM platforms (including Claude). An open-source IPMS with an MCP server layer could expose patent portfolio data, deadline queues, and annuity decision workflows to AI agents — enabling autonomous portfolio monitoring and AI-assisted prosecution support.

---

## Similar Products — Developer Documentation & APIs

### USPTO Open Data Portal

- **Description:** The US Patent and Trademark Office's primary developer platform, replacing the legacy Open Data Portal Beta (shutdown May 29, 2026). Provides free, open API access to patent prosecution data, PTAB decisions, and trademark status.
- **API Documentation:** https://developer.uspto.gov/api-catalog
- **Data Portal:** https://data.uspto.gov/
- **Key APIs:**
  - Patent File Wrapper Search API: https://data.uspto.gov/apis/patent-file-wrapper/search
  - Patent File Wrapper Documents API: https://data.uspto.gov/apis/patent-file-wrapper/documents
  - TSDR Data API (trademarks): https://developer.uspto.gov/api-catalog/tsdr-data-api
  - PTAB API v3: https://developer.uspto.gov/api-catalog/ptab-api-v3-data-odp
  - Bulk Data Directory: https://data.uspto.gov/bulkdata/datasets
- **Developer Guide:** https://data.uspto.gov/apis/getting-started
- **Standards:** REST/JSON, OpenAPI, Swagger documentation for each endpoint
- **Authentication:** API key (required for bulk TSDR downloads; free registration at USPTO.gov)
- **Notes:** All USPTO patent data is US government work — freely usable without restriction for commercial or OSS purposes. The legacy PatentsView API was discontinued May 1, 2025; the PatentSearch API on ODP is the current replacement. PatentsView data tables remain available as bulk CSV downloads.

---

### EPO Open Patent Services (OPS) v3.2

- **Description:** The European Patent Office's free RESTful API for accessing patent bibliographic data, full-text documents, legal status, and family information across the EPO's global patent database (90+ million documents). The most widely integrated patent API by IPMS vendors and analytics tools.
- **API Documentation:** https://ops.epo.org/ (requires developer registration)
- **Developer Registration:** https://developers.epo.org/user/register
- **Python SDK:** https://github.com/ip-tools/python-epo-ops-client (PyPI: `python-epo-ops-client`)
- **Go SDK:** https://pkg.go.dev/github.com/patent-dev/epo-ops
- **PatZilla integration reference:** https://docs.ip-tools.org/patzilla/datasource/epo-ops.html
- **Standards:** REST/JSON, OAuth 2.0 for authentication
- **Authentication:** OAuth 2.0 with Client ID and Client Secret (obtained via EPO Developer Portal)
- **Rate Limits / Licence:** Free tier up to 4 GB/month; commercial licence required for production SaaS applications exceeding free tier. Terms should be reviewed before building a commercial product on EPO OPS.
- **Notes:** OPS v3.2 is the current version as of 2026. An unofficial wrapper library `epo-ops` is also available on npm/GitHub for JavaScript/Node.js usage.

---

### WIPO PATENTSCOPE API

- **Description:** WIPO's free patent database covering 60+ patent collections with emphasis on PCT international applications. Provides the most complete and up-to-date source of PCT filing data, including the International Application Status Report (IASR). Unique cross-lingual semantic search capability.
- **API Documentation:** https://www.wipo.int/en/web/patentscope/data/index
- **API Catalog:** https://apicatalog.wipo.int/
- **API Catalog User Guide:** https://www.wipo.int/en/web/standards/ip-api-catalog/user-guide
- **Standards:** SOAP-based Java API for PATENTSCOPE; WIPO IP API Catalog indexes OAS-compliant APIs from IP offices globally
- **Authentication:** Registration required for programmatic access
- **Notes:** WIPO launched its IP API Catalog (apicatalog.wipo.int) in 2024 as a unified index of APIs from IP institutions worldwide covering patents, trademarks, industrial designs, copyright, geographical indications, and plant variety protection. An OSS IPMS should register its own OAS-compliant API with this catalog for discoverability.

---

### EUIPO API Portal

- **Description:** The European Union Intellectual Property Office's developer API platform for EU trademarks (EUTMs) and Community designs (RCDs). Provides search, retrieval, and filing APIs for all IP rights registered with the EUIPO.
- **API Portal:** https://dev.euipo.europa.eu/
- **Key APIs:**
  - Trademark Search: https://dev.euipo.europa.eu/product/trademark-search_100
  - Design Search: retrieve all information on a given Community design
  - Trademark Filing: apply for an EUTM via API
  - Design Filing: apply for an RCD via API
  - TMClass (Goods & Services): explore and validate NICE Classification entries
- **Developer Guide:** Sandbox portal available at dev.euipo.europa.eu for read/write testing with realistic sample data
- **Standards:** REST/JSON; OpenAPI specification for each API product
- **Authentication:** API key / OAuth 2.0 (varies by API product)
- **Notes:** EUIPO launched a new suite of APIs in 2024–2025 covering trademark search, design filing, and portfolio management. The Sandbox environment allows developers to test integration without affecting production data — highly suitable for OSS IPMS development and testing.

---

### PatSnap Open Platform

- **Description:** PatSnap's developer platform providing access to 170+ million patent documents, AI-powered patent analytics, drug data, antigen-antibody data, and corporate IP data. Offers API, MCP server, and UI widget integration options for embedding PatSnap capabilities in custom applications.
- **API Documentation:** https://open.patsnap.com/tutorials/getStart
- **Open Platform:** https://open.patsnap.com/
- **Data & AI APIs:** https://www.patsnap.com/products/data-apis
- **API Terms:** https://www.patsnap.com/api-product-specific-terms/
- **Standards:** REST/JSON; OpenAPI specification files provided; sandbox environments for testing; code samples in Python, JavaScript, Java
- **Authentication:** Client ID and Client Secret (Base64-encoded); token-based with 30-minute token validity
- **MCP Support:** PatSnap exposes domain-specific AI agents to external LLMs via the Model Context Protocol (MCP)
- **Notes:** PatSnap's API provides automated prior art search, semantic patent search, FTO intelligence, and technology landscape data. An OSS IPMS could integrate PatSnap's API for AI-assisted FTO and prior art modules — though this would create a commercial API dependency. The OSS alternative is to build on free USPTO and EPO OPS data.

---

### Questel Gateway API

- **Description:** Questel's API platform enabling programmatic integration with Questel's IP services including patent search and analytics (Orbit Intelligence), filing, translation, and annuity/renewal services (PAVIS Connect). Allows external docketing systems to trigger Questel renewal instructions via API.
- **API Documentation:** https://www.questel.com/patent/questel-gateway-api/
- **Standards:** REST/JSON; SSL/TLS encryption; authentication tokens
- **Authentication:** Token-based with access controls
- **Key Capabilities:** Prior art search, patent filing and translation, fee management, renewal instruction submission, project management integration
- **Notes:** Questel Gateway API is the primary integration path for connecting a third-party IPMS to Questel's annuity renewal service (PAVIS Connect). This is a commercially licenced API — an OSS IPMS should define a plugin interface that allows users to connect their own Questel credentials without bundling Questel API keys in the OSS codebase.

---

### Anaqua AQX (AQX Sync)

- **Description:** Anaqua's enterprise IPMS with an integration layer called AQX Sync — an API-based enterprise content synchronisation solution for connecting AQX to external document management systems, matter management platforms, HR systems, finance systems, and collaboration suites. Anaqua also exposes a broader API suite for custom integration needs.
- **Integration Overview:** https://www.anaqua.com/application-integration-and-migration/aqx-sync/
- **Connectivity Partner (SeeUnity):** https://seeunity.com/anaquas-aqx-integrations/
- **Standards:** REST APIs; custom integration support for 40+ connected systems
- **Authentication:** Enterprise SSO integration via standard OIDC/SAML
- **Notes:** Anaqua's API is not publicly documented — access requires a commercial agreement with Anaqua. The integration layer supports document management systems (iManage, NetDocuments, SharePoint), legal matter management (Aderant, Elite 3E), and HR/finance systems. An OSS IPMS should define a compatible data exchange interface to allow migration from Anaqua — a common enterprise buying scenario.

---

### Dennemeyer DIAMS

- **Description:** Dennemeyer's IP management and annuity renewal platform (DIAMS iQ for enterprise, DIAMS U for SMBs) with API access to Dennemeyer's IP maintenance and renewal services. Enables programmatic submission of renewal instructions to Dennemeyer's global patent annuity payment network.
- **Product Overview:** https://www.dennemeyer.com/ip-software/diams/
- **Standards:** API-based integration; specifics not publicly documented
- **Authentication:** Token-based
- **Notes:** Dennemeyer's API is not publicly documented — access requires a commercial services agreement. An OSS IPMS should design a plugin architecture where annuity service providers (Dennemeyer, CPA Global, Questel PAVIS) can be connected via standardised plugin interfaces, avoiding hard-coded dependency on any single renewal provider.

---

### AppColl Prosecution Manager

- **Description:** Cloud-based patent docketing system for small-to-mid-size IP law firms, with REST API access and USPTO/EPO/TSDR integration for automated status updates.
- **Product Overview:** https://www.appcoll.com/
- **API Forum:** https://forum.appcoll.com/topic/285/api
- **Support Documentation:** https://support.appcoll.com/
- **Standards:** REST API (specifics require AppColl account for access); USPTO Patent Center and TSDR integration for automated imports
- **Authentication:** API key
- **Integration Sources:** USPTO Patent Center, TSDR, Espacenet (EPO public search)
- **Notes:** AppColl's API is not fully publicly documented but exists for custom integrations. The platform ingests USPTO XML data for automated docketing entry generation. An OSS IPMS targeting the same law firm market should offer compatible import/export formats to facilitate AppColl migration.

---

### Wolters Kluwer Developer Portal

- **Description:** Wolters Kluwer's developer platform primarily oriented toward its tax and accounting software (CCH Axcess), though relevant as a reference for how an established legal/compliance software vendor structures its developer ecosystem.
- **Developer Portal:** https://developers.cchaxcess.com/
- **Standards:** REST/JSON; OpenAPI / WADL definitions; code samples in 9+ languages
- **Authentication:** OAuth 2.0 / WK credentials
- **Notes:** Wolters Kluwer's IP Manager (Acumass) does not have a publicly accessible developer API portal as of 2026. Technical integration with Wolters Kluwer's IP products requires a commercial agreement. The CCH Axcess portal referenced here is for their tax/accounting suite and serves as a model for the developer experience an enterprise-grade OSS IPMS should aspire to.

---

## Notes

**Emerging and Evolving Areas**

- The WIPO IP API Catalog (apicatalog.wipo.int) launched in 2024 and is actively expanding. It will increasingly become the authoritative index of which patent offices offer machine-readable APIs. An OSS IPMS should monitor this catalog to identify newly available patent office integrations as they appear.

- The USPTO's migration from the legacy Open Data Portal Beta to the new ODP (data.uspto.gov) completed in early 2026, with the PatentsView platform also migrating to ODP in March 2026. All new USPTO API integrations should target data.uspto.gov rather than legacy endpoints.

- WIPO's Madrid Protocol trademark management relies on eMadrid (madrid.wipo.int) for portfolio management. There is no officially documented public API for programmatic access to the Madrid Monitor database as of 2026. Trademark management in an OSS IPMS should plan to use WIPO ST.96 XML data exchange for Madrid filings and rely on EUIPO's EUTM APIs for European trademark data.

- Model Context Protocol (MCP) integration is an emerging pattern: PatSnap already offers an MCP server for connecting its patent analytics AI to external LLM platforms. An OSS IPMS with an MCP server layer could expose patent portfolio context to AI agents — enabling natural-language patent portfolio queries, deadline summaries, and AI-assisted prosecution decisions via any MCP-compatible LLM client.

- PCT deadline computation must reference WIPO's official time limits table (regularly updated) rather than hardcoding national phase deadlines. The WIPO PCT eGuide (pctlegal.wipo.int) provides country-specific applicant guides including jurisdiction-specific deadline variations that an IPMS deadline engine must incorporate.
