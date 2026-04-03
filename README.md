# Campus AI Agent — Architecture Design

An AI agent designed for college campuses that handles event information, facility availability, and room/lab booking through a conversational interface.

---

## Overview

The agent accepts natural language queries from students and staff, identifies the intent behind each request, retrieves data from the relevant campus systems, validates institutional constraints, and — when a booking action is required — confirms the user's intent before executing any write operation.

---

## Features

- Identifies whether a query relates to **events**, **facilities**, or **bookings**
- Retrieves live data from campus calendar, facilities database, and reservation systems
- Validates availability, operating hours, capacity, and user eligibility before responding
- Requires **explicit user confirmation** before creating any booking
- Redacts PII and applies policy filters on every response
- Logs every interaction for audit and continuous improvement

---

## Architecture

The system is organised into two parallel flows:

**Main flow (left column)**
```
User query
  → NLP pre-processing
  → Intent classifier
  → Domain routers (Events / Facilities / Bookings)
  → Data retrieval layer
  → Context builder
  → Constraint validator
  → Decision gate (action needed?)
      → No  → Response generator → Guardrails → Response to user → Audit log
```

**Action path (right column — booking intent only)**
```
Decision gate (yes)
  → Confirmation gate (user confirms or cancels)
  → Action executor (write booking, send receipt)
  → Merges back into Response generator
```

The interactive architecture diagram is available at:
**[`campus_ai_agent_architecture_v2.html`](./campus_ai_agent_architecture_v2.html)**

Open it in any browser. Click any node to see component details.

---

## Component Breakdown

### NLP Pre-processing
**Library:** spaCy + custom Named Entity Ruler

Tokenises the raw input, extracts named entities (room names, dates, times, event titles, people), and normalises synonyms so that "reserve", "book", and "schedule" are treated identically. spaCy was chosen for its modular pipeline architecture and fast CPU inference — critical for a low-latency conversational system.

---

### Intent Classifier
**Library:** Hugging Face Transformers (`bert-base-uncased`)

Fine-tuned on campus-specific utterances to classify each query into one of four labels: `events`, `facilities`, `bookings`, or `ambiguous`. If the top two confidence scores are within 15% of each other, the agent asks a clarifying question rather than guessing. Fine-tuning on in-domain data significantly reduces misclassification compared to zero-shot approaches.

---

### Domain Routers
**Pattern:** Dispatcher functions (one per intent)

Each router knows which downstream API or database to call and which fields are required before proceeding. Isolating routers by domain means a new domain (e.g. parking, IT support) can be added without modifying existing logic.

---

### Data Retrieval Layer
**Library:** LangChain tool wrappers + SQLAlchemy + httpx

Three connectors are wrapped as LangChain `Tool` objects:
- **Events API** — campus calendar REST API (schedule, speakers, venue, capacity)
- **Facilities DB** — PostgreSQL database via SQLAlchemy (room specs, hours, equipment)
- **Booking system** — reservation REST/SOAP API (existing slots, user history, policies)

LangChain's Tool abstraction provides unified retry logic, error handling, and a consistent result schema.

---

### Context Builder
**Library:** BM25 (rank-bm25) + custom truncation logic

Merges results from all retrieved sources, scores them by relevance to the original query using BM25, and truncates to fit within the LLM's token budget (~3000 tokens). Handles the case where one retrieval returns no results and a fallback message must be prepared.

---

### Constraint Validator
**Pattern:** Deterministic rule-based Python functions

Applies hard institutional checks before any action is taken:
- Is the requested time slot within operational hours?
- Does the room capacity meet the group size?
- Is the user eligible to book this resource?
- Is the slot already reserved?

These are **not** delegated to the LLM — using deterministic code makes the constraint logic fully auditable and unit-testable, independent of the language model.

---

### Decision Gate
A simple conditional branch: if the intent is informational, route directly to response generation. If the intent requires a write action, route to the confirmation gate first. This is the most critical branching point in the entire flow.

---

### Confirmation Gate
**Pattern:** Structured LLM prompt + front-end confirm/cancel buttons

Serialises the proposed action into a plain-English summary ("Book Lab 304, Friday 3–5 pm, for 12 students — confirm?") and waits for explicit user approval. No booking is created without this confirmation. On cancellation, the conversation state resets to the intent classification stage.

---

### Action Executor
**Library:** httpx (async HTTP client)

POSTs the booking to the reservation API with an idempotency key (derived from user ID + room + timeslot hash) to prevent duplicate reservations on network retry. Captures the booking reference number and passes it to the response generator.

---

### Response Generator
**Library:** LangChain LLMChain + Claude / GPT-4o via API

Uses a campus-branded system prompt to turn structured context and action results into a friendly, grammatically correct response. Common response types (availability list, event details, booking receipt) use slot-filled templates rather than free-form generation — this keeps output consistent and prevents hallucinated details.

---

### Guardrails & Safety Check
**Library:** Microsoft Presidio

Runs before every response reaches the user:
- Redacts PII (student IDs, personal emails, phone numbers)
- Checks for policy violations (must not reveal other users' booking data)
- Applies a fallback template if the LLM output is incoherent or hedged

---

### Audit Log & Feedback Loop
**Libraries:** structlog + Elasticsearch

Every interaction is written to a structured log store with full trace: intent label, retrieved data, constraint verdict, confirmation outcome, and final response. Low-rated interactions are queued for human review and used as negative examples in the next fine-tuning cycle for the intent classifier.

---

## Technology Stack

| Component | Library / Framework | Justification |
|---|---|---|
| NLP pipeline | spaCy + custom NER | Fast CPU inference, modular, easy to extend |
| Intent classification | Hugging Face Transformers | Fine-tuneable on small domain datasets |
| Agent orchestration | LangChain | Unified tool interface, prompt chaining, retry logic |
| LLM backbone | Claude Sonnet / GPT-4o | High-quality instruction following, structured output |
| Data access | SQLAlchemy + httpx | ORM for facilities DB; async HTTP for APIs |
| PII redaction | Microsoft Presidio | Configurable entity recognition, GDPR-aware |
| Logging | structlog + Elasticsearch | Machine-readable logs, queryable for analytics |
| Deployment | FastAPI + Docker | Async, lightweight API layer; containerised for campus IT |

---

## Key Design Decisions

**Constraint validation is deterministic, not LLM-based.**
Availability checks, policy enforcement, and capacity rules are implemented as Python functions — not delegated to the language model. This makes the logic auditable, unit-testable, and independent of the LLM version in use.

**Mandatory confirmation before any write action.**
Even when the agent is highly confident about a booking request, no reservation is created without an explicit user confirmation step in the same session. This prevents accidental double-bookings and wrong-room reservations.

**Separate domain routers per intent.**
Rather than a single monolithic handler, each intent domain has its own router. This keeps concerns isolated, makes each router independently testable, and allows new domains to be added cleanly.

**Two-column flow separation.**
The main informational flow and the booking action path are architecturally separated. This makes it clear that the action path is an exception, not the default, and reduces the risk of unintended write operations.

---

## Files

| File | Description |
|---|---|
| `campus_ai_agent_architecture_v2.html` | Interactive architecture diagram (open in browser) |
| `README.md` | This document |

---

## How to View the Diagram

1. Download `campus_ai_agent_architecture_v2.html`
2. Open it in any modern browser (Chrome, Firefox, Safari, Edge)
3. Click any node in the diagram to see component details
4. No server or installation required — fully self-contained

Or, if hosted on GitHub Pages, visit:
`https://<your-username>.github.io/<your-repo-name>/campus_ai_agent_architecture_v2.html`
