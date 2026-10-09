# API Guidelines

When writing a specification, consider the following points and apply the
ones relevant to the feature. If a behavior is left undefined, clarify it
before assuming it.

- **API contract and versioning:** define the request/response schema, status
  codes, and whether the change is backward compatible. Determine if a new
  version is required and how existing consumers are affected.
- **Authentication and authorization:** define who can call the endpoint and
  under what role, scope, or ownership condition. Contemplate unauthenticated,
  unauthorized, and expired-credential cases.
- **Input validation:** define what is validated at the boundary before
  reaching domain logic, and what happens with malformed, missing, or
  out-of-range data. Distinguish client errors from server errors in the
  response.
- **Idempotency and duplicate prevention:** consider retries, double
  submissions, and out-of-order requests. Define which operations must be
  safe to repeat and how duplicates are detected or avoided.
- **Transactionality and consistency:** define which operations must succeed
  or fail as a unit, and what happens on partial failure. Consider
  consistency between the primary data store and any other affected system.
- **Concurrency:** consider simultaneous requests affecting the same
  resource, race conditions, and locking strategy. Define the expected
  outcome when two operations conflict.
- **Persistence and migrations:** define what is stored, how it evolves over
  time, and compatibility with data created by previous versions. Consider
  schema migrations, backfills, and rollback.
- **Downstream dependencies and failure handling:** contemplate timeouts,
  unavailable dependencies, degraded responses, and retry behavior. Define
  what the caller sees when a downstream dependency fails.
- **Response states and error handling:** contemplate success, partial
  success, validation error, not found, conflict, and server error. Use a
  consistent error format and avoid exposing internal details in the
  response.
- **Long-running and background work:** when applicable, consider work that
  cannot complete within the request/response cycle. Define how the client
  is notified of completion and how duplicate effects are avoided if the
  work is retried.
- **Cancellation and client disconnects:** consider what happens when a
  client disconnects or cancels before a response is returned. Define
  whether in-flight work is stopped, completed, or left to finish
  independently.
- **Performance and scalability:** consider query cost, pagination, response
  size, and behavior under load. Contemplate rate limiting or throttling
  when the endpoint is exposed to untrusted or high-volume clients.
- **Observability:** consider what is logged, measured, and traced for this
  behavior. Ensure enough context is captured to diagnose failures without
  logging sensitive data.
- **Privacy and security:** consider what information is stored, transmitted,
  or returned, how it is protected, and when it is deleted. Avoid sensitive
  data in logs, error messages, and responses to unauthorized callers.
- **Internationalization and formats:** when applicable, consider
  localization of returned content, text length, dates, time zones, numbers,
  and units.
- **Backend validation:** include tests for the relevant scenarios, such as
  invalid input, authorization failures, downstream failures, concurrent
  requests, and retries. Complement automated tests with integration or
  manual checks against a running instance when needed to validate real
  behavior.

These points do not automatically expand scope. Incorporate into SPEC.md the
requirements, technical decisions, and acceptance criteria that follow from
the applicable points and confirmed decisions.
