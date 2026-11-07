# Sentinel Lab — AI Incident-Response Dashboard

## 1. Purpose and scope

Build a learning-focused, single-operator development project that resembles an internal managed-services incident dashboard. It receives monitoring alerts, gathers evidence, proposes a bounded remediation, asks a human to approve the exact action, executes through a restricted adapter, and verifies the outcome.

This is a portfolio and learning environment, not a production incident-management system. Only synthetic data and personally owned disposable infrastructure are allowed. No real customer information, production credentials, or employer infrastructure.

Primary learning goals:
- Refresh Python syntax, typing, testing, and project structure through implementation.
- Practice SQL, database constraints, joins, aggregations, transactions, and migrations.
- Learn FastAPI, validation, HTTP semantics, authentication, authorization, and API integrations.
- Learn evidence-grounded AI tool workflows, human approvals, and auditability.
- Learn Azure, Terraform, Puppet, monitoring, CI/CD, and operational troubleshooting incrementally.

Non-goals for the first release: Kubernetes, multi-agent orchestration frameworks, model training, pandas/NumPy, vector databases, RAG indexing, .NET, AWS, enterprise identity federation, production high availability, automatic destructive remediation.

## 2. Product scenario

A monitoring event reports that a demo web service is unavailable. An operator opens the incident, requests an investigation, reads evidence and a recommendation, proposes an allowlisted restart, approves the exact target, and executes it. The system records what happened and checks health afterward.

Two synthetic tenants, Acme Lab and Northwind Lab, demonstrate isolation. Two local demo services provide controlled targets. Neither tenant represents a real business.

Happy path:
1. Receive an alert and deduplicate it.
2. Persist an incident scoped to one tenant and resource.
3. Gather timestamped health and runbook evidence.
4. Produce a structured investigation that separates facts from hypotheses.
5. Validate an optional action proposal against backend policy.
6. Present exact action details for human approval.
7. Record approval and execute only that immutable proposal.
8. Verify service health and show the result without falsely claiming recovery.

## 3. Delivery boundaries

### Release A: local, deterministic MVP
- Docker Compose PostgreSQL, FastAPI, React/TypeScript/Vite, and two local demo services.
- Database migrations and synthetic seed data.
- Local login, two tenants, operator/viewer memberships.
- Mock monitoring ingestion and explicit incident creation.
- Deterministic investigator; no paid AI account required.
- Bounded health-check evidence collection.
- Simulated restart only; no shell commands or infrastructure writes.
- Approval, expiry, audit history, idempotency, tests, and SQL exercises.

### Release B: AI and resilient execution
- Azure OpenAI adapter behind the same investigation interface.
- Database-backed jobs and a separate worker, not in-process background tasks for durable work.
- Real health checks against allowlisted local targets; restart still simulated until separately enabled.
- Evaluation fixtures, prompt-injection tests, telemetry, integration tests.

### Release C: Azure lab
- Terraform-managed small disposable Azure environment.
- Linux VM managed by Puppet for a demo service.
- Narrow real remediation adapter for that service only.
- Managed identity and restricted secrets access where applicable.
- Budget awareness, cleanup documentation, deployed API/dashboard, verification checks.

### Release D: optional integrations
- Real LogicMonitor adapter, only if an account is available.
- External identity provider and stronger production-style authorization.
- AWS infrastructure and Bedrock adapters as separate, optional work.

Never label a mock integration as implemented real infrastructure. A project can be portfolio-ready at Release A or B if its limitations are explicit.

## 4. Technology choices

| Area | Choice | Boundary |
|---|---|---|
| Runtime | Python 3.12 | Pin exact supported patch/version in tooling |
| Dependency management | uv + pyproject.toml + committed lockfile | No global installs |
| API | FastAPI + Pydantic v2 | Explicit request/response models |
| Persistence | PostgreSQL 16, SQLAlchemy 2, Alembic | Synchronous database sessions initially |
| Driver | psycopg 3 | Parameterized queries only |
| Tests | pytest, httpx | Real PostgreSQL for integration tests |
| Quality | Ruff + mypy | Incrementally strict; do not suppress errors globally |
| Frontend | React + TypeScript + Vite | Minimal styling; no design-system rabbit hole |
| Frontend tests | Vitest + React Testing Library; Playwright for one happy path | Add only when the relevant UI exists |
| Local runtime | Docker Compose | DB data in a named volume |
| CI | GitHub Actions | Tests first; no automatic infrastructure apply |
| Cloud | Azure | AWS deferred |
| IaC | Terraform | Separate app and lab-resource scopes |
| VM configuration | Puppet | Packages, files, service state; no model-generated manifests |
| AI | Deterministic mock, later Azure OpenAI | No agent framework required initially |
| Monitoring | Mock adapter, later LogicMonitor | Fixture format distinct from vendor payload format |

Use ordinary synchronous FastAPI endpoints for blocking SQL calls. Do not put blocking work inside async endpoints without an appropriate execution boundary. Introduce async only when the learner can explain its benefit and tests protect behavior.

## 5. Architecture

```text
React dashboard
    | authenticated, same-origin /api requests
FastAPI API ---- PostgreSQL (tenants, incidents, evidence, actions, audit, jobs)
    |                        |
    | synchronous MVP        | optional durable queue
    v                        v
Application services <--- Worker
    |
    +-- Monitoring adapter: fixture / LogicMonitor
    +-- Evidence adapter: mock / bounded HTTP health check
    +-- Investigator: deterministic / Azure OpenAI
    +-- Policy engine: deterministic authorization and action checks
    +-- Executor: simulation / narrowly scoped lab-service action
    +-- Verifier: health checks with deadlines
```

The model never receives database write privileges, cloud credentials, an unrestricted browser, arbitrary SQL, or shell access. Backend services own all authoritative decisions.

The dashboard and API are served under one origin in deployment. Vite proxies /api locally. This reduces cookie and CORS complexity. Use separate explicit configuration only if a later deployment requires it.

## 6. Repository structure

```text
backend/
  pyproject.toml
  uv.lock
  alembic.ini
  migrations/
  src/sentinel/
    main.py
    config.py
    db.py
    api/{auth,tenants,resources,incidents,investigations,actions,audit,health}.py
    models/
    schemas/
    services/{incidents,investigations,policy,actions,audit}.py
    adapters/{monitoring,evidence,ai,execution}/
    workers/                 # Release B only
  tests/{unit,integration,fixtures}/
frontend/
  src/{api,components,pages,types}/
  tests/
infra/terraform/{bootstrap,app,lab}/
infra/puppet/
demo-services/
scripts/
docs/{architecture,decisions,runbooks,learning,evaluations}/
sql/exercises/
compose.yaml
.env.example
.gitignore
CLAUDE.md
README.md
```

Create folders when needed, not as empty scaffolding. Avoid generic repositories, plugin systems, and abstractions with only hypothetical consumers.

## 7. Data model

Use UUID primary keys, UTC timestamps, explicit foreign keys, and migrations. Never use the model to generate identifiers for authoritative resources.

### Tables

- tenants: id, slug UNIQUE, name, created_at.
- users: id, email UNIQUE, password_hash, is_active, created_at.
- memberships: tenant_id, user_id, role (viewer/operator); composite primary key.
- sessions: id, user_id, token_hash UNIQUE, csrf_token_hash, expires_at, revoked_at.
- resources: id, tenant_id, name, kind, environment, adapter_key, adapter_config JSONB, is_enabled, created_at. Store only non-secret adapter configuration. Unique (tenant_id, id) supports scoped foreign keys.
- incidents: id, tenant_id, resource_id, source, external_event_id nullable, title, description, severity, status, created_at, updated_at, resolved_at nullable. Unique (tenant_id, source, external_event_id) for non-null external IDs. Synthetic submissions require an idempotency key instead.
- investigations: id, tenant_id, incident_id, status, investigator_kind, model_deployment nullable, prompt_version, summary nullable, hypotheses JSONB, limitations JSONB, started_at, completed_at nullable, error_code nullable.
- evidence: id, tenant_id, incident_id, investigation_id, kind, source, observed_at, collected_at, payload JSONB, content_hash. Append-only after capture.
- action_proposals: id, tenant_id, incident_id, investigation_id, resource_id, action_type, arguments JSONB, proposal_hash, status, policy_version, expires_at, created_at. Action-defining fields immutable.
- approvals: id, tenant_id, action_id UNIQUE, approver_user_id, proposal_hash, decision, reason nullable, decided_at. One decision per immutable proposal.
- executions: id, tenant_id, action_id UNIQUE, status, executor_kind, started_at, completed_at nullable, result JSONB, error_code nullable.
- verifications: id, tenant_id, execution_id, checked_at, healthy boolean nullable, details JSONB.
- audit_events: id, tenant_id, actor_type, actor_id nullable, event_type, entity_type, entity_id, request_id, occurred_at, metadata JSONB. No secrets or full prompts.
- idempotency_records: tenant_id, user_id, route_key, key, request_hash, response_status, response_body JSONB, created_at, expires_at. Unique scope tuple.
- jobs (Release B): id, tenant_id, type, entity_id, status, attempts, available_at, lease_until nullable, last_error_code nullable, created_at.

### Integrity rules

- Every resource-scoped row must belong to the same tenant as its parent. Enforce with composite foreign keys where possible, not just UI filtering.
- No physical delete endpoint for incidents, evidence, approvals, executions, or audit events.
- Use database constraints for enumerations and uniqueness.
- Index incidents (tenant_id, status, created_at), evidence (tenant_id, incident_id), audit_events (tenant_id, occurred_at), and jobs (status, available_at).
- Limit JSONB to adapter payloads and structured variable data. Keep relationships and frequently filtered state in normal columns.
- Add database role-level restrictions on audit UPDATE/DELETE where practical; do not claim audit records are tamper-proof.

## 8. State machines

Incident: open -> investigating -> awaiting_approval -> remediating -> resolved.
Alternative exits: investigating -> open (no safe action); awaiting_approval -> open (rejected/expired); remediating -> open (failed/unhealthy/unknown). Failed investigation must not erase the incident.

Investigation: pending -> running -> completed | failed.
Action proposal: proposed -> approved | rejected | expired; approved -> executing -> succeeded | failed | outcome_unknown. Approved proposals expire too; execution must check expiry again.
Execution: running -> succeeded | failed | outcome_unknown.

A successful executor response does not prove recovery. Only a verification result can support incident resolution. A simulated restart must be visibly labelled and must not automatically resolve a real unhealthy target.

Keep transitions in application services. Never expose arbitrary status patching.

## 9. API contract

All business routes below are relative to /api/v1. List routes return {items, total, limit, offset}; default limit 20, maximum 100. Use a stable secondary sort by id. Return UTC ISO-8601 timestamps and UUID strings.

| Method and route | Behavior |
|---|---|
| POST /auth/login | Authenticate, set session cookie; generic invalid-credentials message |
| POST /auth/logout | Revoke session; clear cookie |
| GET /auth/me | User, memberships, CSRF token |
| GET /tenants | Only the current user's tenants |
| GET /tenants/{tenant_id}/resources | Scoped resource list |
| POST /tenants/{tenant_id}/incidents | Operator creates synthetic incident; idempotency required |
| GET /tenants/{tenant_id}/incidents | Filters: status, severity, resource_id, date range |
| GET /tenants/{tenant_id}/incidents/{incident_id} | Detail and linked latest investigation/action |
| POST /tenants/{tenant_id}/incidents/{incident_id}/investigations | Run deterministic investigation in A; enqueue in B |
| GET /tenants/{tenant_id}/investigations/{id} | Results, cited evidence IDs, limitations |
| POST /tenants/{tenant_id}/investigations/{id}/actions | Materialize validated recommendation; no arbitrary command input |
| GET /tenants/{tenant_id}/actions/{id} | Immutable proposal, decision, execution, verification |
| POST /tenants/{tenant_id}/actions/{id}/decision | decision approve/reject, proposal_hash, optional reason |
| POST /tenants/{tenant_id}/actions/{id}/execute | Execute approved proposal; idempotency required |
| GET /tenants/{tenant_id}/audit-events | Scoped, paginated history |
| GET /tenants/{tenant_id}/statistics | Counts by status/severity, approval outcomes; normal SQL aggregation |
| POST /integrations/mock/alerts | Development-only ingestion with separate fixture authentication |
| GET /health/live | Process alive; no dependency details |
| GET /health/ready | Dependency readiness; return 503 when unavailable |

Return 201 for creation; 200 for synchronous results; 202 for accepted background work; 204 for logout. Return 401 unauthenticated, 403 insufficient role, 404 inaccessible/missing scoped entity, 409 illegal transition or reused idempotency key with changed payload, 422 invalid input, 429 rate limit, 503 unavailable dependency.

Error format: {error: {code, message, request_id, details}}. Never leak stack traces, secrets, or existence of another tenant's resources.

Idempotency-Key is required for mutating business operations that create work or decisions. Store the canonical request hash. Same key and same payload returns the stored response; changed payload returns 409. The execution uniqueness constraint remains the final duplicate-execution barrier.

## 10. Authentication and tenant isolation

Release A uses local email/password accounts seeded explicitly for development. Hash passwords with a maintained password-hashing library using Argon2id. Never implement cryptography manually.

Use random opaque session tokens; store only token hashes. HttpOnly cookies, SameSite=Lax, Secure outside localhost. Validate Origin on mutations and require a session-bound CSRF token. Set an explicit session lifetime (8 hours for the lab). Implement basic login throttling and avoid logging credentials.

Viewer: read-only. Operator: create incidents, investigate, decide, execute within their tenant. A single operator may propose and approve in this lab; explicitly document that this is not separation of duties. A later four-eyes mode must disallow self-approval server-side.

Every route checks membership before loading scoped data. Never trust tenant_id from a request without authorization. Worker jobs carry tenant context and revalidate resource scope. Write negative tests for guessed IDs, altered tenant paths, and mismatched parent relationships.

## 11. Evidence and AI contract

Implement two investigators behind a small interface:
- MockInvestigator: deterministic fixture-based output.
- AzureOpenAIInvestigator: structured model output validated by Pydantic.

Input: incident details, authorized resource metadata, bounded evidence, selected local runbook. Exclude secrets and irrelevant customer data.

Output schema:
- summary: string, maximum 2000 characters.
- findings: list of {statement, evidence_ids}.
- hypotheses: list of {description, evidence_ids, missing_information}.
- limitations: list of strings.
- recommendation: null or {action_type, resource_id, reason, evidence_ids}.

Allowed initial action_type: simulated_restart_service. Later: restart_demo_service, separately gated. The model cannot supply arbitrary executable arguments; backend derives service and target from registered resource configuration.

Validation rules:
- All cited evidence IDs exist and belong to this investigation.
- Target resource matches the incident and tenant.
- No unsupported action types.
- Invalid output produces a visible investigation failure; never silently invent a valid result.
- Malicious instructions in logs/runbooks are treated as untrusted content.
- No autonomous repeating loop. First implementation has a fixed gather -> analyze -> validate sequence.

Runbooks are reviewed repository files. Their contents inform recommendations but cannot grant permissions.

AI calls: bounded input size, maximum output tokens, explicit timeout, at most one retry for transient read-only generation failures, redacted error reporting. Persist deployment name and prompt version, not sensitive raw prompts by default. Never route to a different provider automatically without explicit configuration.

## 12. Action safety and execution

The proposal hash is SHA-256 over a canonical JSON serialization of tenant_id, resource_id, action_type, validated arguments, evidence IDs, policy_version, and expires_at. It is an integrity identifier, not a substitute for authorization or a cryptographic approval signature.

Approval screen must show: customer, resource, environment, action, exact arguments, expected effect, risk note, evidence age, expiry, and simulation/real badge.

Default proposal lifetime: 15 minutes, configurable. New evidence or changed targets require a new proposal; do not edit approved proposals.

Execution transaction:
1. Lock proposal row.
2. Check actor permission, tenant/resource scope, approval hash, expiry, policy version, and enabled execution mode.
3. Insert one execution row and move to executing atomically.
4. Commit before external work; external calls do not run inside a long DB transaction.
5. Execute bounded tool.
6. Persist outcome, verify, audit.

Exactly-once external effects are not guaranteed by a database transaction. If a worker crashes or a write times out after submission, mark outcome_unknown and reconcile actual state before any retry. Do not blindly retry remediation.

Real restart adapter remains disabled by default. It may target only the personally owned demo VM and one hardcoded/validated service name. No arbitrary shell strings, user-provided hosts, scripts, package installs, file deletion, or infrastructure destruction. Credentials must not be visible to the model.

HTTP evidence collection must prevent SSRF: resource registry defines targets; allowlist host/port/protocol; prohibit arbitrary URLs and redirects; bound response size and timeout; never expose cloud metadata endpoints. Real cloud targets require explicit allowlist configuration.

## 13. Worker and event processing (Release B)

Use a PostgreSQL jobs table to avoid introducing Redis/Celery for the first durable implementation. Atomically create job and business record in one transaction. Claim eligible jobs using locking, maintain leases, and recover expired leases.

Read-only collection/generation jobs may use bounded retries with backoff. Execution jobs require reconciliation after unknown outcomes, not ordinary retries. This provides at-least-once job handling, not guaranteed exactly-once effects.

Use correlation/request IDs across API, jobs, evidence, and execution. A full message broker and transactional outbox are optional future work, not MVP dependencies.

## 14. Frontend specification

Pages:
1. Login.
2. Tenant selector.
3. Incident list: status/severity filters, pagination, create synthetic incident.
4. Incident detail: description, resource, timeline, evidence, investigate button.
5. Investigation view: findings, hypotheses, limitations, evidence links.
6. Action review: exact proposal details, approve/reject, explicit execute control.
7. Audit view and statistics summary.

UI requirements:
- Always show active tenant and execution mode.
- Treat backend authorization as authoritative; hiding buttons is not security.
- Loading, empty, error, expired-session, and stale-action states.
- No optimistic success for approvals or execution.
- Disable repeat submission while pending but still enforce server-side idempotency.
- Render evidence as plain text; no unsanitized HTML.
- Use labelled controls and keyboard-accessible dialogs.
- MVP may poll running work every 2 seconds, stop on terminal status, and stop when page unmounts. WebSockets deferred.

## 15. SQL learning requirements

Implement statistics in SQL rather than loading every row into Python. Add separate exercises with seed data, expected results, and explanations:
1. Incidents per tenant and status (GROUP BY).
2. Incidents and resource names (JOIN).
3. Resources with no incidents (LEFT JOIN).
4. Latest investigation per incident (ROW_NUMBER window function).
5. Daily incident totals with zero-count days (PostgreSQL date series).
6. Approval and rejection counts per operator.
7. Median verified resolution duration, excluding unresolved incidents.
8. Duplicate alert detection and the supporting uniqueness constraint.
9. Inspect EXPLAIN for a filtered incident query before/after an index.

ORM for ordinary persistence is acceptable. Write and explain explicit parameterized SQL for at least the reporting exercises. Never interpolate request values into SQL strings.

## 16. Testing and acceptance criteria

Unit tests: policy checks, proposal hashing, allowed state transitions, model-output validation, mock investigator, adapter failure classification.

PostgreSQL integration tests: migrations from empty DB; cross-tenant foreign-key rejection; idempotency; duplicate alerts; approval expiry; concurrent execution requests; rollback behavior; audit records; auth/CSRF and membership checks.

Security regression tests:
- Viewer cannot mutate.
- Tenant A cannot read or mutate Tenant B entities.
- Client cannot choose execution target/command.
- Expired or rejected proposal cannot execute.
- Modified proposal hash is rejected.
- Repeated execution produces one execution record.
- Injected log instruction does not authorize tools.
- Invalid evidence IDs and model responses fail safely.
- Arbitrary evidence URLs are rejected.
- Action success with unhealthy verification does not resolve incident.
- Unknown external outcome is not automatically retried.

End-to-end flow: login -> select tenant -> create incident -> investigate -> inspect evidence -> approve -> simulate execution -> inspect audit. Include rejected-action and tenant-isolation flows.

AI evaluation fixtures: healthy endpoint, unavailable endpoint, insufficient evidence, stale evidence, malicious log instruction, unsupported action request. Record schema validity, evidence-reference validity, unsafe-action proposals, unsupported factual claims, latency, and approximate token usage. Small fixture results are not production safety guarantees.

Release A is done when setup is reproducible from the README, all critical tests pass, two tenants are demonstrably isolated, approval gating works, every mutation has audit history, and simulations are unmistakable.

## 17. Terraform, Puppet, and Azure implementation

Do not deploy cloud resources before local acceptance criteria pass.

Terraform scopes:
- bootstrap: state storage, created through a documented bootstrap procedure.
- lab: one small disposable VM, network, restricted administrative access, and demo-service dependencies.
- app: application hosting and database resources only when approved and budgeted.

Remote state may contain sensitive information. Do not commit state, plans, variable files containing secrets, or credentials. Restrict state access. CI validates/formats/plans; applying cloud changes requires a separate reviewed approval. Saved plans are sensitive artifacts with short retention.

Puppet manifest: install reviewed web-server package, manage one demo page/config file, ensure the service is enabled/running. Start with a manual apply on the lab VM; a Puppet server is not required for this project. Verify licensing/account requirements before choosing a distribution; do not assume every Puppet product is free.

Fault injection must be explicit, lab-only, and separate from the normal configuration loop. If Puppet continuously enforces running state, it may repair the stopped service before the incident workflow does. For the first exercise use on-demand Puppet application, document this trade-off, and avoid two competing remediation controllers.

Azure deployment choices are a design decision, not an assumed free tier. Start with the VM lab and keep app local if needed. Before deploying managed PostgreSQL, networking, registry, monitoring, or application hosting, inspect current regional pricing and service constraints. Use managed identity where supported; otherwise use restricted secrets outside source control.

Set budgets and alerts, but do not treat them as hard spending caps. Define cleanup commands and verify deletion of disks, IPs, logs, registries, and other chargeable resources. No cloud creation by a coding agent without explicit owner approval.

## 18. CI/CD and observability

Pull requests: lint, type checks, unit tests, PostgreSQL integration tests, frontend build/tests, migration check, Terraform format/validate when relevant.

Deployment: separate manual workflow, environment-scoped credentials, explicit approvals for infrastructure or real action enablement. Prefer federated identity over long-lived cloud keys when configured.

Structured logs: timestamp, severity, service, request_id, tenant_id when appropriate, event_type, duration, safe error code. Redact passwords, tokens, connection strings, raw sensitive evidence. Do not log full HTTP bodies by default.

Measure: request count/errors/latency, investigation failures, action decisions, execution outcomes, verification failures, queue age, model latency/token usage. An audit trail records who did what; telemetry helps diagnose system behavior. Do not confuse the two.

## 19. Phased backlog and learning checkpoints

Estimates are focused working sessions of roughly 2–3 hours, not calendar promises. Integrations and cloud setup may take longer.

### P0 — Decisions and setup (1 session)
Deliver: repo, backend environment, frontend shell, Compose PostgreSQL, .env.example, first health route, first test.
Learn: Python environment, modules, HTTP app startup.
Exit: clean checkout can start DB/API; test passes.

### P1 — Database and SQL (2–3 sessions)
Deliver: migrations, tenants/users/resources/incidents, repeatable seed command, SQL exercises 1–3.
Learn: SQLAlchemy model vs Pydantic schema, foreign keys, sessions, transactions.
Exit: migrations upgrade empty DB; duplicate and cross-tenant tests pass.

### P2 — Authentication and incident UI (2–3 sessions)
Deliver: sessions, CSRF, memberships, list/detail/create, pagination, negative auth tests.
Learn: cookies, hashing, authorization, frontend API handling.
Exit: tenant A cannot access B; viewer cannot mutate.

### P3 — Evidence and deterministic investigation (2 sessions)
Deliver: registered mock/HTTP targets, evidence persistence, fixed workflow, runbook, structured output.
Learn: protocols/interfaces, timeouts, data validation, evidence provenance.
Exit: unavailable target and insufficient evidence are represented honestly.

### P4 — Approval and simulated execution (2–3 sessions)
Deliver: immutable proposals, hash, expiry, state machine, idempotency, simulated adapter, audit view.
Learn: locking, races, transaction boundaries, side-effect safety.
Exit: duplicate/concurrent/expired/unapproved action tests pass.
Release A complete.

### P5 — Worker and AI adapter (3–4 sessions)
Deliver: jobs/leases, Azure OpenAI adapter, fake-client tests, evaluation fixtures.
Learn: at-least-once processing, unknown outcomes, model schema validation.
Exit: crash/timeout tests and evaluation report; deterministic mode still works offline.
Release B complete.

### P6 — Azure/Terraform/Puppet lab (3–5 sessions)
Deliver: reviewed plan, disposable VM, Puppet manifest, fault-injection runbook, restricted restart, cleanup guide.
Learn: resources/state/modules, identity/networking, configuration vs provisioning.
Exit: manually demonstrated lab incident with approval and independent verification; full cleanup confirmed.

### P7 — Real monitoring and polish (variable)
Deliver: LogicMonitor integration if access exists, API payload contract fixtures, deployed demo if budget allows, architecture diagram, short demonstration video.
Exit: documentation distinguishes real, mocked, and deferred capabilities.

### Optional P8 — AWS extension
Only after earlier releases: add AWS lab modules and Bedrock investigator under existing interfaces. Do not rewrite the platform for hypothetical multi-cloud requirements.

## 20. Expected developer commands

These are target interfaces to implement, not claims that scripts already exist. Keep README commands aligned with the actual repository.
- make setup: install locked backend/frontend dependencies.
- make dev: start local services with documented prerequisites.
- make migrate: apply Alembic migrations.
- make seed: idempotently create synthetic accounts/tenants/resources; print no secrets.
- make test: unit and integration tests with isolated test DB.
- make lint: Python/frontend quality checks.
- make demo: load one deterministic incident scenario; no real remediation.
- make stop: stop containers without deleting data.
- make reset-dev: explicit confirmation before deleting local lab data.

Document development credential setup using environment input or an interactive command. Never publish default production-usable credentials.

## 21. Portfolio deliverables

- Accurate architecture diagram and state diagram.
- Three-minute deterministic demo.
- Threat-model note and explicit security limitations.
- SQL exercises with explanations.
- ADRs for synchronous DB, no agent framework, immutable approvals, and mock-first delivery.
- Test evidence and small AI evaluation report when applicable.
- Setup, fault injection, troubleshooting, deployment, and teardown runbooks.
- Capability table: implemented locally / implemented in cloud / mocked / planned.

Suggested first task for the coding agent: read CLAUDE.md and this specification, summarize Release A, ask only blocking questions, then propose P0 as a small change. Do not generate the whole project at once.
