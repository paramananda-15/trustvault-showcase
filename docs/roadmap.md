# TrustVault: Product Roadmap

> This roadmap outlines the engineering priorities for evolving TrustVault from an MVP into a production-grade, commercially deployable enterprise product.

---

## Current Status: MVP — v1.0.0

The current implementation is a fully functional, Docker-deployable MVP with core DLP, routing, authentication, and observability features operational.

---

## Phase 2 — Production Hardening

### P2.1 — gRPC Internal Transport
Replace the REST/HTTP communication channel between the Go Gateway and the Python ML Engine with gRPC using Protocol Buffers.

- **Impact:** ~5x throughput increase for internal PII analysis calls. Sub-millisecond serialization overhead.
- **Why now:** As the number of concurrent users grows, HTTP overhead becomes the primary bottleneck.

### P2.2 — PostgreSQL Migration
Provide a first-class migration path from SQLite to PostgreSQL for organizations that require higher concurrent write throughput and replication.

- **Impact:** Enables multi-instance gateway deployments sharing a single database.

### P2.3 — Automated Test Suite
Implement comprehensive unit, integration, and end-to-end test suites for the Gateway and ML Engine.

- **Impact:** Prevents regression on security-critical logic during future development.

---

## Phase 3 — Enterprise Features

### P3.1 — SSO / SAML Integration
Integrate with enterprise identity providers: Okta, Microsoft Entra ID (Azure AD), and Google Workspace via OAuth 2.0 and SAML 2.0.

- **Impact:** Eliminates local password management. Enables corporate IT departments to manage TrustVault access through their existing IAM tooling.

### P3.2 — Token Economics & Cost Tracking
Intercept and count input/output tokens per request at the gateway level. Map against model-specific pricing to provide real-time cost estimation and per-department billing breakdowns.

- **Impact:** Gives CTOs and CFOs visibility into AI spend and enables chargeback models.

### P3.3 — Custom Rules Engine
Provide a UI-driven interface for security administrators to define custom detection rules: specific keywords, regex patterns, or semantic similarity thresholds that always trigger a specific lane.

- **Impact:** Enables organizations to protect their own proprietary terminology (e.g., internal project codenames, product names) that generic NLP models may not recognize.

---

## Phase 4 — Advanced Security

### P4.1 — Multi-Modal DLP (File & Document Scanning)
Extend the security boundary to handle file uploads. Extract and scan text from PDFs, CSVs, Excel spreadsheets, and images (via OCR) before allowing them to be processed by AI providers.

- **Impact:** Protects HR, Legal, and Finance departments whose primary data format is documents, not chat text.

### P4.2 — AI Agent & MCP Protocol Support
Implement native governance for Model Context Protocol (MCP) and Agent-to-Agent (A2A) protocol traffic. Provide tool-level access controls for autonomous AI agents.

- **Impact:** Extends TrustVault's security boundary to the fast-growing category of autonomous AI agents, not just human chat interactions.

### P4.3 — Anomaly Detection & Behavioral Baselines
Build per-user behavioral baselines (typical query volume, entity types, models used) and alert security teams when deviations are detected.

- **Impact:** Identifies insider threats and compromised accounts through behavioral analytics rather than just content scanning.

---

## Phase 5 — Scale & Distribution

### P5.1 — Kubernetes Helm Chart
Package the entire TrustVault stack as a production-ready Helm chart for Kubernetes deployments.

### P5.2 — Multi-Region Support
Architecture for active-active gateway deployments across geographic regions with a shared, replicated database.

### P5.3 — Enterprise On-Premise Offering
A fully air-gapped, on-premise deployment option for industries with strict data residency requirements (government, healthcare, defense).
