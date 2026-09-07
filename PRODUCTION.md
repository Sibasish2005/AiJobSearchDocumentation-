# AgentHire — Production Engineering Specification

> **Document status:** Production Blueprint / Engineering Contract  
> **Version:** 1.0.0  
> **Owner:** Solo Developer / Tech Lead  
> **Project:** AgentHire — Autonomous, Human-in-the-Loop Job Application & Interview-Prep Platform  
> **Last updated:** 2026-09-07

---

## 1. Executive Summary

AgentHire is a production-oriented agentic platform that turns a user's resume and job-search intent into a controlled application pipeline:

**Resume → Job Discovery → Explainable Match → Outreach Draft → Human Approval → Gmail Send → Inbox Monitoring → Follow-up → Response Classification → Job Dossier → Voice Interview → STAR Scorecard**

The system is intentionally designed as a **modular monolith with asynchronous workers**, rather than premature microservices. The architecture isolates domain modules, AI orchestration, external tool integrations, persistence, and infrastructure so individual bottlenecks can later be extracted into services.

The defining safety property is:

> **No outbound email may be sent solely because an agent decided to send it. A human approval state is required.**

The project is designed to demonstrate senior-level engineering across:

- requirements and non-functional requirements
- API and domain architecture
- PostgreSQL/pgvector data modeling
- Redis caching, queues, locks, and rate limiting
- asynchronous and scheduled processing
- durable LangGraph workflows
- human-in-the-loop agent execution
- MCP tool integration
- OAuth/security/least privilege
- RAG and hybrid retrieval
- AI evaluation and cost control
- observability and SRE
- Docker and CI/CD
- AWS deployment and infrastructure as code
- testing, failure injection, load testing, and disaster recovery
- architecture decision records and postmortems

---

# 2. Engineering Principles

## 2.1 Primary principles

1. **Correctness before cleverness**
2. **Human control before autonomous side effects**
3. **PostgreSQL is the source of truth**
4. **Redis is disposable infrastructure**
5. **Every external side effect must be idempotent**
6. **External content is untrusted input**
7. **Agents are workflows, not magic**
8. **Every important automated decision is observable**
9. **Failures must be recoverable**
10. **Prefer modular monolith → services only when justified by measured bottlenecks**
11. **Keep domain logic out of HTTP routes**
12. **Use explicit contracts between layers**
13. **Every production decision has a documented trade-off**
14. **Minimize cloud cost without hiding reliability problems**
15. **Design for operation six months after launch**

---

# 3. Product Scope

## 3.1 Core capabilities

### Resume Management
- PDF/DOCX upload
- parsing into structured JSON
- skill/entity extraction
- resume versioning
- embedding generation
- OCR fallback
- resume-to-job semantic matching

### Job Discovery
- job-board/API ingestion
- company career-page ingestion
- pasted job URLs
- deduplication
- deterministic skill overlap
- semantic similarity
- explainable match score
- missing-skill analysis

### Outreach
- direct application mode
- recruiter/networking mode
- ATS helper mode
- grounded drafting
- factuality guardrails
- draft versioning
- human approval/edit/regenerate

### Gmail Integration
- Google OAuth2
- standalone Gmail MCP server
- send email
- search inbox
- retrieve thread
- thread correlation
- token refresh
- idempotent sending
- send quotas

### Pipeline
- application lifecycle
- immutable audit log
- status transitions
- follow-up eligibility
- response detection

### Interview Preparation
- company/role research
- Job Dossier
- browser voice interview
- transcript
- STAR scorecard
- improvement recommendations

---

# 4. Non-Goals

Version 1 explicitly does **not** include:

- automatic blind applications
- automatic outbound sending without approval
- phone calls
- mobile application
- multi-tenant billing
- payment processing
- arbitrary email providers
- unrestricted browser automation
- unrestricted agent tool access
- autonomous hiring decisions

---

# 5. Personas and Trust Boundaries

## 5.1 User

The authenticated user owns:

- resume data
- Gmail connection
- drafts
- applications
- interview data
- approval decisions

## 5.2 Agent

The agent may:

- read authorized application context
- analyze resumes and jobs
- generate drafts
- classify responses
- create dossiers
- conduct interviews

The agent may **not** independently cross the outbound-email approval boundary.

## 5.3 External systems

Untrusted external sources include:

- job descriptions
- recruiter emails
- company pages
- job-board content
- email bodies
- web research results

External content is treated as data, never as system instructions.

---

# 6. Architecture

## 6.1 Logical architecture

```text
                         ┌─────────────────────┐
                         │       Browser       │
                         │ Next.js / React / TS│
                         └──────────┬──────────┘
                                    │ HTTPS
                                    ▼
                     ┌──────────────────────────┐
                     │ CDN / WAF / Load Balancer│
                     └────────────┬─────────────┘
                                  │
                                  ▼
                    ┌────────────────────────────┐
                    │       FastAPI API          │
                    │      Modular Monolith      │
                    ├────────────────────────────┤
                    │ Auth                       │
                    │ Resume                     │
                    │ Jobs                       │
                    │ Matching                   │
                    │ Applications               │
                    │ Drafts                     │
                    │ Inbox                      │
                    │ Dossiers                   │
                    │ Interviews                 │
                    │ Audit                       │
                    │ AI Orchestration            │
                    └───────┬───────────┬────────┘
                            │           │
                    ┌───────▼───┐   ┌──▼───────────┐
                    │ PostgreSQL │   │ Redis        │
                    │ + pgvector │   │ Cache/Queue  │
                    └───────┬────┘   └──────┬───────┘
                            │               │
                            │          ┌────▼─────┐
                            │          │ Workers   │
                            │          │ AI/Email  │
                            │          └────┬─────┘
                            │               │
                            │          ┌────▼──────────┐
                            │          │ LangGraph     │
                            │          │ Durable State │
                            │          └────┬──────────┘
                            │               │
                            │          ┌────▼──────────┐
                            │          │ MCP Client    │
                            │          └────┬──────────┘
                            │               │
                            │          ┌────▼──────────┐
                            │          │ Gmail MCP     │
                            │          │ Server        │
                            │          └────┬──────────┘
                            │               │
                            │          ┌────▼──────────┐
                            │          │ Gmail API     │
                            │          └───────────────┘
                            │
                     ┌──────▼─────────┐
                     │ Observability  │
                     │ Logs/Metrics/  │
                     │ Traces/Evals   │
                     └────────────────┘
```

---

# 7. Deployment Topology

## 7.1 Initial production topology

```text
Internet
   │
   ▼
DNS
   │
   ▼
CDN / WAF
   │
   ├──────────────► Next.js
   │
   └──────────────► FastAPI
                         │
             ┌───────────┼────────────┐
             ▼           ▼            ▼
          Postgres     Redis        Object Storage
             │           │
             │           ▼
             │        Workers
             │           │
             │        LangGraph
             │           │
             │        MCP Client
             │           │
             │        Gmail MCP
             │           │
             └───────────┼────────────► Gmail
                         │
                         ▼
                   Observability
```

## 7.2 Scaling topology

```text
                 Load Balancer
                 /     |      \
                /      |       \
             API-1   API-2    API-N
                \      |       /
                 \     |      /
                   PostgreSQL
                       │
                   Read Replica
                       │
                     Redis
                       │
             ┌─────────┼─────────┐
             ▼         ▼         ▼
          Worker-1  Worker-2  Worker-N
             │         │         │
             └─────────┼─────────┘
                       ▼
                    Queue/DLQ
```

---

# 8. Backend Architecture

## 8.1 Request flow

```text
HTTP Request
    ↓
Middleware
    ↓
Correlation ID
    ↓
Rate Limiter
    ↓
Authentication
    ↓
Authorization
    ↓
Router
    ↓
Controller
    ↓
Application Service
    ↓
Domain Rules
    ↓
Repository
    ↓
PostgreSQL / Redis
    ↓
Service Result
    ↓
Controller
    ↓
Response Schema
    ↓
HTTP Response
```

## 8.2 Layer responsibilities

### Router
Responsible for:

- URL mapping
- request parsing
- dependency injection
- authentication dependency
- response model declaration

### Controller
Responsible for:

- translating HTTP concerns into application commands
- invoking services
- mapping domain exceptions to API responses

### Service
Responsible for:

- business rules
- transactions
- orchestration
- authorization checks
- state transitions

### Repository
Responsible for:

- database queries
- persistence
- query composition
- transaction boundaries where appropriate

### Infrastructure
Responsible for:

- Redis
- Gmail
- LLM providers
- MCP
- object storage
- telemetry

---

# 9. Repository Structure

```text
agenthire/
├── apps/
│   ├── web/
│   │   ├── app/
│   │   ├── features/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── lib/
│   │   ├── providers/
│   │   └── types/
│   │
│   ├── api/
│   │   └── app/
│   │       ├── main.py
│   │       ├── core/
│   │       ├── config/
│   │       ├── database/
│   │       ├── modules/
│   │       │   ├── auth/
│   │       │   ├── users/
│   │       │   ├── resumes/
│   │       │   ├── jobs/
│   │       │   ├── matching/
│   │       │   ├── applications/
│   │       │   ├── drafts/
│   │       │   ├── inbox/
│   │       │   ├── dossiers/
│   │       │   ├── interviews/
│   │       │   └── audit/
│   │       ├── ai/
│   │       │   ├── graph/
│   │       │   ├── agents/
│   │       │   ├── prompts/
│   │       │   ├── evaluators/
│   │       │   ├── gateway/
│   │       │   └── state/
│   │       ├── mcp/
│   │       ├── workers/
│   │       ├── infrastructure/
│   │       └── shared/
│   │
│   └── gmail-mcp/
│       ├── server.py
│       ├── tools/
│       ├── auth/
│       ├── schemas/
│       └── security/
│
├── packages/
│   └── shared-types/
│
├── infrastructure/
│   ├── docker/
│   ├── terraform/
│   ├── aws/
│   └── monitoring/
│
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── contract/
│   ├── e2e/
│   ├── load/
│   ├── security/
│   └── failure/
│
├── evals/
│   ├── golden/
│   ├── datasets/
│   ├── runners/
│   └── reports/
│
├── docs/
│   ├── architecture/
│   ├── adr/
│   ├── api/
│   ├── security/
│   ├── runbooks/
│   ├── incidents/
│   └── postmortems/
│
├── .github/
│   └── workflows/
│
├── docker-compose.yml
├── Makefile
├── README.md
└── .env.example
```

---

# 10. Domain Modules

Every module follows:

```text
module/
├── router.py
├── controller.py
├── service.py
├── repository.py
├── schemas.py
├── models.py
├── exceptions.py
└── policies.py
```

AI-heavy modules additionally contain:

```text
agents/
prompts/
evaluators/
state/
```

---

# 11. Data Model

## 11.1 Core entities

### users

```sql
id UUID PRIMARY KEY
email VARCHAR UNIQUE NOT NULL
display_name VARCHAR
oauth_provider VARCHAR
oauth_subject VARCHAR
oauth_tokens_encrypted TEXT
oauth_expires_at TIMESTAMP
status VARCHAR
created_at TIMESTAMP
updated_at TIMESTAMP
```

### resumes

```sql
id UUID PRIMARY KEY
user_id UUID REFERENCES users(id)
version INT NOT NULL
parsed_json JSONB NOT NULL
skills_extracted TEXT[]
embedding VECTOR(768)
raw_file_url TEXT
file_hash VARCHAR
created_at TIMESTAMP
```

### job_postings

```sql
id UUID PRIMARY KEY
source_url TEXT UNIQUE NOT NULL
title VARCHAR NOT NULL
company VARCHAR NOT NULL
jd_text TEXT NOT NULL
skills_required TEXT[]
apply_type VARCHAR
recipient_email VARCHAR
embedding VECTOR(768)
content_hash VARCHAR
fetched_at TIMESTAMP
created_at TIMESTAMP
```

### applications

```sql
id UUID PRIMARY KEY
user_id UUID REFERENCES users(id)
resume_id UUID REFERENCES resumes(id)
job_id UUID REFERENCES job_postings(id)
status VARCHAR NOT NULL
outreach_mode VARCHAR
match_score NUMERIC(5,4)
match_breakdown_json JSONB
follow_up_count INT DEFAULT 0
last_contact_at TIMESTAMP
next_action_at TIMESTAMP
created_at TIMESTAMP
updated_at TIMESTAMP
```

### drafts

```sql
id UUID PRIMARY KEY
application_id UUID REFERENCES applications(id)
version INT NOT NULL
subject VARCHAR
body TEXT
is_follow_up BOOLEAN DEFAULT FALSE
content_hash VARCHAR
approved_at TIMESTAMP
approved_by UUID
created_at TIMESTAMP
```

### sends

```sql
id UUID PRIMARY KEY
application_id UUID REFERENCES applications(id)
draft_id UUID REFERENCES drafts(id)
idempotency_key VARCHAR UNIQUE NOT NULL
gmail_message_id VARCHAR
gmail_thread_id VARCHAR
provider_status VARCHAR
sent_at TIMESTAMP
created_at TIMESTAMP
```

### inbox_events

```sql
id UUID PRIMARY KEY
application_id UUID REFERENCES applications(id)
gmail_message_id VARCHAR
gmail_thread_id VARCHAR
detected_at TIMESTAMP
classification VARCHAR
confidence NUMERIC(5,4)
needs_human_review BOOLEAN
raw_snippet TEXT
created_at TIMESTAMP
```

### job_dossiers

```sql
id UUID PRIMARY KEY
application_id UUID REFERENCES applications(id)
research_json JSONB
source_urls JSONB
generated_at TIMESTAMP
expires_at TIMESTAMP
```

### interview_sessions

```sql
id UUID PRIMARY KEY
application_id UUID REFERENCES applications(id)
transcript JSONB
scorecard_json JSONB
duration_seconds INT
created_at TIMESTAMP
```

### audit_log

```sql
id UUID PRIMARY KEY
entity_type VARCHAR NOT NULL
entity_id UUID
action VARCHAR NOT NULL
actor VARCHAR NOT NULL
request_id VARCHAR
metadata_json JSONB
timestamp TIMESTAMP NOT NULL
```

---

# 12. Database Engineering

## 12.1 PostgreSQL responsibilities

PostgreSQL is the authoritative source for:

- users
- resumes
- jobs
- applications
- drafts
- sends
- inbox events
- dossiers
- interviews
- audit records
- LangGraph checkpoints

## 12.2 Index strategy

Initial indexes:

```sql
CREATE INDEX idx_applications_user_status
ON applications(user_id, status);

CREATE INDEX idx_applications_next_action
ON applications(next_action_at)
WHERE next_action_at IS NOT NULL;

CREATE INDEX idx_inbox_thread
ON inbox_events(gmail_thread_id);

CREATE INDEX idx_audit_entity
ON audit_log(entity_type, entity_id, timestamp DESC);

CREATE INDEX idx_jobs_company
ON job_postings(company);

CREATE INDEX idx_jobs_fetched_at
ON job_postings(fetched_at DESC);
```

Vector indexes should be introduced after measuring dataset size and query performance.

## 12.3 Transaction rules

Transactions are required for:

- approving a draft
- changing application status
- recording a send
- recording audit events associated with critical state transitions
- processing inbox events
- creating idempotency records

---

# 13. Application State Machine

```text
MATCHED
   ↓
DRAFTED
   ↓
APPROVAL_PENDING
   ├── REJECTED
   ├── EDITED → APPROVAL_PENDING
   └── APPROVED
          ↓
        QUEUED
          ↓
        SENDING
       /       \
   FAILED     SENT
                ↓
       AWAITING_RESPONSE
          ├── FOLLOW_UP_ELIGIBLE
          │        ↓
          │  FOLLOW_UP_DRAFTED
          │        ↓
          │  APPROVAL_PENDING
          │
          └── REPLIED
                 ├── REJECTED
                 ├── SHORTLISTED
                 └── INTERVIEW
                         ↓
                       DOSSIER
                         ↓
                      INTERVIEW
                         ↓
                       SCORECARD
```

Every transition must be validated against an explicit transition table.

Invalid transitions return a domain error rather than silently changing state.

---

# 14. LangGraph Agent Architecture

## 14.1 State

```python
class AgentHireState(TypedDict):
    application_id: str
    user_id: str
    resume: dict
    job: dict
    match_result: dict
    outreach_mode: str
    current_draft: dict
    approval_status: Literal[
        "pending",
        "approved",
        "rejected",
        "edited"
    ]
    send_result: Optional[dict]
    inbox_events: List[dict]
    follow_up_eligible: bool
    dossier: Optional[dict]
    interview_transcript: Optional[list]
    interview_scorecard: Optional[dict]
    errors: List[str]
```

## 14.2 Graph

```text
START
  ↓
parse_resume
  ↓
discover_jobs
  ↓
match_jobs
  ↓
draft_outreach
  ↓
await_approval
  ├──────────── rejected ──────────► END
  │
  ├──────────── edited ────────────► draft_outreach
  │
  └──────────── approved ──────────► send_outreach
                                       ↓
                                   await_response
                                       ↓
                                  classify_response
                                  /       |       \
                              reject  shortlist  interview
                                        ↓
                                   build_dossier
                                        ↓
                                  conduct_interview
                                        ↓
                                  evaluate_interview
                                        ↓
                                       END
```

---

# 15. Human-in-the-Loop Contract

The approval gate is a security boundary.

## 15.1 Approval invariant

```text
IF draft.approved_at IS NULL
THEN send operation MUST fail.
```

## 15.2 Approval workflow

1. Agent creates draft.
2. Draft is persisted.
3. Graph checkpoints state.
4. Workflow interrupts.
5. UI displays draft.
6. User may:
   - approve
   - edit
   - reject
   - regenerate
7. Edited drafts become a new version.
8. Only the explicitly approved version can be sent.
9. Send worker validates approval again.
10. Audit event is written.

The worker must never trust approval state supplied by the browser.

---

# 16. Gmail MCP Server

## 16.1 Tools

```text
send_email(
  to,
  subject,
  body,
  attachment_id,
  idempotency_key
)

search_inbox(
  label,
  thread_id,
  since_date
)

get_thread(
  thread_id
)
```

## 16.2 Security

- OAuth2
- least-privilege scopes
- encrypted credentials
- token refresh
- request validation
- audit logging
- explicit tool allowlist
- no arbitrary Gmail API passthrough

## 16.3 Tool authorization

```text
Agent
  ↓
Tool Request
  ↓
Permission Check
  ↓
User Scope Check
  ↓
Rate Limit
  ↓
Idempotency Check
  ↓
MCP Tool
  ↓
Gmail
```

---

# 17. Idempotency

Every external side effect uses an idempotency key.

Recommended send key:

```text
SHA256(
    user_id +
    application_id +
    draft_id +
    draft_version
)
```

Before sending:

```text
Does idempotency_key exist?
       │
   ┌───┴────┐
   │        │
  YES      NO
   │        │
return     reserve
existing      │
result        ▼
            send
              │
              ▼
          persist result
```

If the worker crashes after Gmail accepts the request but before PostgreSQL records success, the same idempotency key must prevent a duplicate send.

---

# 18. Redis Design

Redis is used for:

### Cache
- job metadata
- company research
- repeated match calculations
- session/cache data where appropriate

### Rate limiting
- API requests
- LLM requests
- email sends
- tool calls

### Distributed locks
- inbox watcher singleton
- duplicate scheduler execution
- user-level agent execution

### Queue
- resume parsing
- job ingestion
- matching
- drafting
- email sending
- inbox processing
- dossier generation
- evaluation

### Pub/Sub / Events
- application status updates
- UI notifications
- worker events

Redis data must always be considered reconstructable.

---

# 19. Queue Architecture

```text
API
 │
 ├── resume.parse
 ├── job.ingest
 ├── match.compute
 ├── outreach.draft
 ├── email.send
 ├── inbox.scan
 ├── dossier.build
 └── interview.evaluate
          │
          ▼
       Queue
          │
    ┌─────┼─────┐
    ▼     ▼     ▼
 Worker Worker Worker
    │     │     │
    └─────┼─────┘
          ▼
        Result
```

## 19.1 Retry policy

Example:

```text
Attempt 1 → immediate
Attempt 2 → +2 sec
Attempt 3 → +8 sec
Attempt 4 → +30 sec
Attempt 5 → +2 min
Then → DLQ
```

Use jitter to avoid synchronized retries.

---

# 20. Dead-Letter Queue

Messages enter DLQ when:

- maximum retries are exhausted
- schema is invalid
- external dependency repeatedly fails
- authorization is invalid
- agent state is corrupt
- human intervention is required

DLQ workflow:

```text
DLQ
 ↓
Alert
 ↓
Inspect payload
 ↓
Classify root cause
 ├── transient → replay
 ├── bug → fix → replay
 ├── invalid input → discard/repair
 └── security issue → quarantine
```

---

# 21. AI Gateway

All model calls should pass through an internal AI gateway.

Responsibilities:

- provider abstraction
- model routing
- timeout
- retries
- fallback
- token budget
- cost accounting
- structured output validation
- prompt versioning
- telemetry
- request correlation

Example:

```text
Application
    ↓
AI Gateway
    ↓
Policy
    ├── token budget
    ├── model selection
    ├── timeout
    └── safety
    ↓
Primary Model
    │
    └── failure
          ↓
       Fallback
```

---

# 22. AI Model Strategy

Use the project PRD's intended model family:

- Gemini 2.5 Flash for general generation
- Gemini 2.5 Flash-Lite for cheaper classification workloads
- `text-embedding-004` for embeddings
- Groq Whisper for speech-to-text fallback
- Web Speech API for browser voice interaction

Models must be configurable rather than hardcoded into business logic.

---

# 23. RAG / Matching Architecture

## 23.1 Hybrid matching

```text
Resume
  │
  ├── Structured skills
  │
  └── Embedding
         │
         ▼
Job
  │
  ├── Required skills
  │
  └── Embedding
         │
         ▼
 ┌─────────────────────┐
 │ Hybrid Ranker       │
 │                     │
 │ Semantic similarity │
 │ + skill overlap     │
 │ + deterministic     │
 │   constraints       │
 └──────────┬──────────┘
            ▼
      Explainable Match
```

## 23.2 Match output

```json
{
  "score": 0.84,
  "matched_skills": [
    "Python",
    "FastAPI",
    "PostgreSQL"
  ],
  "missing_skills": [
    "Kubernetes"
  ],
  "evidence": [
    "FastAPI appears in project experience",
    "PostgreSQL appears in backend project"
  ],
  "recommendation": "Strong match"
}
```

The LLM should not invent resume experience.

---

# 24. Prompt Injection Defense

All external text is wrapped as untrusted data.

Example:

```text
SYSTEM INSTRUCTIONS
↓
APPLICATION POLICY
↓
USER-OWNED DATA
↓
<UNTRUSTED_JOB_DESCRIPTION>
...
</UNTRUSTED_JOB_DESCRIPTION>
↓
MODEL
```

Never allow job descriptions or recruiter emails to redefine:

- system instructions
- tools
- permissions
- approval policy
- identity
- secrets
- application state

Tool calls require explicit server-side authorization regardless of model output.

---

# 25. API Standards

## 25.1 Versioning

```text
/api/v1/...
```

## 25.2 Response envelope

Successful responses:

```json
{
  "data": {},
  "request_id": "req_01J..."
}
```

Errors:

```json
{
  "error": {
    "code": "DRAFT_NOT_APPROVED",
    "message": "The selected draft has not been approved.",
    "request_id": "req_01J..."
  }
}
```

## 25.3 Pagination

Use cursor pagination for high-growth collections.

```json
{
  "data": [],
  "pagination": {
    "next_cursor": "...",
    "has_more": true
  }
}
```

---

# 26. Representative API Contract

## Resume

```text
POST   /api/v1/resumes
GET    /api/v1/resumes
GET    /api/v1/resumes/{id}
DELETE /api/v1/resumes/{id}
```

## Jobs

```text
POST /api/v1/jobs/discover
GET  /api/v1/jobs
GET  /api/v1/jobs/{id}
```

## Applications

```text
POST /api/v1/applications
GET  /api/v1/applications
GET  /api/v1/applications/{id}
POST /api/v1/applications/{id}/approve
POST /api/v1/applications/{id}/reject
POST /api/v1/applications/{id}/regenerate
```

## Inbox

```text
POST /api/v1/cron/inbox-watcher
GET  /api/v1/applications/{id}/messages
```

## Interview

```text
POST /api/v1/applications/{id}/dossier
POST /api/v1/applications/{id}/interview
GET  /api/v1/interviews/{id}
```

---

# 27. Authentication and Authorization

## Authentication

Google OAuth2 is the primary identity mechanism.

## Authorization

Every request resolves:

```text
identity → user_id → resource ownership → policy
```

Example:

```text
GET /applications/abc
        ↓
Authenticated user = U1
        ↓
Application owner = U2
        ↓
DENY 403
```

Never rely on frontend filtering for authorization.

---

# 28. Security Model

## 28.1 Security controls

- HTTPS everywhere
- secure cookies
- OAuth token encryption
- secret management
- least privilege
- RBAC-ready policy layer
- rate limiting
- request validation
- file type validation
- file size limits
- malware scanning strategy
- SQL parameterization
- output encoding
- CSRF protection where applicable
- CORS allowlist
- audit logging
- prompt injection defenses
- tool permission checks
- dependency scanning
- container image scanning

## 28.2 Threat model

| Threat | Mitigation |
|---|---|
| Prompt injection | Treat external text as untrusted |
| Duplicate email | Idempotency keys |
| OAuth theft | Encryption + secret isolation |
| Unauthorized application access | Resource ownership checks |
| API abuse | Rate limiting |
| Agent runaway | Token/tool/iteration budgets |
| Spam reputation | Approval gate + 15/day limit |
| Worker duplication | Locks + idempotency |
| DB corruption | Transactions + backups |
| Dependency outage | Retry/fallback/circuit breaker |
| Malicious upload | File validation/scanning |
| Secret leakage | Secret manager + log redaction |

---

# 29. Email Safety Policy

Hard limits:

```text
MAX_EMAILS_PER_USER_PER_DAY = 15
MAX_FOLLOW_UPS_PER_APPLICATION = 1
APPROVAL_REQUIRED = true
```

Send pipeline:

```text
Approved?
   ↓ yes
Daily quota available?
   ↓ yes
Idempotency key unused?
   ↓ yes
OAuth valid?
   ↓ yes
Recipient valid?
   ↓ yes
Draft version still current?
   ↓ yes
Send
```

Any failed precondition blocks the send.

---

# 30. Scheduler

Scheduled workflows:

- inbox watcher
- token health check
- follow-up eligibility
- stale job cleanup
- expired dossier refresh
- evaluation jobs
- health checks

Scheduler must be externally triggerable so API instances do not depend on in-process cron.

Use QStash/GitHub Actions initially.

For distributed execution:

```text
Scheduler
   ↓
POST /internal/jobs/inbox-scan
   ↓
Authentication
   ↓
Distributed lock
   ↓
Queue
   ↓
Worker
```

---

# 31. Inbox Watcher

The watcher runs approximately twice per day.

Algorithm:

```text
Find applications awaiting response
        ↓
Fetch Gmail threads
        ↓
Deduplicate messages
        ↓
Persist inbox event
        ↓
Classify response
        ↓
confidence < 0.80?
      /       \
    yes        no
    ↓           ↓
human review  update state
                 ↓
          no response for
          5 business days?
             /       \
           yes        no
           ↓           ↓
      draft follow-up END
```

No automatic follow-up is sent.

---

# 32. Response Classification

Classes:

```text
REJECTED
SHORTLISTED
INTERVIEW_INVITE
NEEDS_RESPONSE
INFORMATIONAL
UNKNOWN
```

Confidence threshold:

```text
confidence < 0.80
→ needs_human_review = true
```

The classifier must not silently make high-impact state changes when uncertain.

---

# 33. Job Dossier

A dossier contains:

```json
{
  "company": {},
  "role": {},
  "job_requirements": [],
  "company_context": {},
  "likely_interview_topics": [],
  "resume_alignment": [],
  "potential_gaps": [],
  "questions_to_prepare": [],
  "sources": []
}
```

Research output must preserve source URLs and timestamps.

---

# 34. Voice Interview

Browser:

```text
Microphone
   ↓
Web Speech API
   ↓
Interview UI
   ↓
FastAPI
   ↓
LangGraph Interview State
   ↓
Question Generation
   ↓
Answer
   ↓
Transcript
   ↓
STAR Evaluation
```

Fallback:

```text
Browser audio
   ↓
Groq Whisper
   ↓
Transcript
```

---

# 35. STAR Evaluation

Score:

```text
Situation
Task
Action
Result
```

Additional dimensions:

- relevance
- clarity
- specificity
- technical depth
- communication
- evidence
- improvement areas

Example:

```json
{
  "overall_score": 7.8,
  "star": {
    "situation": 8,
    "task": 7,
    "action": 8,
    "result": 7
  },
  "strengths": [],
  "improvements": []
}
```

---

# 36. Observability

Every request receives:

```text
request_id
trace_id
user_id (redacted/hashed in telemetry where possible)
service
operation
duration
status
```

## 36.1 Logs

Use structured JSON logs.

Example:

```json
{
  "timestamp": "2026-09-07T15:30:00Z",
  "level": "INFO",
  "service": "api",
  "operation": "approve_draft",
  "request_id": "req_123",
  "application_id": "app_123",
  "duration_ms": 84,
  "status": "success"
}
```

Never log:

- OAuth tokens
- secrets
- full email bodies
- sensitive credentials

## 36.2 Metrics

Track:

### API
- request count
- error rate
- p50/p95/p99 latency
- throughput

### Workers
- queue depth
- processing time
- retry count
- DLQ count

### Database
- connection utilization
- query latency
- slow queries
- transaction conflicts

### Redis
- latency
- memory
- hit rate
- queue backlog

### AI
- request count
- latency
- tokens
- cost
- failures
- fallback rate
- evaluation scores

### Gmail
- send attempts
- successful sends
- failures
- rate-limit responses

---

# 37. SLOs

Initial target SLOs:

| Service | Target |
|---|---:|
| API availability | 99.5% |
| Read API p95 | < 500 ms |
| Write API p95 | < 800 ms |
| Queue enqueue p95 | < 300 ms |
| Worker success rate | > 99% |
| AI request success after retry | > 99% |
| Duplicate email rate | 0 |
| Unauthorized send rate | 0 |
| Lost application state | 0 |

These are engineering targets for the project, not claims of externally measured production performance.

---

# 38. Health Endpoints

```text
GET /health/live
GET /health/ready
GET /health/dependencies
```

### Liveness

Checks process health.

### Readiness

Checks whether the instance can serve traffic.

### Dependencies

Reports:

- PostgreSQL
- Redis
- external AI provider
- Gmail integration where applicable

A dependency failure should not necessarily make the entire service unavailable.

---

# 39. Graceful Degradation

Examples:

### Redis unavailable
- API remains usable for database-backed operations
- cache disabled
- queue-backed actions may be blocked or persisted for retry

### LLM unavailable
- retries
- fallback model
- queued processing
- user-visible status

### Gmail unavailable
- application remains viewable
- send remains queued/failed
- no duplicate retry without idempotency

### Research provider unavailable
- dossier marked incomplete
- application pipeline remains functional

---

# 40. Circuit Breakers

External dependencies should have:

```text
CLOSED
  ↓ failures exceed threshold
OPEN
  ↓ cooldown
HALF_OPEN
  ↓ successful probe
CLOSED
```

Apply to:

- LLM providers
- Gmail API
- research providers
- external job APIs

---

# 41. Frontend Architecture

```text
Next.js App Router
        ↓
Feature Modules
        ↓
Custom Hooks
        ↓
TanStack Query
        ↓
API Client
        ↓
FastAPI
```

Example:

```text
useApproveDraft()
      ↓
api.applications.approve()
      ↓
POST /api/v1/applications/{id}/approve
      ↓
invalidate application query
      ↓
refresh UI
```

Use optimistic updates only when rollback semantics are well-defined.

---

# 42. Frontend State Strategy

### Server state
TanStack Query:

- applications
- jobs
- resumes
- drafts
- inbox
- dossiers
- interviews

### Client state

Use local state for:

- modal state
- draft editor
- filters
- UI preferences
- microphone state

Do not duplicate server state into global client stores without a clear reason.

---

# 43. Caching

Cache candidates:

- job metadata
- research results
- repeated embeddings
- static configuration
- expensive read operations

Avoid caching:

- approval decisions
- authoritative application state
- security permissions
- send results

Cache invalidation must be tied to domain events.

---

# 44. Concurrency Control

Potential race:

```text
Worker A → approve
Worker B → regenerate
Worker C → send
```

Prevent using:

- database transactions
- row/version checks
- optimistic concurrency
- distributed locks where needed
- idempotency keys

Draft version must be verified at send time.

---

# 45. Failure Scenarios

## Gmail 429

```text
429
 ↓
retry-after / backoff
 ↓
retry
 ↓
failure threshold
 ↓
DLQ
 ↓
human/operator review
```

## Worker crash

```text
Worker crashes
 ↓
message remains/reappears
 ↓
retry
 ↓
idempotency prevents duplicate side effect
```

## Database outage

```text
DB unavailable
 ↓
readiness fails
 ↓
traffic drains
 ↓
alerts
 ↓
DB recovery
 ↓
readiness restored
```

## Agent stuck

Controls:

- maximum graph steps
- maximum tool calls
- token budget
- wall-clock timeout
- retry budget
- human escalation

## Duplicate scheduler invocation

Use distributed lock:

```text
lock:agenthire:cron:inbox-watcher
```

---

# 46. Testing Strategy

## 46.1 Test pyramid

```text
             E2E
          /-------\
        Contract
       /-----------\
    Integration
   /---------------\
       Unit Tests
```

## Unit tests

Test:

- match scoring
- state transitions
- permission policies
- idempotency generation
- follow-up eligibility
- retry policy
- parsers
- domain services

## Integration tests

Test:

- PostgreSQL repositories
- Redis
- queue workers
- Gmail adapter
- MCP server
- OAuth flows

## Contract tests

Validate:

- API request schemas
- API response schemas
- MCP tool schemas

## E2E

Critical journey:

```text
Login
→ Upload Resume
→ Discover Job
→ Match
→ Generate Draft
→ Approve
→ Queue
→ Send
→ Detect Reply
→ Classify
→ Dossier
→ Interview
→ Scorecard
```

---

# 47. AI Evaluation

Maintain a golden dataset of resume/job pairs.

Example dataset fields:

```json
{
  "resume_id": "golden-01",
  "job_id": "golden-job-01",
  "expected_skills": [],
  "expected_missing_skills": [],
  "acceptable_score_range": [0.70, 0.90],
  "draft_factuality_requirements": []
}
```

Metrics:

- faithfulness
- relevance
- retrieval quality
- classification accuracy
- structured-output validity
- hallucination rate
- latency
- token consumption
- cost

Ragas is used for RAG-oriented evaluation.

---

# 48. AI Quality Gates

A CI evaluation should fail when:

```text
structured_output_validity < 99%
OR
critical factuality < threshold
OR
classifier accuracy < threshold
OR
retrieval recall < threshold
OR
unexpected tool calls > 0
```

Thresholds should be version-controlled and changed through an ADR.

---

# 49. Prompt Management

Every production prompt has:

```text
prompt_id
version
owner
created_at
model
temperature/config
input schema
output schema
evaluation dataset
```

Example:

```text
outreach-drafter:v3
response-classifier:v2
dossier-builder:v4
interview-evaluator:v2
```

Do not silently change production prompts.

---

# 50. Cost Engineering

Track:

```text
LLM tokens
LLM requests
embedding requests
voice transcription duration
database usage
object storage
network
compute
observability
```

Example budget model:

```text
monthly_cost =
  compute
+ database
+ redis
+ storage
+ network
+ AI
+ observability
```

The original MVP targets free-tier infrastructure; production scaling should explicitly document where free-tier assumptions stop being valid.

---

# 51. CI/CD

Pipeline:

```text
Pull Request
   ↓
Lint
   ↓
Type Check
   ↓
Unit Tests
   ↓
Integration Tests
   ↓
Security Scan
   ↓
AI Evaluation
   ↓
Build Docker Image
   ↓
Push Artifact
   ↓
Deploy Staging
   ↓
Smoke Tests
   ↓
Approval
   ↓
Production
   ↓
Health Check
   ↓
Monitor
```

---

# 52. Deployment Strategy

Initial strategy:

- staging environment
- production environment
- immutable Docker images
- health checks
- migrations before compatible application rollout
- rollback procedure

Later:

- rolling deployments
- blue/green
- canary
- feature flags

Never perform destructive database migrations without a rollback or recovery plan.

---

# 53. Database Migration Strategy

Use expand/contract.

### Expand

Add new nullable column/table.

### Deploy

Application supports old + new schema.

### Backfill

Populate data safely.

### Switch

Application begins using new field.

### Contract

Remove old field only after compatibility window.

---

# 54. Backup and Disaster Recovery

Production database policy:

- automated backups
- point-in-time recovery where supported
- restore testing
- backup monitoring

Targets:

```text
RPO: 1 hour
RTO: 4 hours
```

These are initial engineering targets.

A backup is not considered reliable until a restore has been tested.

---

# 55. Disaster Recovery Runbook

```text
Incident detected
      ↓
Declare incident
      ↓
Identify affected dependency
      ↓
Protect user data
      ↓
Enable graceful degradation
      ↓
Restore dependency
      ↓
Verify integrity
      ↓
Replay safe jobs
      ↓
Run smoke tests
      ↓
Restore normal traffic
      ↓
Postmortem
```

---

# 56. Infrastructure as Code

Terraform modules:

```text
terraform/
├── environments/
│   ├── staging/
│   └── production/
├── modules/
│   ├── networking/
│   ├── compute/
│   ├── database/
│   ├── redis/
│   ├── storage/
│   ├── monitoring/
│   └── iam/
└── variables.tf
```

Infrastructure must be reproducible.

---

# 57. Docker

Services:

```text
web
api
worker
scheduler
gmail-mcp
```

Use:

- multi-stage builds
- non-root containers
- minimal base images
- pinned dependencies
- health checks
- read-only filesystem where practical
- environment-based configuration

---

# 58. Environment Strategy

```text
.env.local
.env.test
staging secrets
production secrets
```

Never commit:

- OAuth secrets
- API keys
- database passwords
- encryption keys
- private credentials

---

# 59. Configuration

Centralized typed settings:

```text
APP_ENV
DATABASE_URL
REDIS_URL
GOOGLE_CLIENT_ID
GOOGLE_CLIENT_SECRET
OAUTH_ENCRYPTION_KEY
GEMINI_API_KEY
GROQ_API_KEY
LANGFUSE_PUBLIC_KEY
LANGFUSE_SECRET_KEY
```

Configuration must fail fast when mandatory production secrets are missing.

---

# 60. Rate Limiting

Layers:

```text
CDN/WAF
   ↓
API rate limiter
   ↓
User-level limiter
   ↓
Endpoint limiter
   ↓
Provider limiter
```

Examples:

```text
POST /resume/upload
POST /jobs/discover
POST /draft/regenerate
POST /application/approve
POST /interview
```

Expensive endpoints have stricter limits.

---

# 61. Capacity Model

Initial design assumptions:

```text
Users:                 1–100
Active users/day:      20
Applications/user/day: 10
Emails/user/day:       15 hard cap
Jobs processed/day:    2,000
Inbox scans/day:       200
AI operations/day:     1,000
```

Illustrative peak API load:

```text
20 active users
× 10 operations/minute
= 200 operations/minute
≈ 3.3 requests/second
```

A small API deployment can support this workload; actual limits must be established through load testing.

---

# 62. 100× Scaling Plan

If workload increases 100×:

### API
Scale horizontally.

### Workers
Increase worker count based on queue depth.

### PostgreSQL
- optimize indexes
- connection pooling
- read replicas
- partition large event/audit tables if necessary

### Redis
- increase capacity
- isolate queues/cache
- Redis Streams or dedicated queue infrastructure

### AI
- provider routing
- batching
- caching
- queue-based admission control
- multiple model providers

### Gmail
External provider quotas become a hard dependency; throughput cannot simply be increased by adding servers.

### Architecture evolution

```text
Modular Monolith
      ↓
Async Workers
      ↓
Measured bottleneck
      ↓
Extract only bottleneck
      ↓
Service
```

Potential future services:

```text
Job Ingestion Service
AI Inference Service
Email Service
Research Service
Interview Service
```

---

# 63. Distributed Systems Principles

The system explicitly demonstrates:

- idempotency
- retries
- exponential backoff
- jitter
- timeouts
- circuit breakers
- distributed locks
- eventual consistency
- durable queues
- dead-letter queues
- replay
- backpressure
- graceful degradation
- optimistic concurrency
- auditability

---

# 64. Event Model

Potential domain events:

```text
ResumeUploaded
JobMatched
DraftCreated
DraftApproved
DraftRejected
EmailQueued
EmailSent
InboxMessageDetected
ResponseClassified
FollowUpDrafted
ApplicationShortlisted
DossierCreated
InterviewCompleted
ScorecardGenerated
```

Example:

```json
{
  "event_id": "evt_123",
  "event_type": "DraftApproved",
  "aggregate_id": "app_123",
  "user_id": "user_123",
  "version": 4,
  "occurred_at": "2026-09-07T15:30:00Z",
  "metadata": {}
}
```

---

# 65. Outbox Pattern

For critical state transitions:

```text
BEGIN TRANSACTION
    update application
    insert outbox event
COMMIT
        ↓
Outbox publisher
        ↓
Queue/Event Bus
```

This prevents the failure mode:

```text
DB updated
but event lost
```

---

# 66. Auditability

Every high-impact action records:

```text
who
what
when
which resource
which version
request ID
agent/tool
result
```

Examples:

```text
USER approved draft v3
AGENT generated draft v3
AGENT classified reply as SHORTLISTED
MCP send_email invoked
SYSTEM blocked send due to rate limit
```

---

# 67. Operational Dashboard

Dashboard panels:

### API
- traffic
- latency
- errors

### Workers
- queue depth
- retries
- DLQ

### Database
- CPU
- connections
- slow queries

### AI
- model usage
- token cost
- latency
- error rate
- fallback rate

### Product
- jobs discovered
- drafts created
- approvals
- sends
- replies
- interviews

### Reliability
- SLO compliance
- incidents
- dependency health

---

# 68. Alerts

Critical alerts:

```text
API error rate > 5%
API p95 latency > 2s
DB connection saturation > 80%
Queue backlog > threshold
DLQ messages > 0
AI failure rate > 10%
Gmail 429 spike
Unauthorized send attempt
Duplicate-send detection
OAuth token health failure
```

Alert thresholds must be tuned using real production telemetry.

---

# 69. Runbooks

Required runbooks:

```text
runbooks/
├── api-outage.md
├── database-outage.md
├── redis-outage.md
├── gmail-failure.md
├── llm-provider-outage.md
├── queue-backlog.md
├── dlq-replay.md
├── oauth-expiry.md
├── duplicate-send.md
├── deployment-rollback.md
└── data-restore.md
```

Every runbook contains:

1. symptoms
2. impact
3. diagnosis
4. mitigation
5. recovery
6. verification
7. escalation
8. post-incident actions

---

# 70. ADR Index

Required ADRs:

```text
ADR-001 Modular Monolith
ADR-002 PostgreSQL as Source of Truth
ADR-003 Redis for Cache and Queue
ADR-004 LangGraph for Durable Agent Workflows
ADR-005 Human Approval Before Email Send
ADR-006 MCP for Gmail Tool Boundary
ADR-007 PostgreSQL pgvector for Matching
ADR-008 Async Workers for Long Operations
ADR-009 External Scheduler
ADR-010 AI Gateway
ADR-011 Idempotent Email Sending
ADR-012 Hybrid Semantic + Deterministic Matching
ADR-013 API Versioning
ADR-014 Expand/Contract Database Migrations
ADR-015 Observability Strategy
ADR-016 Free-Tier MVP Architecture
ADR-017 AWS Production Evolution
```

Each ADR contains:

```text
Context
Decision
Alternatives
Trade-offs
Consequences
Migration/rollback
```

---

# 71. Performance Engineering

Measure before optimizing.

Primary targets:

- API p95 latency
- DB query latency
- queue wait time
- worker processing time
- embedding latency
- LLM latency
- Gmail API latency
- frontend LCP
- bundle size

Use profiling to identify actual bottlenecks.

---

# 72. Frontend Performance

Targets:

- route-level code splitting
- server rendering where useful
- image optimization
- streaming where useful
- TanStack Query caching
- pagination
- optimistic UI only where safe
- minimal client components
- Web Worker for CPU-heavy browser work where justified

---

# 73. Database Performance

Required engineering exercises:

```text
EXPLAIN
EXPLAIN ANALYZE
index selectivity
connection pooling
slow query logs
transaction isolation
deadlocks
N+1 detection
pagination performance
vector search performance
```

Every non-trivial query should have a reason for its index.

---

# 74. Security Testing

Include:

- dependency audit
- SAST
- secret scanning
- container scanning
- authentication tests
- authorization tests
- rate-limit tests
- prompt injection tests
- MCP permission tests
- malicious file tests
- SSRF considerations for URL ingestion
- input validation tests

---

# 75. Load Testing

Scenarios:

### API baseline
```text
10 RPS
```

### Moderate
```text
50 RPS
```

### Stress
```text
100 RPS
```

### Worker load
```text
1,000 queued jobs
```

Measure:

- throughput
- p50
- p95
- p99
- error rate
- queue delay
- DB saturation
- memory
- CPU

Do not claim these are production capabilities until the tests have actually been run.

---

# 76. Failure Testing

Inject:

- DB latency
- Redis unavailable
- Gmail timeout
- Gmail 429
- LLM timeout
- LLM 5xx
- worker crash
- queue duplication
- malformed job description
- malformed email
- expired OAuth token
- duplicate scheduler execution

Success means:

```text
Failure detected
→ system remains safe
→ no unauthorized side effect
→ state remains recoverable
→ operator/user can understand what happened
```

---

# 77. Production Readiness Checklist

## Product

- [ ] Resume upload works
- [ ] Job discovery works
- [ ] Matching is explainable
- [ ] Draft generation works
- [ ] Approval gate works
- [ ] Gmail send works
- [ ] Inbox watcher works
- [ ] Follow-up drafting works
- [ ] Response classification works
- [ ] Dossier works
- [ ] Voice interview works
- [ ] STAR scorecard works

## Backend

- [ ] Layered architecture
- [ ] Domain state machine
- [ ] Transactions
- [ ] Idempotency
- [ ] Rate limiting
- [ ] Retries
- [ ] Timeouts
- [ ] Circuit breakers
- [ ] Queue
- [ ] DLQ
- [ ] Distributed locks

## AI

- [ ] Structured outputs
- [ ] Prompt versioning
- [ ] Token budgets
- [ ] Model fallback
- [ ] Tool authorization
- [ ] Prompt injection defenses
- [ ] Golden dataset
- [ ] Ragas evaluation
- [ ] Langfuse tracing

## Security

- [ ] OAuth
- [ ] Secret management
- [ ] Least privilege
- [ ] Authorization
- [ ] Audit logs
- [ ] Security scans
- [ ] File validation

## Operations

- [ ] Docker
- [ ] CI/CD
- [ ] Staging
- [ ] Production
- [ ] Health checks
- [ ] Logs
- [ ] Metrics
- [ ] Traces
- [ ] Alerts
- [ ] Runbooks
- [ ] Backup
- [ ] Restore test
- [ ] Rollback procedure

---

# 78. Engineering Definition of Done

A feature is **not done** when the happy path works.

A production feature is done when:

```text
Requirement
    ↓
Acceptance Criteria
    ↓
Architecture
    ↓
Data Model
    ↓
API Contract
    ↓
Implementation
    ↓
Unit Tests
    ↓
Integration Tests
    ↓
Security Review
    ↓
Observability
    ↓
Failure Handling
    ↓
Documentation
    ↓
CI/CD
    ↓
Staging Validation
    ↓
Production Deployment
    ↓
Monitoring
```

---

# 79. Senior-Level Evidence to Collect

The project should produce measurable artifacts, not only source code.

Collect:

```text
Architecture diagram
C4 diagrams
Sequence diagrams
Data-flow diagrams
Deployment diagram
ERD
API specification
OpenAPI document
ADR collection
Threat model
Capacity estimate
Load-test report
Failure-test report
AI evaluation report
Cost report
Observability dashboard
CI/CD screenshots
Deployment history
Incident report
Postmortem
Runbooks
```

---

# 80. Postmortem Template

```markdown
# Incident: <name>

## Summary

## Impact

## Timeline

## Detection

## Root Cause

## Contributing Factors

## What Went Well

## What Went Poorly

## Immediate Fix

## Permanent Fix

## Prevention

## Action Items

| Action | Owner | Priority | Due |
|---|---|---|---|
| | | | |
```

---

# 81. Example Incident Scenarios

## Incident A — Gmail Duplicate Send Risk

### Detection
Idempotency conflict detected.

### Root cause
Worker retried after provider timeout.

### Resolution
Idempotency record prevented second send.

### Lesson
Timeout does not mean provider did not process the request.

---

## Incident B — LLM Provider Outage

### Detection
AI gateway error rate exceeds threshold.

### Resolution
Circuit breaker opens and fallback model is used.

### Product behavior
Draft generation becomes slower but application tracking remains available.

---

## Incident C — Queue Backlog

### Detection
Queue depth exceeds SLO.

### Response

```text
Inspect worker saturation
→ increase worker concurrency
→ inspect dependency latency
→ apply backpressure if necessary
→ drain queue
```

---

# 82. Architecture Evolution

## Stage 1 — Local

```text
Next.js
FastAPI
Postgres
Redis
Worker
MCP
```

## Stage 2 — Production MVP

```text
CDN
Next.js
FastAPI × N
Postgres
Redis
Workers × N
Scheduler
MCP
Observability
```

## Stage 3 — Growth

```text
Load Balancer
API cluster
Worker pools
Read replica
Dedicated queue
Object storage
AI gateway
```

## Stage 4 — Service extraction

Only extract a service when one of these is demonstrated:

- independent scaling requirement
- deployment isolation
- reliability isolation
- security boundary
- team ownership boundary
- resource profile incompatibility

---

# 83. Technology Map

| Concern | Technology |
|---|---|
| Frontend | React + Next.js + TypeScript |
| UI | Tailwind CSS |
| Server state | TanStack Query |
| Backend | Python + FastAPI |
| Validation | Pydantic |
| ORM | SQLAlchemy |
| Migrations | Alembic |
| Database | PostgreSQL |
| Vector | pgvector |
| Cache | Redis |
| Queue | Redis-backed worker architecture |
| Scheduler | QStash / GitHub Actions |
| AI | Gemini |
| Embeddings | text-embedding-004 |
| Orchestration | LangGraph |
| Agent framework concepts | LangChain |
| Tool protocol | MCP |
| Email | Gmail API |
| Voice | Web Speech API |
| STT fallback | Groq Whisper |
| Evaluation | Ragas |
| Tracing | Langfuse |
| Containers | Docker |
| Cloud | AWS |
| IaC | Terraform |
| CI/CD | GitHub Actions |
| Observability | OpenTelemetry / Prometheus / Grafana concepts |
| Testing | pytest / Playwright |
| Security | OAuth2 / least privilege / OWASP practices |

---

# 84. Engineering Trade-offs

## Modular monolith vs microservices

**Decision:** modular monolith.

Reason:

- solo developer
- lower operational complexity
- easier transactions
- faster development
- strong domain boundaries
- can be split later

## PostgreSQL vs separate vector database

**Decision:** PostgreSQL + pgvector.

Reason:

- one source of truth
- simpler deployment
- transactional metadata + vector retrieval
- sufficient for initial workload

## Redis queue vs Kafka

**Decision:** Redis-backed queue initially.

Reason:

- workload is moderate
- simpler operations
- lower cost
- Kafka would add operational overhead without current need

## Human approval vs full autonomy

**Decision:** human approval.

Reason:

- protects sender reputation
- prevents hallucinated outreach
- makes side effects auditable
- aligns with product trust model

---

# 85. Senior Interview Questions This Project Must Answer

Be able to explain:

1. Why modular monolith?
2. Why PostgreSQL?
3. Why pgvector?
4. Why Redis?
5. Why a queue?
6. Why LangGraph?
7. Why MCP instead of direct Gmail calls from the agent?
8. How does HITL work?
9. How do you prevent duplicate emails?
10. What happens after a worker crashes?
11. What happens after Gmail returns 429?
12. What happens if Gmail accepts the email but your DB write fails?
13. What happens if PostgreSQL is down?
14. What happens if Redis is down?
15. How do you scale API instances?
16. How do you scale workers?
17. What becomes the bottleneck at 100×?
18. How do you protect against prompt injection?
19. How do you authorize tool calls?
20. How do you evaluate agent quality?
21. How do you detect hallucinated resume claims?
22. How do you control LLM cost?
23. How do you roll back a deployment?
24. How do you migrate the database without downtime?
25. How do you restore from backup?
26. What are your SLOs?
27. How do you detect incidents?
28. How do you replay failed jobs?
29. Why not Kubernetes immediately?
30. When would you extract a microservice?

---

# 86. Senior Engineering Operating Loop

For every significant decision:

```text
What problem are we solving?
        ↓
What are the constraints?
        ↓
What scale are we designing for?
        ↓
What can fail?
        ↓
How do we detect failure?
        ↓
How do we recover?
        ↓
What is the simplest reliable solution?
        ↓
What are the trade-offs?
        ↓
How much does it cost?
        ↓
How do we secure it?
        ↓
How do we test it?
        ↓
How do we deploy it?
        ↓
How do we roll it back?
        ↓
How do we operate it six months later?
```

---

# 87. Final Project Positioning

Do **not** describe AgentHire as:

> "An AI job application website."

Describe it as:

> **"A production-oriented agentic job-search platform built around a modular monolith and asynchronous worker architecture, using durable LangGraph workflows, human-in-the-loop approval, MCP-based Gmail tool integration, PostgreSQL/pgvector, Redis, hybrid retrieval, AI evaluation, idempotent external side effects, observability, security guardrails, CI/CD, and cloud deployment."**

The project demonstrates the progression:

```text
Programming
   ↓
Full Stack
   ↓
Backend Engineering
   ↓
Database Engineering
   ↓
Distributed Systems
   ↓
System Design
   ↓
AI / RAG
   ↓
Agentic AI
   ↓
MCP
   ↓
AI Production Engineering
   ↓
Cloud / DevOps
   ↓
Observability / SRE
   ↓
Technical Leadership
```

---

# 88. Final Standard

AgentHire should be judged by whether its author can independently demonstrate:

```text
Requirement
→ Constraints
→ Capacity Estimate
→ Architecture
→ Data Model
→ API Contract
→ Implementation
→ Tests
→ Security
→ AI Evaluation
→ Observability
→ CI/CD
→ Cloud
→ Load Test
→ Failure Test
→ Production
→ Postmortem
```

That is the bar for treating the project as a senior-engineering capstone.

---

## Appendix A — Initial Environment Matrix

| Environment | Purpose | Data | Deployment |
|---|---|---|---|
| local | development | synthetic | Docker Compose |
| test | automated tests | ephemeral | CI |
| staging | release validation | synthetic/sanitized | cloud |
| production | real usage | real | cloud |

---

## Appendix B — Release Checklist

```text
[ ] PR reviewed
[ ] Tests green
[ ] AI eval green
[ ] Security scan green
[ ] Migration reviewed
[ ] Rollback verified
[ ] Staging smoke test passed
[ ] Monitoring dashboard checked
[ ] Alerts checked
[ ] Release notes written
[ ] Production deployment
[ ] Health checks passed
[ ] Error rate normal
[ ] Queue healthy
[ ] Post-release verification complete
```

---

## Appendix C — Project Maturity Levels

### Level 1 — Functional
Happy-path product works.

### Level 2 — Production Backend
Auth, DB, queues, retries, rate limits, tests.

### Level 3 — Senior Engineering
Architecture, capacity, failure handling, observability, security.

### Level 4 — AI Production
Durable agents, MCP, evaluation, cost controls, guardrails.

### Level 5 — Production Operations
CI/CD, cloud, SLOs, alerts, backups, load tests, incident response.

### Level 6 — Tech Lead
ADR/RFC discipline, architecture evolution, trade-off analysis, postmortems, cost engineering, migration strategy.

**Target:** Level 5+.

