# TrustVault: Known Limitations

> This document honestly describes the current limitations of the TrustVault MVP. Transparency about trade-offs demonstrates engineering maturity.

---

## 1. ML Model False Negatives

**Limitation:** The NLP-based entity recognition relies on a statistical model (SpaCy `en_core_web_lg`). This model may fail to recognize entities that are:
- Very short single-word names that appear in ambiguous grammatical contexts.
- Company names that also function as common words (e.g., "Apple" in a sentence about fruit).
- Obfuscated or intentionally misspelled sensitive data.

**Impact:** A small percentage of sensitive prompts may pass through as Green Lane when they should be Yellow Lane.

**Mitigation:** The Regex secret scanner provides a deterministic backstop for the highest-risk category (API keys, database credentials). Custom rules (Roadmap Phase 3.3) will allow organizations to supplement ML detection with deterministic keyword matching.

---

## 2. HTTP Internal Latency

**Limitation:** The internal communication between the Go Gateway and the Python ML Engine uses standard REST over HTTP. Every prompt incurs the overhead of an HTTP request/response cycle plus JSON serialization.

**Impact:** Adds approximately 100–300ms of latency to every prompt. For real-time streaming chat, this delay is perceptible but generally acceptable in an enterprise context where security is the priority.

**Mitigation:** Roadmap Phase 2.1 (gRPC migration) will replace this with a binary, compressed protocol with sub-10ms overhead.

---

## 3. Single-Writer Database (SQLite)

**Limitation:** SQLite is an embedded, single-writer database. Under concurrent write load (many users simultaneously completing chats), write contention can cause brief queuing delays.

**Impact:** At low-to-medium scale (under ~100 concurrent users), this is imperceptible. At enterprise scale (1,000+ concurrent users), this becomes a throughput bottleneck for audit logging.

**Mitigation:** The database layer is abstracted behind an interface to allow a straightforward migration to PostgreSQL (Roadmap Phase 2.2) for production deployments.

---

## 4. English-Primary NER

**Limitation:** The primary NER pipeline is optimized for English text. While a multilingual transformer model is included for Hindi, coverage for other languages (Mandarin, Spanish, Arabic, etc.) is limited.

**Impact:** Organizations with non-English-speaking employees may experience reduced PII detection accuracy in non-English prompts.

**Mitigation:** Additional language-specific transformer models can be added to the analysis pipeline as organizational needs dictate.

---

## 5. In-Memory Vault (No Persistence Across Restarts)

**Limitation:** The token-to-entity mapping (e.g., `<PERSON_1>` = "Steve") is stored in the gateway's process memory. If the gateway container restarts during an active conversation, the vault is lost.

**Impact:** Active conversations are disrupted by gateway restarts. This is an edge case in normal operation but relevant for rolling deployments.

**Mitigation:** The conversation history (including masked content) is persisted in SQLite. A future enhancement will reconstruct the vault from the database on startup.

---

## 6. No Native File Upload DLP

**Limitation:** File uploads sent directly as binary attachments to AI provider file endpoints (e.g., `/v1/files`) are not currently scanned. Only text embedded in chat message payloads is analyzed.

**Impact:** Users who upload sensitive PDFs, CSVs, or images directly to supported AI platforms may bypass the text-based DLP pipeline.

**Mitigation:** This is addressed directly in Roadmap Phase 4.1 (Multi-Modal DLP).

---

## 7. No Billing or Token Tracking

**Limitation:** The gateway does not currently count input/output tokens or track costs per user or department.

**Impact:** Organizations cannot use TrustVault for AI spend management or chargeback reporting.

**Mitigation:** Roadmap Phase 3.2 (Token Economics) addresses this directly.
