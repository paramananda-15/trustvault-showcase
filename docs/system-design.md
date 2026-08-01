# TrustVault: System Design

> This document covers design decisions, trade-offs, and system-level thinking behind TrustVault. No implementation details are included.

---

## Core Design Principles

1. **Security must be invisible** — Users must not experience any friction from the security layer.
2. **Fail-open, log always** — The system prioritizes availability. If the analysis engine is unavailable, traffic flows through and is flagged for audit.
3. **Separation of concerns at the runtime level** — Network I/O and ML compute are isolated in separate processes and containers.
4. **OpenAI protocol compatibility** — Any existing tool (CLI, IDE plugin, agent) works without modification.

---

## Component Design

### The Three-Lane Routing System

The three-lane model was chosen over a binary block/allow model because enterprise security is not binary. A query containing a colleague's first name is fundamentally different from one containing a hardcoded database password. A binary model would either be too restrictive (blocking everything with a name) or too permissive (allowing everything that isn't a secret).

| Lane | Trigger Conditions | Gateway Behavior |
|------|-------------------|-----------------|
| 🟢 Green | No recognized sensitive entities | Pass-through to cloud AI |
| 🟡 Yellow | PII entities detected (PERSON, EMAIL, PHONE, LOCATION, ORG) | Pseudonymize → Forward masked → Unmask response |
| 🔴 Red | Secrets (API keys, DB URIs) or high-confidence proprietary code | Block or redirect to local offline AI |

### Pseudonymization vs. Redaction

**Redaction** would replace `"My boss Steve"` with `"My boss [REDACTED]"`. This breaks the AI's ability to provide contextually accurate help.

**Pseudonymization** replaces it with `"My boss <PERSON_1>"`. The AI can still reason about this person — count references to them, understand their role, generate advice — while having no knowledge of their real identity. This is the approach defined in GDPR Article 4(5) as a data protection technique.

### Stateless Analysis Engine

The ML engine is intentionally designed as a stateless, ephemeral service. Each analysis request is completely independent. Conversation context (the token-to-entity mapping, e.g., `<PERSON_1>` = "Steve") is maintained exclusively by the gateway service in memory, keyed by Conversation ID.

**Why:** Statelessness allows the ML engine to be horizontally scaled — multiple instances can serve analysis requests independently without shared state coordination.

### Deduplication Log Window

Modern AI chat interfaces generate multiple background API calls per user turn (e.g., title generation, conversation tagging, follow-up suggestion). Without deduplication, a single user message generates 4–6 audit log entries.

The gateway uses a short time-window deduplication strategy: if a log entry for the same user identity exists within a defined recency window, the new entry is either merged (upgrading the severity lane if higher) or discarded.

**Why:** Clean audit logs are critical for security teams. Signal-to-noise ratio in DLP logs directly impacts how quickly analysts can identify genuine threats.

---

## Database Design Philosophy

### Single-File Embedded Store (MVP Choice)

SQLite was chosen deliberately for the MVP phase. The primary requirement was zero additional infrastructure: a developer should be able to clone the repository and run the entire stack with a single command.

**Tables:**
- `users` — Corporate identity store (name, email, hashed password, role)
- `api_keys` — Tenant API keys with encrypted upstream credentials
- `conversations` — Per-turn chat history with original and masked content
- `logs` — Security audit log with lane, entity type, confidence, and user attribution

**Future Migration Path:** The database access layer is deliberately abstracted behind an interface. Migrating from SQLite to PostgreSQL is a configuration change, not a refactor.

---

## Security Model

### Defense in Depth
TrustVault implements multiple independent security controls:

1. **Perimeter:** JWT authentication on every request
2. **Analysis:** ML-based PII and code classification
3. **Transport:** All upstream traffic is over HTTPS (TLS 1.3)
4. **Storage:** Upstream API keys encrypted at rest (AES-GCM)
5. **Audit:** All requests logged with full attribution

### Threat Model

| Threat | Mitigation |
|--------|-----------|
| Employee pastes customer PII | Yellow Lane pseudonymization |
| Developer pastes API key | Regex short-circuit → Red Lane |
| Proprietary algorithm leakage | ML code classifier → Red Lane |
| Stolen database file | AES-GCM encrypted credentials |
| Session hijacking | Short-lived JWTs + one-time handoff tokens |

---

## Trade-offs Acknowledged

| Decision | Benefit | Cost |
|----------|---------|------|
| Fail-open on ML engine error | High availability | Security gap during engine downtime |
| HTTP internal transport (vs gRPC) | Simple debugging | Higher latency at scale |
| SQLite (vs PostgreSQL) | Zero-dependency deployment | Limited concurrent write throughput |
| Pseudonymization (vs redaction) | Seamless UX | Requires stateful token vault in memory |
| ML NER (vs strict dictionaries) | Catches unknown entities | Occasional false negatives |
