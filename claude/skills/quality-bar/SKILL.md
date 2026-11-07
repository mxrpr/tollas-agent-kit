---
name: quality-bar
description: Shared code quality standard (security, correctness, error handling, concurrency, performance, architecture, tests, docs, tooling) with Critical/High/Medium/Low severity definitions. Use when implementing or reviewing code changes, self-reviewing a diff, or checking whether code would pass a comprehensive review.
---

# Quality Bar

Shared standard for the dev pipeline. The implementation agent (code-agent) builds to it; the review agent (code-review) judges against it. It mirrors the dimensions and severities used by the comprehensive-review plugin, so code that meets this bar should not produce Critical or High findings there.

Scope: apply it to the code you added or changed and to the code paths it directly affects. Pre-existing issues outside that scope are reported as "pre-existing", not fixed silently and not counted against the change.

## Severity definitions

- **Critical**: exploitable security hole, data loss or corruption, crash or outage on a normal path, broken core functionality.
- **High**: security weakness needing specific conditions, incorrect behaviour on a realistic edge case, unhandled failure of an external dependency, serious performance problem on a hot path, missing tests for security-critical or core logic.
- **Medium**: maintainability problems, minor edge cases, inconsistent conventions, missing tests for non-critical paths.
- **Low**: style, naming, small readability improvements.

Critical and High must be zero before a change is handed off. Medium should be fixed unless there is a stated reason not to.

## 1. Security (OWASP Top 10, CWE)

Critical/High if violated:
- [ ] All untrusted input (user, network, file, env, IPC, deserialized data) is validated for type, length, range and format at the trust boundary.
- [ ] No injection: SQL/NoSQL use parameterized queries; shell commands use argument arrays, never string concatenation; no `eval` of untrusted data; output is encoded for its context (HTML, JS, URL, log).
- [ ] File paths built from input are canonicalized and confined to an allowed root (no path traversal).
- [ ] Every operation that reads or changes protected data checks authentication AND authorization (who is calling, and may they touch THIS resource).
- [ ] No secrets, keys, tokens or passwords in code, logs, error messages or test fixtures committed to the repo.
- [ ] Cryptography uses vetted libraries and current algorithms; no homemade crypto, no MD5/SHA1 for security, no fixed IVs or salts; random values for security use a CSPRNG.
- [ ] Deserialization of untrusted data uses safe formats/settings (no arbitrary type instantiation).
- [ ] Errors returned to callers do not leak stack traces, internal paths, queries or secrets.
- [ ] No insecure defaults: debug mode off, permissive CORS off, TLS verification on, least-privilege permissions.
- [ ] New dependencies are maintained, pinned, and free of known CVEs.

## 2. Correctness and error handling

Critical/High if violated:
- [ ] Behaviour matches the task's definition of done, including stated edge cases.
- [ ] Edge cases handled: empty, null/None, zero, negative, max size, unicode, duplicate, missing key, concurrent call.
- [ ] Every fallible call (I/O, network, parse, DB, external API) has its failure handled: no swallowed exceptions, no empty catch blocks, no ignored error returns, no `unwrap`/`!`/bare `.Result` on fallible values outside tests.
- [ ] Error messages say what failed and why, with enough context to debug, and without sensitive data.
- [ ] Resources (files, connections, locks, handles, subscriptions) are released on every path, including error paths.
- [ ] Partial failures leave data consistent (transactions, rollback, or idempotent retry).
- [ ] Timeouts on all network and external calls.

## 3. Concurrency and performance

Critical/High if violated:
- [ ] Shared mutable state is protected; no race conditions, no check-then-act without a lock or atomic operation.
- [ ] No deadlock risk: consistent lock order, no lock held across await/I/O when avoidable.
- [ ] No blocking calls on async/event-loop/UI threads.
- [ ] No N+1 queries or calls inside loops that could be batched.
- [ ] Collections, caches, queues and buffers are bounded; large results are paginated or streamed.
- [ ] No needless quadratic work on inputs that can grow.

## 4. Code quality and architecture

High if seriously violated, otherwise Medium:
- [ ] Follows the project's existing patterns, layering and module boundaries; no new dependency direction that inverts the architecture.
- [ ] No circular dependencies; no tight coupling to concrete implementations where the project uses abstractions.
- [ ] Functions do one thing; deep nesting and long functions are split.
- [ ] No duplicated logic where a helper already exists in the codebase.
- [ ] No dead code, commented-out code, debug prints, or TODOs without an issue reference.
- [ ] Public API changes are backward compatible, or the break is intentional and documented.
- [ ] No over-engineering: no abstractions, options or features beyond the task.

## 5. Tests

High if violated for security-critical or core logic, otherwise Medium:
- [ ] New behaviour has tests; bug fixes have a regression test that fails without the fix.
- [ ] Tests cover the error paths and edge cases from sections 1-3, not only the happy path.
- [ ] Tests assert behaviour, not implementation details; assertions are specific.
- [ ] Tests are isolated and deterministic (no real network, no sleep-based timing, no order dependence).

## 6. Documentation

Medium unless it hides a breaking change (then High):
- [ ] Non-obvious logic, invariants and security assumptions are commented (why, not what).
- [ ] Public functions/types follow the project's doc comment convention.
- [ ] README, API docs, config docs and changelog updated when behaviour, setup or interfaces change.

## 7. Tooling

High if violated:
- [ ] The project builds with no new warnings.
- [ ] The project's linters, formatters and static analyzers pass on changed files (e.g. clippy, eslint, ruff, mypy, semgrep, `dotnet build -warnaserror`, `go vet`), when the project has them.

## 8. Lessons from past reviews

Patterns that past comprehensive reviews flagged High or Medium, often more than once. The implementation agent must not repeat them. The review agent rates any repeat as at least High.

This section holds the most important lessons.

### Catalog and caps
The complete catalog, all severities, is split by stack in this skill's directory (`~/.claude/skills/quality-bar/`):
- `review-lessons-general.md`: language-agnostic; read for every change.
- `review-lessons-rust.md`: Rust backend.
- `review-lessons-frontend.md`: React/TypeScript frontend.
- `review-lessons-sql.md`: SQL, databases and migrations.

Read the general file plus the file for each stack the change touches.

Hard caps, enforced by whoever writes a lesson:
- Section 8: at most 60 rules.
- Each catalog section (`##`): at most 30 rules.
- Each catalog file: at most 150 rules.

A new lesson that would exceed a cap is not appended. Instead, merge it into the closest existing rule, or generalize two or more existing rules into one, so the count stays at or below the cap. Never drop a High rule to make room; merge Low rules first. When any count passes 90% of its cap, run the `lessons-compact` skill. A new stack file (e.g. `review-lessons-python.md`) may be created when a lesson fits no existing file; add it to the list above.

### External calls, retries and partial failure
- [ ] Every upstream error is classified as transient (retry), permanent for one item (skip and count), or fatal (authentication, exhausted rate limit: abort the run). A per-item skip never swallows a fatal error.
- [ ] Any skip, truncation, page cap or failed sub-fetch is reported as partial or incomplete, never as a clean success. One failed sub-list does not discard its siblings.
- [ ] A dropped connection or timeout while reading a body is transient; only a real decode error counts as a bad payload.
- [ ] Optional or cosmetic enrichment calls get no rate-limit retries and stop for the rest of the run after the first rate limit.
- [ ] A retry or restart gets its own request/page budget, so the retry alone can still finish a maximal valid run.
- [ ] When a deadline cancels in-flight work that was already paid for (rate limiter, quota), keep or cache what already arrived.
- [ ] Fallback and live-read paths get a short timeout and no default retries; every HTTP, SMTP and DB call has a timeout.
- [ ] 4xx is not "permanent" as a class: 408/409/423/425/429, connection reset and body-read timeout are transient.
- [ ] A failed lookup never overwrites or blanks a previously stored value; keep the old value or surface the failure.
- [ ] Field types and shapes of an external API come from its docs or a recorded response, never a guess; when they disagree, accept both shapes.
- [ ] When an upstream payload loses fields, list every field the consumers read and restore all of them, not only the visible symptom.
- [ ] When a direct lookup is replaced by a search endpoint, pin the search's implicit filters (active-only, visibility, limits) explicitly.
- [ ] An input allowlist in front of an external system matches what that system accepts (or a safe superset), never a narrower guess.

### Bounds and performance
- [ ] Every list call pages to the end or has an explicit, logged cap; every page loop has a max-pages and repeated-token guard.
- [ ] User-controlled ranges (days, items) have a maximum; expensive queries have a statement timeout; expensive routes have a concurrency limit.
- [ ] Never sort, window or DISTINCT rows that carry a large payload column (JSON blobs): sort narrow keys, then join the payload.
- [ ] Batch writes are one set-based statement (e.g. `UNNEST`), not a per-row loop inside a transaction.
- [ ] In-process caches and limiters assume a single replica; state it, or keep the state shared.

### Correctness and data integrity
- [ ] Stored timestamps come from the item's own event data, never from sync or observation time.
- [ ] Dedup keys use stable identity, never a type derived from the item's current status (output must not depend on when the sync saw it).
- [ ] Retry and backoff clocks advance only on real attempts; failed, skipped or swallowed ticks neither reset nor age them.
- [ ] Date and number arithmetic on user input uses checked operations; no user-reachable panic or overflow.
- [ ] Every path that changes a given state (create, enable, re-enable, link, role change) triggers the same side effects (sync, reset, cache invalidation). Grep every writer of that state.
- [ ] A guard or lock added to one writer goes on every writer of the same data, written once in a shared helper, and each writer is tested.
- [ ] Rows from several tables are locked in one global order; same-table rows are locked `ORDER BY id`.
- [ ] Before a delete, check what cascades; never silently erase records that other features treat as history.
- [ ] Comments next to changed code still match it (e.g. a "no lock needed" comment after adding a lock).
- [ ] Untrusted text (titles, descriptions) placed into LLM prompts goes through the project's sanitizer.
- [ ] Scoping and authorization filters (tenant, project allowlist) go through one chokepoint, not repeated per query.

### Frontend
- [ ] Use `||`, not `??`, when an empty string also means "missing".
- [ ] After a mutation, invalidate every query that displays the changed data.
- [ ] Conflict (409) and validation errors are shown to the user in the form or dialog that caused them.

### Single source of truth
- [ ] No hand-kept list that mirrors an enum or registry; derive it from the enum or registry.
- [ ] Before writing a mapping (status code to error, tag constants, labels), grep for an existing one and reuse or extract it.
- [ ] A payload one component writes and another reads (plugin to plugin, backend to frontend) has a contract test.

### Tests
- [ ] Wiring: a new helper is exercised through the real entry point (job, fetch function, route), so removing the call turns a test red. A helper tested only in isolation does not count.
- [ ] Every role-restricted route has a test that each lower role is refused (status and body), including the role just below the gate, paired with the allowed role succeeding.
- [ ] Every timeout, deadline, early-exit, give-up and failure branch has a test that reaches it, with data where the exit is not also the natural end of the input.
- [ ] Each guard is proven red on its own; one red test through one guard proves nothing about the others.
- [ ] Mocks do not encode the change's own assumption about an external API; mock shapes come from docs or recorded responses, and frontend mocks use the error shape the backend really sends.
- [ ] DB-backed tests fail loudly or report themselves as skipped when no database is present; they never pass silently or fall back to a dev database URL.
- [ ] When tuples become parallel arrays, the test data varies every paired column, so a misaligned or cross-joined result fails.
- [ ] Never write a test that pins known-bad behavior as intended.

### Logging
- [ ] Logs carry tenant and entity ids, not usernames, emails, connection strings or secrets.

### Branch readiness
- [ ] Lint and build against the target branch's current rules (rebase-check), not the merge-base's.
- [ ] Re-check migration numbers for collisions with the target branch. A migration squash does not rely on `CREATE TABLE IF NOT EXISTS` to upgrade partially migrated databases. Indexes on large tables are built without a long table lock. Code does not require a migration the deploy may not have run.
