# AGENTS.md

This file provides guidance to AI coding agents working in this repository.
All paths in this document are relative to the repository root.

<!-- FOR THE PERSON
This is a reusable template for backend/API projects. Copy this file as
AGENTS.md into the project root and fill in every [PENDING] placeholder with
the actual stack and repository conventions. Remove sections that don't apply
(e.g. messaging, if the project doesn't publish events) and adjust path/package
names to match the project. Keep the overall structure and the spec-driven
workflow unless the project already has its own established process.
-->

## Required reading

Before planning, reviewing, or modifying this project, read
[docs/GENERIC_RULES.md](docs/GENERIC_RULES.md) in full and apply it alongside
this file. The link is an explicit reading requirement; do not assume your
tool automatically imports linked Markdown files.

GENERIC_RULES.md contains reusable working rules. This file adds the project's
backend conventions and spec-driven workflow. For repository conventions,
project-specific rules refine the generic defaults. Neither file overrides
the user's explicit instructions or the agent's higher-priority instructions.
If documents conflict in a way that affects behavior or scope, clarify the
conflict before implementing the affected part.

## Project

[PENDING: brief project description, e.g. "REST/GraphQL API built with
[LANGUAGE] and [FRAMEWORK]".]

- Module/service structure: [PENDING: single-module monolith, monorepo with
  multiple services, etc.]
- Package/namespace root: [PENDING]
- Baseline functionality: [PENDING: what the system does today, in one sentence]
- The existing [PENDING: reference feature] is the reference for new
  features. Inspect its implementation before extending the service.

## Commands

Run commands from the repository root.

```bash
[PENDING: build/compile command]
[PENDING: command to run unit tests]
[PENDING: command to run integration tests]
[PENDING: lint/format command]
[PENDING: command to run the service locally]
[PENDING: command to run database migrations]

# Run a single test; replace with the project's actual filter syntax
[PENDING: command to run a specific test or class]
```

[PENDING: where unit tests vs. integration tests live, e.g. "Unit tests live
in `src/test`; integration tests live in `src/integrationTest` and require a
running database (see docs/ for local setup)."]

There is no separate [PENDING: quality tool] configuration beyond what's in
the repository. Use the existing configuration; do not introduce a new
quality tool as an unrelated change.

For code changes, run [PENDING: list of commands required before reporting
completion] before reporting completion. Run relevant integration tests and
manual checks (e.g. via [PENDING: HTTP client, Postman/Bruno collection, curl
scripts]) when acceptance criteria require end-to-end behavior. For
documentation-only changes, verify content and references without requiring
a full build. Report any checks that could not run and the reason.

## Spec-driven workflow

### Reference documents

Each template carries its own instructions for the agent, its section
structure, and its identifier scheme. Read the template in full and follow it;
this file does not restate its content and must not contradict it.

- `docs/SPEC_TEMPLATE.md`: copy to the feature's `SPEC.md` when creating or
  updating a specification. It owns the spec's sections, its `RF-` requirement
  and `CA-` acceptance-criteria identifiers, and its approval states.
- `docs/PLAN_TEMPLATE.md`: copy to the feature's `PLAN.md` once the
  specification is approved. It owns the plan's sections and approval states.
  Reference requirements and criteria by their spec identifiers.
- `docs/API_GUIDELINES.md`: consult when writing or reviewing requirements,
  acceptance criteria, the technical plan, and backend validation. Consider
  API contract and versioning, authentication and authorization, input
  validation, idempotency, transactionality, concurrency, error handling and
  status codes, persistence and migrations, performance and scalability
  (pagination, caching, query cost), observability (logging, metrics,
  tracing), and data privacy. Apply only relevant items; clarify undefined
  product behavior instead of inventing it.

Preserve each template's structure and its embedded comments in the copy. Do not
rewrite the shared templates for an individual feature.

### Feature documents

Keep each feature's documents together:

- `docs/features/<feature-name>/SPEC.md`
- `docs/features/<feature-name>/PLAN.md`
- `docs/features/<feature-name>/TASKS.md`

Use an existing feature directory when continuing its work.

### Sequence

Each stage is gated by the previous document's state. A document is only
"Approved" when the user says so: a complete document is not an approved
one, and the agent never changes that state on its own.

1. **Specification:** complete `SPEC.md` from `docs/SPEC_TEMPLATE.md`,
   collaboratively and section by section, following the template's own
   instructions. Do not start the plan until the user approves the spec.
2. **Plan:** write `PLAN.md` from `docs/PLAN_TEMPLATE.md` for the approved
   specification, following the template's own instructions. Additionally,
   record whether subagents are needed and their bounded responsibilities; do
   not assume delegation is required or available. Do not start tasks until
   the user approves the plan.
3. **Tasks:** derive `TASKS.md` from the approved plan. There is no shared
   template for it, so this file defines it: small, ordered, verifiable
   checkboxes, each with an identifier, objective, scope, dependencies, the
   spec criteria it resolves, and its validation method. Keep tasks concise
   enough to execute and detailed enough to determine when they are done.
4. **Implementation:** execute the tasks within the agreed scope, preserving
   the architecture below. Authorization to implement must be explicit; it is
   not implied by the documents' state. Update task status as work progresses.
5. **Validation:** verify acceptance criteria with appropriate evidence and
   record the result in `TASKS.md`, including any outstanding checks.

Do not treat a filled template as resolution of unanswered questions. Ask
about missing product decisions that affect behavior; resolve routine technical
details from the code and established conventions. Do not ask for renewed
permission for steps the user has already authorized.

If implementation reveals a requirement gap or contradiction, clarify the
affected behavior and update the relevant documents before continuing that
part. Keep the specification, plan, tasks, and resulting behavior consistent.

## Architecture

[PENDING: architectural style name, e.g. "Clean Architecture" / "Hexagonal" /
"Layered"] with [PENDING: flow pattern, e.g. "explicit request/response
boundaries"]. The existing [PENDING: reference feature] is wired end to end
with this structure, under [PENDING: source root]:

| Path | Responsibility |
| --- | --- |
| `[PENDING]/api/*Controller.[ext]` (or `*Handler`) | Entry point: parses requests, calls use cases, maps responses and status codes |
| `[PENDING]/api/dto/*.[ext]` | Wire DTOs (request/response), separate from domain models |
| `[PENDING]/api/mapper/*.[ext]` | DTO-to-domain and domain-to-DTO mapping functions |
| `[PENDING]/domain/model/*.[ext]` | Plain domain models |
| `[PENDING]/domain/*Repository.[ext]` | Repository contract owned by domain |
| `[PENDING]/domain/usecase (or service)/*.[ext]` | Use cases / application services containing business logic |
| `[PENDING]/data/*RepositoryImpl.[ext]` | Repository implementation: persistence access and mapping to domain |
| `[PENDING]/data/entity/*.[ext]` | Persistence entities/models, separate from domain models |
| `[PENDING]/config or di/*.[ext]` | Dependency wiring, configuration, environment |

### Layer dependencies

- `api -> domain` and `data -> domain`.
- Domain owns repository interfaces and must not depend on the API layer,
  data implementations, persistence entities, or framework-specific classes.
- The API layer accesses domain use cases and models, never data
  implementations or persistence entities directly.
- Data implements domain contracts and maps persistence representations to
  domain models. Dependency injection wires implementations to contracts.

### Dependency injection

- [PENDING: DI framework or mechanism used, e.g. Spring, NestJS providers,
  manual constructor injection]. Prefer constructor injection for
  project-owned classes.
- Follow the existing configuration in [PENDING: path] for wiring interfaces
  to implementations.
- Match dependency scopes (singleton, request-scoped, transient) to their
  intended lifetime and existing conventions.

### API design and contracts

- [PENDING: REST / GraphQL / gRPC / other]. Follow the existing conventions
  for [PENDING: route naming, versioning, error format].
- Define request/response contracts in [PENDING: OpenAPI/schema location]
  and keep it in sync with the implementation.
- Validate input at the boundary before it reaches domain logic.
- Map domain/application errors to HTTP status codes consistently; do not
  leak persistence or framework exceptions to the response.

### Persistence and data access

- [PENDING: ORM/query builder/driver used, e.g. Prisma, TypeORM, SQLAlchemy,
  JPA/Hibernate, raw SQL]. Do not introduce an alternative data access tool
  without justification.
- Migrations live in [PENDING: path] and are [PENDING: reversible /
  non-reversible]; follow the existing migration process.
- Keep persistence entities in the data layer and expose only domain models
  to callers outside it.

### Concurrency, transactions, and errors

- Use [PENDING: concurrency model of the language/framework, e.g.
  async/await, coroutines, threads] consistently with the rest of the
  codebase.
- Wrap multi-step writes in transactions where partial failure would leave
  inconsistent state; document the boundary in the plan.
- Make write endpoints idempotent where the spec requires safe retries.
- Handle expected failures explicitly and reflect them in the API response as
  required by the spec. Do not swallow cancellation/interruption signals.
  Keep blocking work off any event loop or reactive thread where applicable.

### Security

- [PENDING: authentication mechanism, e.g. JWT, OAuth2, sessions] and
  [PENDING: authorization mechanism, e.g. roles, scopes, ABAC].
- Validate and sanitize all external input; never trust client-provided
  identifiers for authorization decisions.
- Avoid logging sensitive data (secrets, PII, tokens); follow
  [PENDING: the project's logging policy, if any].

### Observability

- [PENDING: structured logging library/format]
- [PENDING: metrics and tracing, if applicable]
- Log enough context to diagnose failures without leaking sensitive data.

### External integrations

- [PENDING: conventions for calls to other services/external APIs: HTTP
  client used, timeout and retry handling, circuit breaking if applicable].
- Keep integration-specific DTOs out of the domain layer; map at the
  boundary, following the existing pattern in [PENDING: example path].

## Completion report

Summarize the implemented behavior, the affected feature documents, and the
validation results. Link acceptance criteria to evidence in `TASKS.md`.
Clearly identify anything incomplete or unverified; do not present a test name,
an unexecuted command, or "should work" as proof of success.
