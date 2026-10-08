# 🧠 AI Business Training Platform

> A full-stack AI-powered business training and readiness platform combining **RAG, conversational AI, automated assessment, document intelligence, and role-based administration**.

Built as an end-to-end production-oriented application with **Next.js, TypeScript, PostgreSQL/Supabase, Pinecone, and Anthropic**.

---

## 🚀 Overview

This platform transforms business documents and training requirements into an interactive learning and assessment system.

It provides separate experiences for administrators and learners while integrating AI into multiple parts of the workflow:

- Business and user management
- Document ingestion and AI extraction
- RAG-powered knowledge retrieval
- AI-generated assessment questions
- MCQ quizzes and examinations
- RAG-grounded conversational readiness assessment
- Learner progression tracking
- Appointment and eligibility workflows
- Automated notifications
- PDF document generation
- Background processing with retries and job leases
- Role-based authorization and PostgreSQL Row-Level Security

The project was designed as a **full-stack system rather than an AI demo**, with security, reliability, testing, and operational constraints considered at the architecture level.

---

# ✨ Key Features

## 🏢 Business & User Management

Administrative workflows support:

- Business records
- User and learner management
- Role-based access
- Training assignments
- Business categories
- Appointments
- Notifications
- Assessment configuration

---

## 📄 Document Intelligence

Business documents can be uploaded and processed through an asynchronous pipeline.

```text
Document
   │
   ▼
Private Storage
   │
   ▼
Page-Level Processing
   │
   ▼
AI Extraction
   │
   ▼
Structured Data
   │
   ▼
Chunking
   │
   ▼
Embeddings
   │
   ▼
Pinecone
```

The processing pipeline is designed to handle large documents without relying on a single long-running serverless request.

See [`docs/document-processing.md`](docs/document-processing.md).

---

## 🔎 Retrieval-Augmented Generation

The platform uses RAG to ground AI workflows in business-specific knowledge.

```text
User Query
    │
    ▼
Query Embedding
    │
    ▼
Metadata-Constrained Retrieval
    │
    ▼
Candidate Passages
    │
    ▼
Reranking
    │
    ▼
Relevant Context
    │
    ▼
Anthropic
    │
    ▼
Grounded Response
```

The retrieval layer is reused by multiple workflows including:

- Administrative Q&A
- AI question generation
- Learner study content
- Conversational readiness assessment

The vector database is treated as a retrieval layer while PostgreSQL remains the application system of record.

See [`docs/rag-pipeline.md`](docs/rag-pipeline.md).

---

## 🤖 AI Question Generation

Assessment questions can be generated from business-specific knowledge.

```text
Business Documents
       │
       ▼
      RAG
       │
       ▼
Relevant Context
       │
       ▼
AI Question Generation
       │
       ▼
Structured Output
       │
       ▼
Schema Validation
       │
       ▼
Question Bank
```

Generated output is validated before entering structured application workflows.

This prevents malformed model responses from directly becoming application data.

---

## 💬 Conversational AI Readiness Assessment

The platform includes a conversational AI assessment that evaluates learner readiness using business-specific knowledge.

The system combines:

- Persistent interview sessions
- Predefined assessment flow
- RAG-based knowledge retrieval
- Natural-language interaction
- Structured answer evaluation
- Final readiness evaluation

Importantly, the application controls the assessment workflow while the LLM handles conversational understanding and evaluation.

```text
Assessment Flow
       │
       ▼
Question
       │
       ▼
Learner Answer
       │
       ▼
RAG Context
       │
       ▼
AI Evaluation
       │
       ▼
Persist Result
       │
       ▼
Next Defined Question
```

This provides conversational UX without delegating security-sensitive workflow control to an autonomous agent.

See [`docs/ai-interview.md`](docs/ai-interview.md).

---

# 🧑‍🎓 Learner Platform

Learners have access to an application flow covering:

- Dashboard
- Study content
- Quizzes
- Examinations
- Conversational readiness assessment
- Progression tracking
- Appointments
- Eligibility
- Name-card generation

Progression is derived from authoritative application data rather than relying on a single manually maintained status field.

See [`docs/adr/0003-derived-progression.md`](docs/adr/0003-derived-progression.md).

---

# 🔐 Security Architecture

Security is implemented at multiple layers.

### Authentication

Supabase Authentication manages user identity and sessions.

### Role-Based Access Control

Different user roles have different application capabilities.

### PostgreSQL Row-Level Security

RLS provides database-level data isolation.

```text
Authenticated User
        │
        ▼
Application Authorization
        │
        ▼
PostgreSQL
        │
        ▼
RLS Policy
        │
   ┌────┴────┐
   ▼         ▼
Allowed    Denied
```

### IDOR Protection

Record identifiers supplied by the client are treated as untrusted input.

Automated tests verify that manipulating identifiers cannot bypass ownership restrictions.

### Private Storage

Business documents are stored using private object storage and controlled access.

### Server-Side Secrets

AI provider credentials and privileged database credentials remain server-side.

### AI Data Boundary

RAG retrieval is constrained before information reaches the LLM.

> **RAG determines relevant information; authorization determines permitted information.**

See [`docs/security.md`](docs/security.md).

---

# ⚙️ Background Processing & Reliability

Expensive or failure-prone operations are handled asynchronously.

The project uses a database-backed job model with:

- Durable jobs
- Job leases
- Retry attempts
- Backoff
- Deterministic identifiers
- Persisted processing state
- Resumable workflows

```text
             Job Queue
                 │
                 ▼
           Claim + Lease
                 │
                 ▼
              Process
             /       \
            ▼         ▼
        Success     Failure
           │           │
           ▼           ▼
       Complete      Retry
                       │
                       ▼
                    Backoff
```

This architecture allows the application to work within serverless execution constraints while still supporting long-running document and AI workflows.

See [`docs/adr/0004-notifications-table-queue.md`](docs/adr/0004-notifications-table-queue.md) and [`docs/document-processing.md`](docs/document-processing.md).

---

# 🧪 Testing

Testing covers both conventional application behavior and AI-specific failure modes.

### Unit Tests

- Domain logic
- Validation
- Assessment logic
- RAG utilities
- Interview state
- Progression
- Processing helpers

### Integration Tests

- PostgreSQL
- Supabase
- RLS
- Authentication
- Document processing
- RAG
- AI workflows
- Background jobs

### End-to-End Tests

Playwright tests cover complete administrative and learner workflows.

### Security Tests

- RLS isolation
- Role authorization
- Cross-user access
- IDOR scenarios
- Protected routes

### AI Failure Tests

- Provider failures
- Invalid model output
- Schema validation failures
- Retrieval failures
- Processing interruptions

See [`docs/testing.md`](docs/testing.md).

---

# 🏗️ Architecture

At a high level:

```text
                         ┌─────────────────────┐
                         │       Browser       │
                         │ Admin / Learner UI  │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │      Next.js        │
                         │ App Router + APIs   │
                         │ Server Actions      │
                         └──────────┬──────────┘
                                    │
             ┌──────────────────────┼──────────────────────┐
             │                      │                      │
             ▼                      ▼                      ▼
      ┌─────────────┐       ┌─────────────┐       ┌─────────────┐
      │  Supabase   │       │   Pinecone  │       │  Anthropic  │
      │ PostgreSQL  │       │ Vector DB   │       │     AI      │
      │ Auth + RLS  │       │ Retrieval   │       │ Generation  │
      │   Storage   │       └─────────────┘       └─────────────┘
      └─────────────┘
             │
             ▼
      Background Jobs
      Queue / Lease / Retry
```

For a deeper explanation, see [`docs/architecture.md`](docs/architecture.md).

---

# 🧠 Engineering Decisions

Several architectural decisions were made to keep the system maintainable and reliable.

### Next.js + Supabase + Vercel

A unified full-stack application with managed PostgreSQL, authentication, storage, and serverless deployment.

See [`docs/adr/0001-stack.md`](docs/adr/0001-stack.md).

### RLS as the Isolation Boundary

Authorization is enforced at the database level rather than relying exclusively on application query filters.

See [`docs/adr/0002-rls-isolation.md`](docs/adr/0002-rls-isolation.md).

### Derived Progression

Progression is calculated from authoritative facts instead of relying on a potentially stale aggregate status.

See [`docs/adr/0003-derived-progression.md`](docs/adr/0003-derived-progression.md).

### Database-Backed Background Jobs

PostgreSQL is used as a lightweight durable queue for asynchronous operations where introducing a separate queue service would add unnecessary infrastructure.

See [`docs/adr/0004-notifications-table-queue.md`](docs/adr/0004-notifications-table-queue.md).

### Database-Enforced Business Rules

Critical eligibility calculations are protected at the database boundary while corresponding application logic provides predictable domain behavior.

See [`docs/adr/0005-eligibility-enforced-in-database.md`](docs/adr/0005-eligibility-enforced-in-database.md).

---

# 🛠️ Technology Stack

| Area                 | Technologies                              |
| -------------------- | ----------------------------------------- |
| Frontend             | Next.js, React, TypeScript, Tailwind CSS  |
| Backend              | Next.js Server Actions, API Routes        |
| Database             | PostgreSQL, Supabase                      |
| Authentication       | Supabase Auth                             |
| Authorization        | PostgreSQL RLS, Role-Based Access Control |
| AI                   | Anthropic                                 |
| RAG                  | Pinecone, Embeddings, Reranking           |
| Validation           | Zod                                       |
| Documents            | PDF.js, PDF-Lib, React PDF, Mammoth       |
| Testing              | Vitest, Playwright                        |
| Local Infrastructure | Docker, Supabase CLI                      |
| Deployment           | Vercel                                    |
| Development          | pnpm, Git, ESLint, Prettier               |

---

# 📁 Project Structure

```text
.
├── app/
│   ├── api/
│   ├── admin/
│   ├── auth/
│   └── learner/
│
├── components/
│
├── lib/
│   ├── auth/
│   ├── db/
│   ├── domain/
│   ├── integrations/
│   └── pdf/
│
├── scripts/
│
├── supabase/
│   ├── migrations/
│   └── seed/
│
├── tests/
│   ├── e2e/
│   ├── integration/
│   └── unit/
│
├── docs/
│   ├── adr/
│   ├── architecture.md
│   ├── rag-pipeline.md
│   ├── ai-interview.md
│   ├── document-processing.md
│   ├── security.md
│   └── testing.md
│
└── README.md
```

---

# 🚀 Local Development

## Requirements

- Node.js
- pnpm
- Docker Desktop
- Supabase CLI

## Install

```bash
pnpm install
```

## Start Local Database

```bash
pnpm db:start
```

## Reset Local Database

```bash
pnpm db:reset
```

## Generate Local Environment

```bash
pnpm db:env
```

## Run Development Server

```bash
pnpm dev
```

The application can then be accessed through the local Next.js development server.

---

# 🧪 Development Commands

### Lint

```bash
pnpm lint
```

### Typecheck

```bash
pnpm typecheck
```

### Unit Tests

```bash
pnpm test:unit
```

### Integration Tests

```bash
pnpm test:integration
```

### End-to-End Tests

```bash
pnpm test:e2e
```

### Formatting

```bash
pnpm format
```

---

# 👩‍💻 My Contribution

I designed and implemented the platform end-to-end from business requirements.

My responsibilities included:

### Architecture

- Designed the overall application architecture.
- Defined boundaries between UI, domain logic, database, AI integrations, and background processing.
- Designed the PostgreSQL schema and relationships.
- Designed authentication, authorization, and RLS policies.

### AI & RAG

- Designed and implemented the document-to-RAG pipeline.
- Integrated Pinecone vector search.
- Implemented retrieval filtering and reranking.
- Integrated Anthropic for conversational and generation workflows.
- Designed RAG-grounded AI question generation.
- Implemented structured AI output validation.
- Designed and implemented the conversational readiness assessment.

### Backend

- Built Next.js API routes and server actions.
- Implemented database operations and CRUD workflows.
- Implemented business rules and progression logic.
- Built background processing with queues, leases, retries, and scheduled jobs.
- Designed workflows around serverless execution constraints.

### Security

- Implemented authentication and role separation.
- Designed PostgreSQL RLS policies.
- Implemented cross-user isolation.
- Added IDOR and authorization tests.
- Kept privileged operations and external AI credentials server-side.

### Testing

- Implemented unit tests.
- Implemented integration tests.
- Implemented end-to-end tests.
- Added RLS and IDOR security coverage.
- Added AI provider failure tests.
- Added document-processing and large-input failure scenarios.

### Frontend

- Built both administrative and learner-facing interfaces.
- Implemented dashboards, assessments, study flows, interview interfaces, appointments, and progression views.

AI-assisted development tools were used during implementation, while the architecture, system design, application logic, integration decisions, testing strategy, and debugging were reviewed and directed throughout development.

---

# 📌 Project Status

The project is in the final stage of development and deployment preparation.

The architecture is designed for deployment using:

```text
Vercel
   │
   ├── Next.js Application
   │
   └── Server-side Workflows
          │
          ├── Supabase
          ├── Pinecone
          └── Anthropic
```

The repository contains a sanitized version of the project intended to demonstrate the engineering architecture and implementation patterns without exposing private business or user information.

---

# 🔒 Public Repository Notice

This repository is a sanitized portfolio version of a larger business application.

Private client information, credentials, production secrets, and sensitive business data have been excluded.

The code and documentation are presented to demonstrate:

- Full-stack architecture
- AI integration
- RAG implementation
- Backend engineering
- Database design
- Security architecture
- Reliability patterns
- Testing strategy

---

# 📚 Technical Documentation

| Document                                           | Description                                                      |
| -------------------------------------------------- | ---------------------------------------------------------------- |
| [Architecture](docs/architecture.md)               | Overall application architecture                                 |
| [RAG Pipeline](docs/rag-pipeline.md)               | Retrieval, embeddings, reranking, and grounded generation        |
| [AI Interview](docs/ai-interview.md)               | Conversational readiness assessment architecture                 |
| [Document Processing](docs/document-processing.md) | Large-document processing, queues, retries, and resumability     |
| [Security](docs/security.md)                       | Authentication, RLS, IDOR protection, and AI security boundaries |
| [Testing](docs/testing.md)                         | Unit, integration, E2E, security, and AI failure testing         |

### Architecture Decision Records

- [ADR 0001 — Next.js + Supabase + Vercel](docs/adr/0001-stack.md)
- [ADR 0002 — RLS as the Data Isolation Boundary](docs/adr/0002-rls-isolation.md)
- [ADR 0003 — Derived Progression State](docs/adr/0003-derived-progression.md)
- [ADR 0004 — Database-Backed Job Queue](docs/adr/0004-notifications-table-queue.md)
- [ADR 0005 — Database-Enforced Business Rules](docs/adr/0005-eligibility-enforced-in-database.md)

---

## ⭐ Why This Project Matters

This project demonstrates the intersection of **AI engineering and full-stack software engineering**.

Rather than building a standalone chatbot, the system integrates AI into a broader application containing:

**RAG + Conversational AI + Structured Generation + PostgreSQL + Authentication + RLS + Background Jobs + Testing + Full-Stack UX**

The key engineering challenge was not simply connecting an LLM API.

It was designing the surrounding system so that AI functionality remained:

- Grounded
- Validated
- Secure
- Testable
- Recoverable
- Integrated with deterministic application workflows
