# Review lessons: language-agnostic (read for every change)

Part of the review-lessons catalog (see "Catalog and caps" in section 8 of `SKILL.md`). Tag = highest severity seen: [H] High, [M] Medium, [L] Low.

## 1. External calls: budgets, retries, caching

- [H] Classify each upstream error: transient (retry), permanent for one item (skip and count), fatal (auth, exhausted rate limit: abort the run). A per-item skip never swallows a fatal error.
- [H] 4xx is not "permanent" as a class: 408/409/423/425/429, connection reset and body-read timeout are transient. Decide per status; log the real cause (sanitized error chain), not "decode failed".
- [H] A swallowed per-item failure still travels up as a typed, `#[must_use]` flag in the result; callers deciding reset/retry/coverage read that flag.
- [H] Optional or cosmetic enrichment: zero or one short rate-limit retry; after the first rate limit, set a run-wide flag that skips remaining lookups.
- [M] Skip, truncation, page cap, shift or failed sub-fetch is reported as partial/incomplete, never clean success. When an upstream refuses (auth, permission, timeout), report "unknown", not zero.
- [M] Record "covered/searched" markers only for entities actually searched; a skipped, budget-cut or failed search never counts.
- [M] When a budget stops a loop part-way, rotate the start point between runs (or persist a cursor) so the tail is not starved.
- [M] A retry or restart gets its own budget. Run restart machinery only where it can improve the result (opt-in per caller); drop the first pass's rows before restarting.
- [M] Offset paging over a mutating set needs shift detection (overlap an anchor row, restart with own budget); a shift degrades to incomplete plus warning, never a hard failure with a misleading message.
- [M] Turning a fail-fast path into a retrying one: compute worst-case wall time (waves x retries x max wait) and check every other caller of that function.
- [M] Don't auto-retry timeouts, non-transient 4xx or load-shed 429s on interactive paths; timeout plus retry multiplies wait.
- [M] Concurrency caps limit in-flight requests, not rate: add shared cooldown, honor `Retry-After` and rate-limit headers.
- [M] Wait and budget bounds are wall-clock (not only sleep time) and fit inside the enclosing job deadline.
- [M] Every HTTP client has a connect timeout as well as a request timeout; fallback/live-read paths have a short timeout and no default retries.
- [M] Each retried request is charged against the local quota gate (check inside the retry closure, not once per item). Make the shared entry point charge, not each caller.
- [M] Reserve a share of a shared upstream budget for interactive use; one low-privilege user must not drain the budget sync depends on (per-user bucket).
- [M] Don't call for data a sibling fetch in the same run already returns; order steps so the cheap source seeds the cache first.
- [M] Cache successful lookups of near-immutable data across runs (TTL, per-tenant key, size cap, successes only). Give upstream failures a short negative cache.
- [M] Request only the cheapest option set the consumer reads; pin the exact option list in a test.
- [M] Batching many items into one request amplifies one bad item into a whole-batch failure: validate each item (allowlist plus length cap, same check at save and sync) before batching.
- [M] Run batch chunks sequentially and stop at the first auth error (one bad credential costs one request, not one per chunk).
- [M] Estimate steady-state call growth (repos x open items x calls) before adding a per-item lookup; prefer a coarser list endpoint.
- [L] Build the HTTP client once and reuse it; resolve shared lookups once per sync, not once per pass.
- [L] Cap response body size when reading.

## 2. External calls: failure handling

- [H] A failed lookup must not overwrite or blank a previously stored value (links, titles). Keep the old value or surface the failure.
- [M] A failure in one sub-fetch must not skip independent siblings (pushes vs PR lists); a failed list must not discard lists that already succeeded.
- [M] On a fatal error in a concurrent fan-out, stop at the first fatal result; persist rows already in hand, then return the error. Never await slow sinks while the stream is live.
- [M] Save results incrementally per unit so a deadline or abort does not discard the whole run; report persisted counts even after a later fatal error.
- [M] Decide explicitly whether a partial run counts as success (advance cursor, run follow-up steps). A failed tick must not also skip unrelated work.
- [M] One failed or skipped entity (user, repo, project) must not reset state for the whole source; scope the reset to the entity.
- [M] Upstream field types come from docs or a recorded response; when uncertain, decode both shapes (untagged enum) and validate numeric strings.
- [M] When a payload shrinks, restore every field consumers read from the same lookup; keep fields that describe "now" (state, merged) out of historical rows.
- [M] When correctness depends on an upstream search index or operator, prove it on more than one account and server version, state the minimum version, warn when every item resolves to unknown.
- [M] Replacing a direct lookup with a search: pin the search's implicit filters (active-only, visibility, limits).
- [M] An input allowlist in front of an external system matches what that system accepts, or a safe superset.
- [L] Map every upstream status to a structured error variant, never prose matched by substring. Ambiguous statuses (203 on throttle, 404 for permission denied) get a documented, tested mapping.

## 3. Bounds and performance

- [H] A request deadline must not cancel in-flight fetches already paid for: run the fill detached and time out only the wait, so the next read hits the cache.
- [M] Every list call pages to the end or has an explicit, logged cap; every continuation-token loop has its own max-pages and repeated-token guard. A fixed `$top` without paging silently truncates.
- [M] Bound cost by the age/size of requested data, not range length: cap how far back `from` may go, add a statement timeout and a per-user/tenant concurrency limit on heavy reads.
- [M] Filter out rows that will be discarded (drafts, withdrawn) before expensive upstream work.
- [M] Multiple blocks measured under one shared deadline concurrently, never sequentially with a deadline each.
- [M] A per-request concurrency cap doesn't cap the tenant: add a per-tenant semaphore around upstream work when a page fires N requests.
- [M] Destructive or maintenance routes: rate limit before the expensive scan, statement/lock timeout, dry-run, second-run guard, documented removal condition.
- [L] Cache eviction sweeps run only when full or at an interval, not O(N) under a write lock on every insert.
- [L] Per-tenant caps on shared global caches (noisy neighbor); per-entity client maps evict on tenant/source removal.
- [L] Invalidate expensive aggregates after the data they depend on changes. Expect bloat after bulk deletes.

## 4. Correctness and data integrity

- [H] Stored timestamps come from the item's own event data, never from sync/observation time or `min(event, now)`.
- [H] A row type chosen from the item's current status and used in the dedup key makes output depend on when the sync saw the item: key on stable identity, derive status separately.
- [H] Retry/backoff clocks advance only on real attempts; failed, skipped or swallowed ticks neither reset nor age them.
- [H] Checked arithmetic on user-supplied dates and numbers, in the shared helper, on every path (admin, manual sync, plugins).
- [M] Every path that changes a state (create, enable, re-enable, link, role change) triggers the same side effects. If a concurrent run blocks it, queue or retry; never drop silently.
- [M] A toggle or config write during a running job must not reset that job's markers: generation counter or compare-and-set.
- [M] Model a multi-column state machine as one pure transition function plus one guarded UPDATE; enumerate and test every reachable state. Put it in the domain/service layer where jobs can reuse it.
- [M] One rule derived in several places (approval, merge time, own-item check) is written once; precedence differences make two screens disagree.
- [M] A repair/dedupe that deletes a duplicate verifies the kept counterpart is real and readable, not fabricated or derived.
- [M] Match grouped results back to parents by key, never by position or zip.
- [M] Deletion policy is consistent: if deleting a parent is refused for closed records, cascading from another owner must not erase them. Check what `ON DELETE CASCADE` reaches.
- [M] Mutations checked for multi-tenant and multi-membership combinations; one global status flag must not activate or block sibling memberships.
- [M] Derived "ongoing/waiting" metrics consult the entity's current state, not only the last transition.
- [M] Carrying state across ordered history: enforce or assert the order; take the latest value by timestamp, not arrival order.
- [M] A stored field that changes meaning gets a comment where it is built, and a note that old rows need a re-sync.
- [M] Two fields encoding one fact merge into one type; one flag for two causes splits per cause with its own wording.
- [M] Store coverage as an absolute timestamp, not relative days. If a view offers a wider range than collected, return the data horizon.
- [M] Identity matching: exact hit first, else a single case-insensitive hit; 2+ hits are ambiguous. Never first-wins. Tightening attribution: release note, cause-separated logs, check the dedup key includes the actor.
- [M] A fetch window follows the query semantics the feature needs (closed in window, still open regardless of age); test with an old item active inside the window.
- [M] Same class of failure, same status (state conflict is 409 everywhere, not 400). Domain errors the frontend must distinguish carry a machine-readable code.
- [L] Delete guards for unreachable states and dead fields.
- [L] Invalid persisted values must not break a whole aggregate; guard per row and count bad rows visibly.
- [L] Treat null, wrong-typed or string-number fields in external JSON as defined cases and test them.
- [L] Clamp "since"/retention boundaries so old open items are not re-inserted every sync.
- [L] Replace positional bool/Option params with an enum or options struct; repeated predicates (`Closed | Cancelled`) become a method; fetch an entity once per request.
- [L] Parallel implementations behave and log consistently on bad input. Rotation/bucket logic tested on both parity cases and wrap-around.

## 5. Concurrency

- [M] Hardening one writer: inventory every writer of rows referencing the same parent; guard all through one shared helper; test each.
- [M] Per-tenant state has a single owner and one invalidation entry point; don't make callers remember two `forget` calls.
- [M] In-process caches and limiters assume a single replica: enforce it, or share state and broadcast invalidation.
- [L] Warn-once and first-time flags are process-global; per-tenant behavior needs per-tenant state.

## 6. Security

- [M] Stored secrets are bound to the endpoint they were saved for: when base URL or TLS flag changes, require the secret again (backend and frontend).
- [M] Routes live-reading upstream entities by caller-supplied key with the service credential restrict keys to stored activity or an allowlist.
- [M] Rate-limit and shed per user, not per tenant.
- [M] A scope-filter fix repairs already-stored out-of-scope rows; every read route applies the same filter through one chokepoint.
- [M] Cancel/revive of privileged invites requires the privilege needed to issue them. Admin-only by default; confirm when a broader role can change org-wide objects.
- [M] Untrusted upstream text into LLM prompts goes through the sanitizer at every sink, quotes and pipes escaped, length capped locally.
- [L] Validate and percent-encode path segments built from external ids (no `.`/`..`).
- [L] Reject userinfo in configured URLs; warn on `http://` with token auth; recheck redirects against the host allowlist.
- [L] Strip request URLs from transport errors shown to low-privilege users; generic messages for config validation failures.
- [L] Sanitize untrusted strings before log fields (log forging). Secret redaction in logged paths is allowlist/route-pattern based.
- [L] No DB URLs with passwords on process command lines; reject args with leading `-`.
- [L] Destructive admin actions and on-behalf-of crediting record the acting account.
- [L] Secret-bearing config is not logged or bound on every tick; compare by hash or generation.

## 7. Logging

- [M] Log tenant and entity ids, not emails, usernames, names, free text, repo/project names or connection strings. Wrap tenant work in a span carrying the tenant id.
- [M] Fan-out failures: one aggregated `warn!` per call, per-item detail at `debug!`.
- [M] Shift/incomplete conditions log at warn with the reason.
- [L] Best-effort failures log at the same level as peers (`warn!`), say what is lost; one event, one line, right level; failures never at success level.
- [L] Truncate upstream error text in logs. Test helpers never print credentials.

## 8. Single source of truth

- [M] No hand-kept list mirroring an enum or registry; coverage guards enumerate the real registry and assert registry id equals the enum string.
- [M] A guard proving a fixture row exists is not a guard that each participant's test exists.
- [M] Grep for an existing mapping/helper before writing one; extract at the second copy, never add the 7th.
- [M] Types every plugin/producer must fill live in a typed struct in the shared contract crate, not in comments.
- [M] A payload with domain rules shared across components: persist the derived value in the producer, or add a contract test from real producer to real consumer.
- [M] User-visible fallback text derived from one state is written once and imported.
- [M] New endpoints get their own typed response; no ill-fitting schema reuse.
- [M] Two entry points of one component share one setup helper (parse, validate, allowlist, client).
- [L] Shared SQL guard fragments are one parameterized helper doing one job. Constants shared by prod and demo data are defined once.
- [L] Every cross-language wire enum has a golden fixture. Type-level dependencies don't invert layers.
- [L] Named key types (`type PrKey = (String, i64)`); repeated large struct literals get a constructor or `Default` with `..`.

## 9. Architecture

- [M] Services don't import API DTOs and API modules don't import services back; services return their own structs, one entry point for handlers and jobs.
- [M] New policy logic goes in its own module with pure decision functions; don't grow huge files or 500-line functions; split 100+ line handlers.
- [M] No API parameters without a client.
- [M] A flag that only produces a log line is documented as such, or wired into the result.
- [L] A repository doesn't write tables owned by another; compose in one transaction via the owner or a service.
- [L] `pub(crate)` by default; explicit imports, no deep `use super::*` in guard modules.
- [L] Test-only helpers stay out of production contract crates.
- [L] Rename things when their meaning changes; remove unused impls, dead fields, leftover wrappers.
- [L] Merge one branch's migrations into one before merge.

## 10. Tests

- [H] Wiring: exercise a new helper through the real entry point, with seeded data where the guard matters; reverting the call turns the suite red.
- [H] Every role-gated route: table-driven test that each lower role gets 403 with state unchanged, including the role just below the gate, paired with the allowed role succeeding.
- [H] Deadline, timeout, skip, early-exit and give-up branches reached by paused-clock tests, with data where the exit is not the natural end; multi-tick tests prove a cut-off entity is eventually reached.
- [H] Never pin contested or known-bad behavior as intended; settle the decision first or label the test.
- [M] Each failure class gets its own case (401, 403, 404, 429, 500, undecodable, dropped body); both sides of a classification (quota 403 retried, permission 403 not).
- [M] Accumulators and flags (`+=`, `|=`): a test where an earlier item sets the flag and a clean item follows.
- [M] A failing sink/persist step is surfaced, not swallowed by `unwrap_or(0)`.
- [M] Assert which entity was acted on and exact values (keys, author, links, request counts), not that something ran.
- [M] Absence assertions that constraints already guarantee are vacuous; don't count them.
- [M] A concurrency cap is tested on every method path that takes the slot.
- [M] Mocks don't encode the change's own API assumption; frontend mocks use the backend's real error shape and status.
- [M] Test the client as production builds it (timeouts, retries); pin its literals.
- [M] Timeout paths assert the setting is applied and the mapped status (504), not elapsed time.
- [M] Assert wire field names and both values of a response flag.
- [M] Cache invalidation on a write is tested (entry gone), with a sibling tenant's entry surviving.
- [M] Regroup/zip logic tested with unequal sizes and reversed ids. Predicate edge cases (null, wrong type, missing key) each paired with a presence control.
- [M] Shell scripts handling URLs tested with query strings, missing env, unreachable DB, zero rows.
- [L] Paused/injected clocks or `Notify`/channels; no real sleeps or `now()` re-reads. Every race/barrier wait has a timeout and aborts on failure.
- [L] Coverage-guard comparisons use multisets or counts.
- [L] Shared fixtures and builders hoisted into one helper; named-field structs over positional tuples.

## 11. Docs and comments

- [M] User docs, LLM prompts and UI notices state the rule the code applies; update all in the same change.
- [M] Changing a policy: grep for every stale comment and doc naming the old claim.
- [M] Documented limits and counts match computed values; shared-helper docs don't assume one caller.
- [L] Comments next to changed code still match it (e.g. "no lock needed" after adding a lock); no overstated guarantees.
- [L] A shortcut's comment states the assumption it relies on; deliberate omissions get a one-line comment.
- [L] Comments name things that exist, cite no finding IDs, carry no history (that goes to commits).
- [L] Behavior changes altering historical counts get a release note; published docs get broken tables and stale plans fixed.
