# TRIMURTI — AI-Native Investigation Platform

An AI-native case-management and investigation platform for law enforcement, covering four domains: **EOW** (Economic Offences), **NDPS** (Narcotics), **BNS** (General Crime), and **CYBER** (Cybercrime). It digitizes evidence, makes it searchable by meaning, maps connections between people/accounts/devices, and gives investigators an AI agent that can search and reason over case evidence — while keeping every action hashed, audit-logged, and legally admissible.

> **Note on this repository:** this project is currently in early-stage discussions with the Delhi Cyber Crime Branch. To protect it while those discussions are ongoing, the source code has not been uploaded here — only this README, describing the system's design and capabilities. Happy to walk through the codebase directly on request.

---

## Project Overview

Criminal investigations generate a flood of unstructured evidence — scanned FIRs, bank statements, call records, chat screenshots, seized-device extractions — spread across paper files, phones, and PDF exports, with no single place connecting them. TRIMURTI turns that raw evidence into a searchable, connected, AI-assisted investigation workspace: one shared FastAPI/PostgreSQL backend serving a web console for desk-based investigators and an offline-first Android field app for officers on the ground, with every piece of evidence hashed and audit-logged so it stays admissible under India's Section 65B / BSA Section 63 evidentiary requirements.

---

## Problem Statement

Investigators today are drowning in unstructured evidence with no easy way to:
- **Search across it** — finding whether a name appears anywhere in 200 pages of bank records means physically reading all 200 pages.
- **Find hidden connections** — a phone number appearing across three unrelated documents, or a shared account linking two separate cases, is easy to miss without manual cross-referencing.
- **Ask questions about a case** without reading every document themselves.
- **Keep it legally sound** — whatever tooling exists still has to produce evidence that holds up in court, provably unaltered.

TRIMURTI addresses all four: digitize → structure → index by meaning → connect → let an AI agent search and reason over it → keep everything provably intact.

---

## Features

- **Document digitization (OCR)** — a fast native-text pass (PyMuPDF) combined with an AI-vision (Gemini) reconciliation pass for every page, running asynchronously via a Celery background queue so large uploads never block the officer. Per-page status tracking means one bad scan never blocks the rest of a document.
- **Document-type-aware chunking** — six distinct chunking strategies (bank statements, call detail records, chat screenshots, FIRs, court orders, generic text), each preserving the structure specific to that document type, with automatic fallback so nothing ever fails to index.
- **Hybrid RAG search** — a single SQL query combining PostgreSQL full-text (keyword) search with pgvector semantic (embedding) search, custom-weighted to favor exact-match accuracy — built directly on Postgres, no third-party RAG framework.
- **Reranking cascade** — a tiered, first-success-wins reranking layer (external API → local cross-encoder → LLM judge → deterministic fallback) that refines search results without ever leaving search unavailable.
- **Magic Intake** — AI-assisted case creation: uploads an initial complaint/FIR, extracts structured fields (names, dates, case category) via an LLM, and prefills a form for the officer to review and confirm — no case is created without an explicit human step.
- **Tool-calling AI agent runtime (ReAct pattern)** — a registry of 745 domain-specialist agent personas across the 4 case domains; an agent autonomously decides when to search evidence, reasons over what it finds, and answers, instead of the officer digging manually.
- **Multi-agent orchestration** — for questions that genuinely span more than one investigative thread, a supervisor decomposes the question, dispatches independent sub-questions to specialist personas running in parallel (Celery group+chord), supports agent-to-agent handoff for sequential dependencies, and synthesizes the results into one answer — explicitly surfacing disagreements between specialists rather than silently picking one side.
- **Self-critique safety layer** — a second, independent AI pass validates every agent response against retrieved evidence and checks for unauthorized-action claims before it's ever shown to the officer.
- **Permission-gated tool execution** — any sensitive action (freezing an account, generating a legal certificate) is checked against the real, database-backed permissions of the logged-in user — never against an AI's own claim of approval.
- **Entity graph & resolution** — extracted entities (people, accounts, phone numbers, devices) are linked into a Memgraph-backed investigation graph, with write-time dedup and a human-reviewed similarity-merge queue for cross-document/cross-case resolution.
- **Legal compliance** — Section 65B / BSA Section 63 certificate generation with Merkle-root evidence hashing, hash-chain verification, and RSA signing; a genuinely tamper-evident, hash-chained audit log for every state change.
- **Offline-first Android field app** — evidence capture, interrogation recording, and case work that functions with no signal and syncs when connectivity returns, with hardware-backed evidence signing at capture.

---

## Architecture

```
Angular SPA ─cookies+CSRF─┐
Android app ─Bearer JWT───┤──► FastAPI /api/v1 (33 routers · 239 REST + 3 WS)
                           │      └─ services/ (45+ modules) ─► model_gateway (multi-LLM)
                           │                                  ├─ agent runtime + 745-persona registry
                           │                                  ├─ multi-agent orchestration (Celery group+chord)
                           │                                  ├─ hybrid RAG (pgvector + full-text)
                           │                                  └─ graph intelligence (Memgraph)
                           │      └─ Celery Beat sweeps every 30 min
                           │         (entity-embedding + semantic-link + link-prediction refresh)
                           ▼
        Postgres + pgvector · Memgraph · Valkey/Celery (+Beat) · MinIO/S3
```

**Layering:** endpoints validate + authorize + delegate only; all business logic lives in `services/`; every SQL write goes through a shared mutations layer that writes an audit-log row with a real, lock-serialized tamper-evident hash chain. All LLM/embedding calls route through one shared model-gateway abstraction — no service ever calls a provider SDK directly — preserving runtime provider-switching, fallback, and budget accounting across every AI feature.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | Python 3.12, FastAPI, SQLAlchemy 2, Alembic, Celery |
| Database | PostgreSQL 14 + pgvector |
| Graph | Memgraph (Bolt/Cypher, MAGE algorithms) |
| Queue / Cache | Valkey (Redis-compatible) + Celery + Celery Beat |
| AI / LLM | Google Gemini (primary), provider-swappable via a shared gateway (Anthropic/OpenAI/NVIDIA/Ollama-compatible) |
| Web frontend | Angular 21, Tailwind 4, standalone components |
| Mobile | Kotlin, Jetpack Compose, Hilt, Room + SQLCipher (offline-first) |
| Object storage | MinIO / S3-compatible |
| Testing | pytest, vitest, Gradle unit tests |

---

## Screenshots

*(Add screenshots of the web console — dashboard, investigation workspace, graph view, agent chat — and the Android field app's key screens here before publishing.)*

| Web Console | Android Field App |
|---|---|
| _screenshot placeholder_ | _screenshot placeholder_ |

---

## Demo Video

*(Add a short walkthrough video link here — e.g. a Loom/YouTube link showing: upload a document → watch it get extracted → search evidence → ask the agent a question → view the graph.)*

---

## Results

- **239 REST endpoints + 3 WebSockets** across 33 routers, covering case lifecycle, evidence, entities/graph, AI agents, and legal compliance.
- **745 specialist agent personas** across 64 categories and 4 case domains (46 hand-curated, 699 generated and quality-scored).
- **6 document-chunking strategies** with automatic fallback, ensuring retrieval never silently fails.
- **Real, tamper-evident audit hash chain** verified clean under 60 concurrent writers.
- **Multi-agent orchestration** verified end-to-end at the infrastructure level: migration tested (upgrade → downgrade → upgrade) against a live Postgres instance; Celery chord/parallel-dispatch mechanics verified against a real worker and a real queue — a real task-registration bug was caught and fixed during this testing, something a mocked test alone would not have surfaced.
- Built with an explicit honesty discipline throughout: every simulated/heuristic data source (external registry lookups, jurisdiction risk scoring) is labeled as such at the API and UI boundary rather than presented as real evidence.

---

## My Contribution

I designed and built the platform's AI/retrieval core end-to-end:

- The **hybrid RAG pipeline** from scratch on raw PostgreSQL + pgvector (no retrieval framework), including the keyword/semantic fusion scoring, candidate overfetching, and the four-tier reranking cascade.
- The **document-type-aware chunking system** and dual-embedding indexing across six document formats, with deterministic fallback handling.
- The **dual-pipeline OCR system** (fast native-text extraction + AI-vision reconciliation), running asynchronously with per-page failure isolation and automatic retry.
- **Magic Intake**, the AI-assisted case-creation flow.
- The **tool-calling ReAct agent runtime**, including the manual tool-call parser used for providers without native tool-calling support.
- The **self-critique safety layer** and the permission-gated tool-execution model that never trusts an AI's own claim of authorization.
- The **multi-agent orchestration layer** — supervisor decomposition, parallel sub-agent dispatch via Celery, agent-to-agent handoff, and cross-agent synthesis with contradiction-surfacing — built additively on top of the existing single-agent runtime without disturbing it, and verified at the database/queue level against real infrastructure rather than mocks alone.

Throughout, I prioritized honest engineering over polish: every fallback path, every simulated data source, and every known limitation is explicitly labeled in the code and documentation rather than glossed over — because this is a system of legal record, and overclaiming reliability here has real consequences.
