# ProfilerX — Architecture & Flow Diagrams

Visual reference for system topology, technology choices, and main request paths. For narrative detail see [ARCHITECTURE.md](./ARCHITECTURE.md) and [LOW_LEVEL_DESIGN.md](./LOW_LEVEL_DESIGN.md).

---

## Technology overview

ProfilerX is a **decoupled SPA + REST API** portfolio platform with an optional **dedicated chat edge** for LLM traffic in Docker.

| Layer | Technologies | Purpose |
|--------|----------------|---------|
| **Public UI** | React 18, Vite 5, Tailwind CSS, React Router, Zustand, Axios, Framer Motion, DOMPurify, react-quill | Portfolio pages, Ask Me UI, admin console; centralized API client and local Basic-auth token handling |
| **Edge (Docker)** | OpenResty (nginx + Lua) in `frontend` image | Static assets, reverse proxy of browser API traffic to Spring Boot; optional cache purge hook from backend |
| **Core API** | Java 17, Spring Boot 3.2, Spring Web, Spring Security, Spring Data JPA, Bean Validation | REST endpoints, HTTP Basic for admin, JPA persistence, DTO contracts |
| **Chat edge** | Java 17, Helidon SE 3.2 (`chat-bot-service`) | Ask Me handler on a worker pool; server-to-server calls into Spring Boot internal integration surface; invokes Gemini/Ollama |
| **Persistence** | Spring Data JPA, Hibernate | Entities and repositories; schema evolution via configured `ddl-auto` |
| **Database** | PostgreSQL (typical in Compose/prod) or H2 file (local/dev options) | Profile aggregate, analytics, audit, bot history, email bookkeeping, system settings |
| **Files** | Paths under `PORTFOLIO_DATA_DIR` | Profile photo, email-pack attachments, generated PDFs — kept on disk with DB metadata |
| **Documents** | PDFBox, Apache POI | Resume PDF generation; PDF/DOCX text extraction on admin upload |
| **Email** | Brevo HTTP API (`BrevoTransactionalMailService`), optional JavaMail SMTP | Transactional mail and profile pack delivery |
| **AI** | Pluggable Gemini / Ollama | Backend `LlmClient`; chat-bot `LlmHttpClient` for completions |
| **Ops** | Docker Compose, GitHub Actions (`deploy.yml`) | Local/prod topology and deploy automation |

**Cross-cutting:** CORS from `portfolio.cors.allowed-origins`; shared server secret for chat-bot → Spring internal integration; servlet filters for rate limits on Ask Me, intent scoring, and profile-email flows.

---

## System architecture (logical layers)

```mermaid
flowchart TB
  subgraph Client["Client"]
    Browser[Browser]
  end

  subgraph Presentation["Presentation"]
    React[React SPA — Vite]
  end

  subgraph Edge["Edge (optional in Docker)"]
    OpenResty[OpenResty — static + API gateway to core]
    Helidon[Helidon chat-bot — Ask Me handler]
  end

  subgraph Application["Application — Spring Boot"]
    Controllers[Controllers]
    Services[Services — profile, AI, email, analytics, PDF]
    Security[Security + protection filters]
  end

  subgraph Data["Data"]
    JPA[(JPA / Hibernate)]
    DB[(PostgreSQL or H2)]
    FS[File storage — data dir]
  end

  subgraph External["External integrations"]
    LLM[Ollama / Gemini APIs]
    Brevo[Brevo / SMTP]
  end

  Browser --> React
  React -->|HTTP JSON / multipart| OpenResty
  OpenResty -->|core API traffic| Controllers
  React -->|Ask Me questions| Helidon
  Helidon -->|internal integration + service credential| Controllers
  Controllers --> Security
  Security --> Services
  Services --> JPA
  JPA --> DB
  Services --> FS
  Services --> Brevo
  Services --> LLM
  Helidon --> LLM
```

---

## Runtime topology (Docker Compose — conceptual)

```mermaid
flowchart LR
  Browser[Browser]
  FE[frontend :80 — OpenResty]
  CB[chat-bot :8081 — Helidon]
  BE[backend :8080 — Spring Boot]
  DB[(PostgreSQL or H2 volume)]
  LLM[Ollama / Gemini]
  Brevo[Brevo API / SMTP]

  Browser --> FE
  FE -->|proxied API to core| BE
  Browser -->|Ask Me| CB
  CB -->|internal integration (authenticated)| BE
  BE --> DB
  BE --> Brevo
  CB --> LLM
  BE -.->|optional AI from core| LLM
```

---

## Request flow — public profile load

```mermaid
flowchart TD
  A[User opens home] --> B[React loads public profile]
  B --> C[ProfileController]
  C --> D[ProfileService assembles ProfileDTO]
  D --> E[(Database — profile + collections)]
  E --> D
  D --> F[200 + cache headers]
  F --> G[UI renders sections]
```

---

## Request flow — Ask Me (Docker path via Helidon)

```mermaid
flowchart TD
  A[User submits question] --> B[Chat-bot receives Ask Me request]
  B --> C[Validate + rate limit]
  C --> D[Fetch AI runtime config from core]
  D --> E[Fetch LLM-oriented profile bundle from core]
  E --> F[Build prompt]
  F --> G[LLM completion — Gemini/Ollama]
  G --> H[Sanitize answer]
  H --> I[Optional persist Q&A metadata to core]
  I --> J[JSON response to UI]
```

---

## Request flow — admin resume upload (simplified)

```mermaid
flowchart TD
  A[Admin authenticated — Basic] --> B[Admin uploads resume document]
  B --> C[Extract text — PDF/DOCX]
  C --> D[Process + persist profile]
  D --> E[(JPA + files on disk)]
  E --> F[Optional cache purge / profile reload]
  F --> G[Admin UI refreshes]
```

---

## Request flow — send profile email

```mermaid
flowchart TD
  A[Visitor requests profile by email] --> B[SendProfileProtectionFilter]
  B -->|pass| C[SendProfileService composes message]
  C --> D[Brevo / SMTP]
  D --> E[Success or failure persistence]
  E --> F[Response to client]
  B -->|reject| G[Rate limit / validation error]
```

---

## Sequence — Ask Me (chat-bot and core)

```mermaid
sequenceDiagram
  participant UI as React
  participant CB as Helidon chat-bot
  participant BE as Spring Boot
  participant LLM as Gemini/Ollama

  UI->>CB: Ask Me question
  CB->>BE: Read AI runtime configuration (internal)
  CB->>BE: Read profile bundle for prompts (internal)
  CB->>LLM: Completion request
  LLM-->>CB: Generated text
  CB->>BE: Record Q&A metadata (internal, optional)
  CB-->>UI: Answer payload
```

---
