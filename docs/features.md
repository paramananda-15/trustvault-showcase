# TrustVault: Feature Specifications

> This document describes what each feature does and why it exists. No implementation details are included.

---

## Core Security Features

### 1. Three-Lane Intelligent Routing
**What:** Every prompt submitted through TrustVault is automatically classified into one of three security lanes — Green (safe), Yellow (PII detected), or Red (secrets/IP detected) — and handled accordingly.

**Why:** A binary block/allow model is too blunt for enterprise use. Security teams need granular control that balances protection with productivity. The three-lane model allows clean traffic to flow freely while applying proportional controls to sensitive data.

---

### 2. Real-Time PII Pseudonymization
**What:** When a Yellow Lane prompt is detected, all identified sensitive entities are replaced with structured placeholder tokens (e.g., `<PERSON_1>`, `<EMAIL_ADDRESS_2>`) before the message is forwarded to any cloud AI provider. These tokens are mapped and stored for the duration of the conversation.

**Why:** Prevents Personally Identifiable Information from ever leaving the corporate network boundary. Satisfies GDPR, CCPA, and HIPAA data minimization requirements for AI interactions.

---

### 3. Real-Time Response Token Restoration
**What:** As the AI provider streams its response back, the gateway intercepts the byte stream and replaces placeholder tokens with the original values before the text reaches the user's screen.

**Why:** Delivers a completely seamless user experience. Users read a natural, coherent response with real names and details. The underlying security transformation is entirely invisible to them.

---

### 4. Proprietary Code Detection (Red Lane)
**What:** Code segments embedded in user prompts are evaluated by an ML classifier that scores their confidentiality. Segments that score above a defined threshold are routed to the Red Lane.

**Why:** Regex cannot distinguish between publicly available boilerplate code and a company's proprietary trading algorithm. Semantic code analysis provides the accuracy required for protecting intellectual property.

---

### 5. Secret Pattern Detection (Red Lane Short-Circuit)
**What:** Before any ML inference runs, a fast regex scanner checks for known secret patterns — API keys (OpenAI, AWS), database connection strings, JWT tokens, and generic password patterns.

**Why:** These patterns are deterministic and require no ML to detect reliably. Running the regex check first provides instantaneous protection for the most critical threat category with zero latency cost.

---

### 6. Local AI Fallback for Red Lane
**What:** When a request is classified as Red Lane and a local Ollama instance is available, the request (with all sensitive entities already masked) is rerouted to the local AI model instead of being blocked outright.

**Why:** Blocking all Red Lane requests would frustrate developers working on legitimate (but internally sensitive) tasks. Routing to a local, air-gapped model ensures they still receive AI assistance while the data never crosses the network boundary.

---

## Administration Features

### 7. Encrypted API Key Vault
**What:** Administrators issue TrustVault API keys to teams or applications. The corresponding upstream provider credentials (e.g., OpenAI keys) are encrypted and stored centrally. Employees never interact with raw provider credentials.

**Why:** Eliminates API key sprawl. Centralized key management means revocation is instant and auditable. If a team member leaves the company, their access can be revoked without rotating the underlying API keys.

---

### 8. Real-Time Security Analytics Dashboard
**What:** The admin dashboard provides live visualization of security events: lane distribution over time, most frequently detected entity types, per-model usage statistics, and per-user activity.

**Why:** Security teams need observability to demonstrate compliance posture, identify high-risk users or departments, and detect anomalous usage patterns that may indicate policy violations.

---

### 9. Conversation History & Audit Log
**What:** Every user turn (prompt and response) is persisted with metadata including: conversation ID, lane classification, detected entity types, confidence score, model used, and user identity.

**Why:** Provides a forensic audit trail for security investigations and regulatory compliance reporting. Enables analytics on trends over time.

---

## Authentication Features

### 10. JWT-Based Dashboard Authentication
**What:** Dashboard users authenticate with email and password. Upon successful login, the system issues a short-lived, cryptographically signed JWT that authorizes subsequent API calls.

**Why:** Stateless authentication scales horizontally without session storage. JWT claims carry user identity and role without requiring a database lookup on every request.

---

### 11. Secure Chat UI Handoff
**What:** When a dashboard user clicks "Launch Chat," the system generates a single-use, time-expiring handoff token. The user is redirected to the Chat UI, which validates this token to establish an authenticated session.

**Why:** Prevents the need for separate credentials for the Chat UI. Single Sign-On experience within the TrustVault ecosystem. Handoff tokens expire after a single use, eliminating replay attack vectors.
