---
name: full-simplify
description: Whole-repository quality cleanup — review every source file (tests included) from four cleanup angles — reuse, simplification, efficiency, altitude — then apply the fixes. Quality only; it does not hunt for bugs.
when_to_use: >
  Use when the user wants a codebase-wide quality cleanup rather than a review of
  recent changes — "simplify the whole repo", "clean up the codebase", "find
  duplication across the project", "full-repo simplify". Do NOT use for a diff, a
  PR, or uncommitted work; a diff-scoped cleanup is far cheaper.
argument-hint: "[<path or module to scope to>]"
arguments:
  - target
---

# Full-Repo Simplify

`whole repo → 4 cleanup agents per review unit → apply the fixes`

The whole repository is the review scope, not a diff.

You are improving the quality of the code, not hunting for bugs. Review it for
reuse, simplification, efficiency, and altitude issues, then fix what you find.
Do not look for correctness bugs — that belongs to a separate correctness review.

## Phase 0 — Gather the scope

Run `git ls-files` (restricted to the target path if one was passed) to get the
file set under review (outside a git repo, list files with Glob or `find` and
apply `.gitignore` rules by hand). Tests are in scope. Exclude generated output, vendored and
dependency directories (`node_modules`, `dist`, `build`, `vendor`, `target`,
`.venv`, `static/vendor`-style copies), binaries, fonts, images, lockfiles, and
test fixture data; say what you excluded. Treat the remaining files as the review
scope.

If the scope is too large for one agent to read (roughly more than 40 files),
group it into review units of related files following the repo's own module
boundaries, and run Phase 1 once per unit. Each unit gets its own 4 agents; keep
at most ~8–10 agents running at once (two units at a time) and start the next
unit as a slot frees up.

## Phase 1 — Review (4 cleanup agents per unit, in parallel)

For each review unit (or the whole scope when it is small), launch **4
independent review agents** via the subagent tool, the unit's four in a single
message so they run concurrently. Pass each agent the file set (or unit)
and one of the four angles below. Each returns its findings with `file`, `line`,
a one-line `summary`, and the concrete cost (what is duplicated, wasted, or
harder to maintain).

### Reuse

Flag code that re-implements something the codebase already has — Grep
shared/utility modules and adjacent files, and name the existing helper to call
instead. Also flag duplication between modules and name which copy should
become the shared one.

### Simplification

Flag unnecessary complexity: redundant or derivable state, copy-paste with
slight variation, deep nesting, dead code. Only flag a symbol as dead after
grepping every reference across the repo, including dynamic ones. Name the
simpler form that does the same job.

### Efficiency

Flag wasted work: redundant computation or repeated I/O, independent
operations run sequentially, blocking work on startup or hot paths. Also flag
long-lived objects built from closures or captured environments — they keep the
entire enclosing scope alive for the object's lifetime (a memory leak when that
scope holds large values); prefer a class/struct that copies only the fields it
needs. Name the cheaper alternative.

### Altitude

Check that code fixes the root cause at the right depth rather than patching a
symptom with a fragile bandaid. Special cases layered on shared infrastructure
are a sign the fix isn't deep enough — prefer the simpler, more general change
to the underlying mechanism over adding special cases, and name that change.

## Phase 2 — Apply the fixes

Wait for all agents to complete, dedup findings that point at the same line or
mechanism, and fix each remaining one directly. Skip any finding whose fix would
change intended behavior, require changes well outside the reviewed scope, or
that you judge to be a false positive — note the skip rather than arguing with
it. Never edit generated, vendored, or ignored files. Finish with a brief summary
of what was fixed and what was skipped (or confirm the code was already clean).

## If the subagent tool is unavailable

When no subagent tool is available in this context, the 4-agent fan-out can't
run. Work through all four angles yourself, in this same context, one review
unit at a time — do not skip an angle for lack of fan-out. Collect findings in
the same format, then apply Phase 2 unchanged. State clearly in your summary
that this was a single-pass review done without subagents, not the full 4-agent
fan-out, so whoever reads it isn't misled about what actually ran.
