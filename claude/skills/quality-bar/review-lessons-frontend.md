# Review lessons: frontend (React/TypeScript)

Part of the review-lessons catalog (see "Catalog and caps" in section 8 of `SKILL.md`). Tag = highest severity seen: [H] High, [M] Medium, [L] Low.

## 1. Frontend

- [M] Charts aggregate (e.g. weekly) once a range would render thousands of nodes or sub-pixel bars; disable animation for big series.
- [H] Changed UI lines use semantic tokens only (no palette classes, `dark:` color variants, hex, title case); exemption lists only shrink. Lint on a branch rebased onto the target.
- [M] After a mutation, invalidate every query showing data derived from it, not only the edited list; test it.
- [M] Edit dialogs surface 409/validation errors like their add siblings; toasts name the real cause.
- [M] Fixing an empty-string-means-missing bug (`||` not `??`) for one field: fix every sibling field and renderer read the same way.
- [M] Keyboard hooks inside a dialog coordinate with its Escape handler (`onEscapeKeyDown`); test the primitive inside a dialog.
- [M] One accessible model per chart (don't wrap a focusable accessibility layer in `role="img"`).
- [L] Types match the wire shape including `null`; Zod schemas parse the fields the page reads (no `as unknown as T`).
- [L] Infinite queries: exclude `isFetchNextPageError` from page-level error states.
- [L] Retry controls don't retry non-retryable errors; 429 shows reason and a retry path.
- [L] Empty state and chart derive from the same data. Filter changes cancel their earlier in-flight queries.
- [L] Group and label dates in UTC; time-axis labels spanning years include the year. Heading levels follow nesting.
- [L] Job-watch state lives in a store or is derived from the API, not component state.
- [L] Exhaustive maps use `Record<Id, ...>` with no fallback cast; registry keys equal manifest ids.

## 2. Frontend tests

- [L] Ordering assertions on already-ordered fixtures are tautologies: serve reversed. `waitFor(() => expect(x).toEqual([]))` passes on the first tick; wait on a settled signal.
- [L] Test names don't overclaim; comments state behavior, not review finding IDs. Hand-mirrored fake servers share a JSON fixture with the real one.
