# TrustVault: System Architecture

> This document describes the high-level system architecture of TrustVault. No source code or implementation details are included.

---

## Architectural Overview

TrustVault is composed of four primary services, each running as an isolated Docker container, communicating over a private internal network.

```mermaid
graph LR
    subgraph "Corporate Network Perimeter"
        subgraph "User Interfaces"
            A[Chat UI<br/>:8081]
            B[Admin Dashboard<br/>:3000]
        end

        subgraph "TrustVault Core"
            C[🛡️ API Gateway<br/>:7878]
            D[🧠 ML Analysis Engine<br/>:8000]
            E[🗄️ SQLite Database]
        end
    end

    subgraph "External — Internet"
        F[☁️ OpenAI / Anthropic<br/>Cloud LLMs]
    end

    subgraph "Local — Offline"
        G[🏠 Ollama<br/>Local AI Models]
    end

    A & B -->|All Traffic| C
    C <-->|Text Analysis| D
    C <-->|Persistence| E
    C -->|Safe / Masked Traffic| F
    C -->|Sensitive Traffic Only| G
```

---

## Component Descriptions

### 1. API Gateway (`:7878`)
The central control point. It is a fully OpenAI-compatible REST API endpoint. Any application that can speak to OpenAI can be instantly redirected through TrustVault with a single configuration change.

**Responsibilities:**
- Authenticating all inbound requests (JWT / API Key)
- Orchestrating the analysis pipeline for every prompt
- Enforcing lane-based routing decisions
- Proxying and streaming responses back to clients
- Managing conversation history and the in-memory token vault

### 2. ML Analysis Engine (`:8000`)
A stateless inference service that accepts raw text and returns a classification verdict and anonymized payload.

**Responsibilities:**
- Named Entity Recognition (NER) for PII detection
- Confidentiality scoring for code segments
- Multilingual text analysis
- Producing a structured anonymization mapping (token vault)

### 3. Admin Dashboard (`:3000`)
A Next.js web application providing full observability into the security gateway.

**Responsibilities:**
- Real-time display of security lane events
- API key issuance and management for corporate tenants
- Analytics and trend visualization

### 4. Chat UI (`:8081`)
A customized, production-ready AI chat interface pre-configured to route all traffic through the TrustVault gateway.

---

## Authentication Flow

```mermaid
sequenceDiagram
    participant U as User
    participant DB as Dashboard (:3000)
    participant GW as Gateway (:7878)
    participant CU as Chat UI (:8081)

    U->>DB: POST /api/auth/login (email + password)
    DB->>GW: Validate credentials
    GW-->>DB: Issue JWT (access token)
    DB-->>U: Set JWT cookie

    U->>DB: Click "Launch Chat"
    DB->>GW: POST /api/auth/chat-handoff (JWT)
    GW-->>DB: One-time handoff token (30s TTL)
    DB-->>U: Redirect to Chat UI with handoff token

    U->>CU: GET /?token=<handoff>
    CU->>GW: POST /api/auth/validate-handoff
    GW-->>CU: Confirm identity, establish session
    CU-->>U: Logged in — chat session begins
```

---

## Data Flow: Full Request Lifecycle

```mermaid
flowchart TD
    A([User sends message]) --> B{Auth Valid?}
    B -->|No| C[401 Unauthorized]
    B -->|Yes| D[Extract prompt text]
    D --> E[Send to ML Engine]
    E --> F{Lane Classification}
    F -->|🔴 Red| G[Block or reroute to local AI]
    F -->|🟡 Yellow| H[Pseudonymize PII tokens]
    F -->|🟢 Green| I[Forward as-is]
    H --> J[Inject privacy system prompt]
    J --> K[Forward masked payload to cloud AI]
    K --> L[Stream response]
    L --> M[Real-time unmask tokens in stream]
    I --> L
    G --> N[Stream local AI response or block message]
    M & N --> O([User receives response])
    O --> P[Persist to audit log]
```

---

## Deployment Architecture

```mermaid
graph TD
    subgraph "Docker Host Machine"
        subgraph "docker-compose network: trustvault_default"
            A[trustvault-proxy<br/>ghcr.io/org/trustvault-proxy:latest]
            B[trustvault-pii-engine<br/>ghcr.io/org/trustvault-pii-engine:latest]
            C[trustvault-dashboard<br/>ghcr.io/org/trustvault-dashboard:latest]
            D[trustvault-chat-ui<br/>ghcr.io/org/trustvault-chat-ui:latest]
        end
        E[SQLite volume<br/>./data/trustvault.db]
        F[Ollama<br/>Native host process]
    end

    subgraph "GitHub Container Registry"
        G[Pre-built Docker Images]
    end

    G -->|docker compose pull| A & B & C & D
    A <-->|Internal Docker DNS| B
    A --- E
    A -->|host.docker.internal:11434| F
```

> **Zero Build Deployment:** Teammates pull pre-built images from GitHub Container Registry (GHCR). No Go compiler, Python environment, or Node.js installation is required on their machines.
