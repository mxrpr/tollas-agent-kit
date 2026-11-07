---
name: lessons-compact
description: Compact the global review-lessons store (quality-bar section 8 and the `review-lessons-<stack>.md` catalog) - merge duplicates, fix misplaced rules, bring every file under its cap, and list rules that could become lint or CI checks. Use when the user says "compact lessons", "clean up lessons", "/lessons-compact", when a cap is near, or about monthly.
argument-hint: "[--dry-run]"
---

# Lessons compact

Keep the global lesson store short enough that agents read it, without losing what it teaches.

Store:
- `~/.claude/skills/quality-bar/SKILL.md`, section 8 "Lessons from past reviews" (`- [ ]` lines). Caps and the catalog file list are in its "Catalog and caps" subsection.
- `~/.claude/skills/quality-bar/review-lessons-*.md` (lines tagged `[H]`, `[M]`, `[L]`).

With `--dry-run`, do steps 1-4 and report the proposed changes without editing anything.

## 1. Snapshot

1. Copy section 8's file and every catalog file to `/tmp/lessons-compact-<YYYY-MM-DD>/` before editing, and tell the user the path, so the change can be reverted.
2. Count rules: section 8 only, each catalog file, and each `##` section (`grep -c '^- \['` per file; per section by walking headings). Note which are over cap or within 10% of it.

## 2. Find what to change

Read section 8 and all catalog files in full, then list:

1. **Duplicates:** two or more lines stating the same pattern, within a file or across files. Keep the clearest wording, the highest severity tag, and any detail only one of them has.
2. **Near-duplicates to generalize:** several narrow rules that one broader rule covers without losing a concrete check (e.g. three "test the X branch" rules become one "test every Y branch, including X, Z" rule). The merged rule must still be checkable against a diff; never merge into a vague platitude.
3. **Misplaced rules:** a rule in `general` that only applies to one stack, or a stack file rule that is language-agnostic; a rule in the wrong `##` section.
4. **Section 8 consistency:**
   - Every section 8 rule has a matching `[H]` line in a catalog file. Add the line if it is missing.
   - Every `[H]` catalog line seen in two or more independent reviews is a candidate for section 8, if under cap.
5. **Stale or contradictory rules:** a rule another rule contradicts, or one that no longer makes sense. Do not remove these yourself: ask the user (step 4).
6. **Tooling candidates:** rules a machine could enforce (lint rule, clippy setting, test helper, CI check, schema check). List these only; do not implement them. When the user later adds the check, the rule can be deleted.

## 3. Rules for editing

- Never lose a High rule's substance: merge it, never drop it.
- Merge Low rules before Medium, Medium before High.
- Keep each line one imperative rule, general (no project names, paths, issue numbers, finding ids).
- Edit only section 8 of `SKILL.md` and the catalog files; never sections 1-7.
- After editing, every count is at or below its cap, ideally at or below 90%, to leave room for new lessons.
- Renumber `##` sections if any were merged or emptied; delete empty sections.

## 4. Confirm, apply, report

1. Before editing, show the user a terse plan: counts before → after per file; merges (old lines → new line); moves; proposed removals of stale or contradictory rules, which need their explicit yes. Apply only after the user approves (skip this on `--dry-run`: stop here).
2. Apply the edits and recount.
3. Report, terse:
   - Counts before → after: section 8 and each catalog file, against caps.
   - Merged: count, plus the new lines.
   - Moved: count.
   - Removed with approval: each line.
   - Tooling candidates: one line each, `rule → proposed check (lint/clippy/test helper/CI)`.
   - Snapshot path.
