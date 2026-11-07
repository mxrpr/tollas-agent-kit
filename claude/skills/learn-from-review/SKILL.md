---
name: learn-from-review
description: Turn the findings of a finished comprehensive review (`.full-review/` folder) into general lessons in the global quality-bar skill, so future code avoids them. Use after `comprehensive-review:full-review` finishes, or when the user says "learn from review", "record lessons", "/learn-from-review".
argument-hint: "[path to a worktree or .full-review folder; default: current repo root]"
---

# Learn from review

Record the reusable lessons of one comprehensive review in the global lesson store:

- `~/.claude/skills/quality-bar/SKILL.md`, section 8 "Lessons from past reviews": the most important rules (`- [ ]` lines).
- `~/.claude/skills/quality-bar/review-lessons-<stack>.md`: the full catalog, one file per stack (`general`, `rust`, `frontend`, `sql`), lines tagged `[H]`, `[M]` or `[L]`.

Never store lessons in the project.

## 1. Locate the review

1. Resolve the folder: the argument if given (a worktree path or its `.full-review/`), otherwise `$(git rev-parse --show-toplevel)/.full-review/`. Stop and say so if it does not exist or has no `05-final-report.md`.
2. If `.full-review/lessons-recorded` exists, read it. It holds the review dates already processed. If the final report has no round newer than those dates, say "already recorded" and stop.

## 2. Gather findings

1. Read `05-final-report.md` in full (every round). This is the primary source.
2. Measure the phase files: `*.md` except `05-final-report.md`; skip `*.log`, `*.txt`, `state.json`.
   - Under ~100 KB in total: read them yourself.
   - Larger: delegate to one `general-purpose` subagent (model `sonnet`) per ~200 KB batch, in parallel. Prompt: read those files and section 8 plus the catalog files above; return only lessons not already covered, one line each as `<stack> | <H/M/L> | <section> | <rule>`, no preamble.
3. Only rounds newer than the dates in `lessons-recorded` count.

## 3. Turn findings into rules

For each finding that reveals a reusable pattern (a coder could avoid it next time):

- Write one imperative rule about the general pattern, not the instance. No project names, file paths, issue numbers, finding ids, people or real identifiers: the store is shared by every project.
- Severity = the finding's final severity after the report's merges and downgrades.
- Skip one-off nits with no reusable lesson, and findings the report rejected.
- Stack: `rust`, `frontend` or `sql` when the rule only applies there; otherwise `general`. A lesson for a stack with no file may start a new `review-lessons-<stack>.md` (same header as the others); add it to the list in section 8 "Catalog and caps".

## 4. Deduplicate against the store

Read section 8 of `SKILL.md` and every catalog file you will write to. For each rule:

- Already covered: skip it, or sharpen the existing line if the new case adds something, or raise its severity tag if the finding was more severe.
- New: place it as follows:
  - High or Critical: a `- [ ]` line under the best-matching `###` subsection of section 8, AND an `[H]` line in the matching `##` section of the catalog file.
  - Medium or Low: an `[M]` or `[L]` line in the matching `##` section of the catalog file only.

## 5. Enforce the caps

Caps are defined in section 8 "Catalog and caps" of `SKILL.md`: section 8 at most 60 rules, each catalog `##` section at most 30, each catalog file at most 150.

Count before writing (`grep -c '^- \['` per file; count per section and in section 8 only). If a new rule would exceed a cap, do not append it. Instead merge it into the closest existing rule, or generalize two or more existing rules into one. Merge Low rules first; never drop a High rule. Recount after editing.

Edit only section 8 of `SKILL.md` and the catalog files. Never edit sections 1-7, and never reword other lessons beyond sharpening or merging.

## 6. Mark and report

1. Append the review date(s) just processed to `.full-review/lessons-recorded`, one per line.
2. Report to the user, terse:
   - Added: each new line, with its file.
   - Sharpened or merged: each changed line, before and after.
   - Skipped as already covered: count only.
   - Counts after the change: section 8, and each catalog file touched, against its cap.
   - If any count is above 90% of its cap, recommend running `/lessons-compact`.
