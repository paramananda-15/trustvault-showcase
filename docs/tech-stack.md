# TrustVault: Technology Stack

> This document justifies every technology choice in TrustVault from an engineering and business perspective.

---

## Backend Services

### Go (Golang) — The API Gateway
- **Version:** 1.24
- **Why Go over Node.js or Python:** Go's concurrency model (goroutines + channels) is uniquely suited to the TrustVault proxy's core job: keeping thousands of long-lived HTTP/SSE streaming connections open simultaneously. Each streaming AI response can take 10–30 seconds. Node.js's event loop and Python's GIL both struggle under this profile. A Go goroutine consumes ~8KB of memory vs ~1MB per OS thread in traditional models.
- **Why Go over Rust:** Go provides comparable performance with vastly faster development iteration and a larger standard library for HTTP/networking use cases.

### Python 3.12 + FastAPI — The ML Engine
- **Why Python:** Python is the only viable language for production ML inference. PyTorch, Hugging Face Transformers, SpaCy, and Microsoft Presidio all have Python as their first-class citizen. Wrapping these in any other language adds complexity without benefit.
- **Why FastAPI over Flask/Django:** FastAPI is ASGI-native (async I/O), making it the fastest Python framework for I/O-bound ML serving. It also provides automatic schema validation and OpenAPI documentation generation at zero extra cost.

---

## ML & NLP Components

### Microsoft Presidio — PII Detection Framework
- **Why Presidio over custom regex:** Presidio is a battle-tested, open-source DLP framework from Microsoft used in Azure AI environments. It provides a plugin architecture for custom recognizers and handles dozens of global PII types out of the box.

### SpaCy `en_core_web_lg` — Named Entity Recognition
- **Why SpaCy over BERT-based NER:** SpaCy's statistical NER model provides an excellent speed-accuracy trade-off for real-time, synchronous analysis. BERT-based models are 10–50x slower for NER tasks. For a proxy that introduces latency on every request, SpaCy's sub-100ms inference is critical.
- **Pipeline Optimization:** Non-essential pipeline components (`tagger`, `parser`, `lemmatizer`) are disabled at load time to reduce inference time and memory footprint.

### CodeBERT / PyTorch — Code Confidentiality Classifier
- **Why a semantic classifier over regex:** Regex can detect known secret patterns (API key formats) but cannot reason about whether a block of code represents a proprietary algorithm vs. boilerplate. CodeBERT was pre-trained on millions of code repositories, giving it semantic understanding of code structure and novelty.

### Hugging Face Transformers — Multilingual NER
- **Why:** The base SpaCy model is English-only. For multinational enterprises, employees communicate in local languages. A multilingual transformer model extends PII detection coverage beyond English text.

---

## Frontend

### Next.js 14 (React) — Admin Dashboard
- **Why Next.js:** The dashboard is a data-heavy admin interface requiring server-side rendering for fast initial loads, API route generation, and a rich ecosystem of UI libraries (Recharts for analytics, Tailwind CSS for styling).
- **Why not a pure SPA (Vite/CRA):** Next.js's file-system-based routing and API route handlers eliminate the need for a dedicated Express/FastAPI backend for the dashboard, simplifying the service count.

### SvelteKit — Chat User Interface
- **Why SvelteKit:** The chat interface is a heavily customized fork of Open WebUI, which is built on SvelteKit. SvelteKit's compiler-based approach produces extremely lean JavaScript bundles, which is critical for a real-time streaming chat interface where every millisecond of rendering latency is perceptible to the user.

---

## Infrastructure

### Docker + Docker Compose
- **Why Docker:** Guarantees that the system runs identically across every developer's machine, regardless of their operating system or installed software. This is the single most important factor for team onboarding.
- **Multi-Stage Builds:** Each service uses a multi-stage Dockerfile to produce minimal final images (e.g., the Go proxy compiles in a `golang:alpine` builder and runs in a bare `alpine` final image, keeping the image under 20MB).

### GitHub Actions + GHCR (GitHub Container Registry)
- **Why GHCR over Docker Hub:** GHCR is natively integrated with GitHub's permission model. Images are automatically scoped to the repository's access controls without additional configuration.
- **Matrix Strategy:** The CI pipeline builds all four service images in parallel using a matrix strategy, cutting total build time significantly compared to sequential builds.

### SQLite (via CGO)
- **Why SQLite over PostgreSQL for MVP:** SQLite provides a fully ACID-compliant relational database as a single, portable file. It requires zero infrastructure (no database server process, no connection management). This allows the entire TrustVault stack to run with `docker compose up` — no additional services needed.
