---
name: map-simplify
description: Map-reduce quality cleanup for large codebases — split the repo into cohesive chunks, review each chunk line by line for local cleanups (map), then review all chunk summaries together for cross-chunk duplication and missing abstractions (reduce), and apply approved fixes in verified batches. Quality only; it does not hunt for bugs.
allowed-tools:
  - Bash
  - Read
  - Edit
  - Write
  - Grep
  - Glob
  - Agent
  - AskUserQuestion
when_to_use: >
  Use when the user wants a codebase-wide simplification of a repo too large for
  one review wave (roughly 50k+ hand-written lines) — "map-simplify", "simplify
  this huge repo", "chunked cleanup", "find duplication across the whole
  codebase". Small repos get a cheaper single-wave mode (offered in Step 2);
  for a diff, PR, or uncommitted work a diff-scoped cleanup is far cheaper.
argument-hint: "[<path or module to scope to>]"
arguments:
  - target
---

# Map Simplify

You are improving the quality of a large codebase, not hunting for bugs. Find
duplication, accidental complexity, wasted work, and bandaid-depth fixes, then
fix what the user approves. Do not look for correctness bugs — that belongs to
a separate correctness review.

This is a four-angle cleanup (Reuse, Simplification, Efficiency, Altitude)
scaled to a whole large repo as map-reduce:

- **Map (inner loop)** — the code is split into chunks small enough for an
  agent to read in full. Chunks are processed in waves; two agents review each
  chunk line by line and write a **chunk card**, a compact summary of it.
- **Reduce (outer pass)** — one card alone says little, but all cards together
  show how chunks relate. Two agents read every card at once and find what no
  single chunk can show: the same helper written twice, the same bandaid
  repeated across chunks. They open specific files the cards point at to
  confirm a finding.

Three rules shape everything below:

1. **Discover everything before changing anything.** Fixing chunks while the
   reduce pass is still pending wastes work: a cross-chunk merge can delete the
   code a chunk fix just polished.
2. **Match each angle to where it can see the problem.** Simplification and
   Efficiency are local (map); Reuse and Altitude are cross-chunk (reduce).
3. **Find duplication by search, not by grouping.** Two copies in distant
   chunks never share a review, so Reuse works from the combined helper
   inventory and a clone detector.

## Inputs

- `$target` (optional): a path, directory, or module to scope to. Free text —
  treat it as a hint to interpret, not a validated path. If omitted, the whole
  repo is in scope. Suggest a single feature slice as a pilot when the user has
  not run this skill on the repo before.

## Goal

Duplication, accidental complexity, wasted work, and bandaid-depth fixes
removed across the codebase — within chunks and between them — with
**behavior unchanged**, and with nothing applied that the user did not approve.

## Working files

Keep all intermediate state in a run directory outside the repo (the session
scratchpad if one exists, otherwise a temp directory), so the run survives
context compaction and nothing leaks into the working tree:

- `scope.txt` — in-scope files, and the exclusion list with reasons
- `chunks.json` — chunks: id, purpose, files, line count
- `clones.json` — clone pairs, split into within-chunk (per chunk) and
  cross-chunk
- `cards/<chunk-id>.md` — one card per chunk
- `findings/<chunk-id>.json` — deduped map findings
- `findings/reduce.json` — cross-chunk findings
- `plan.md` — the ranked, approved plan and each batch's outcome

## Steps

### 1. Establish scope

- `git ls-files` (restricted to `$target` if given) for tracked files.
- Exclude, and record why: generated code (codegen output, protocol/ORM
  clients, `*.g.dart`, `*.pb.*`, files with a "generated — do not edit"
  header, paths marked `linguist-generated` in `.gitattributes`), bundled or
  minified build output (a single huge file of dense one-line code is a strong
  signal — check its first lines), vendored and dependency directories,
  lockfiles, license dumps, binaries, and assets.
- Read the repo-root `CLAUDE.md`, `AGENTS.md`, and `README` if present, for
  intended architecture and any rules about what must not be touched.
- Count hand-written lines per language and per top-level directory.

If the in-scope code is small enough for a single wave (roughly under ~40
files or ~10k lines), tell the user map-reduce is overkill and offer
**single-wave mode** as an option in Step 2's confirmation: skip chunking,
cards, the clone split, and Step 4; in Step 3 launch **4 agents** over the
whole scope at once — Simplification and Efficiency as written, plus Reuse
and Altitude using their Step 4 instructions against the code itself (Grep
for existing helpers and duplicated logic instead of reading cards) — then
continue with Steps 5–7. If they invoked `/map-simplify` by name, keep the
full map-reduce as the default.

**Success criteria**: `scope.txt` written; the user is told the hand-written
line count and exactly what was excluded and why.

### 2. Split into chunks

Build chunks from the **import graph**, not from folders alone. Many repos are
organized by layer (`widgets/`, `services/`, `models/`), and a feature cuts
across all of them — a chunk per layer folder mixes unrelated features and
splits related ones.

- Extract imports for every in-scope file with Grep (e.g. `^import`,
  `^from .* import`, `require(`, `#include`). Resolve them to in-scope files.
- Cluster files that import each other heavily into feature slices: a state
  holder plus the views that render it plus the services and models only it
  uses. Pair cross-language halves of one mechanism (a platform channel and
  its native handler, a client and the server endpoint it calls) in one chunk.
- Target **~2–6k lines per chunk**. A single file above ~4k lines is its own
  chunk. Shared utilities used everywhere form their own chunk(s).
- Every in-scope file lands in exactly one chunk. Give each a one-line purpose.

Then run the **clone detector once over all in-scope files** — never per
chunk, since the most valuable clones span chunks. If one is available or
installable without touching the repo (e.g. `npx jscpd <dirs> --min-tokens 60
--reporters json --output <run dir>/jscpd`, ignoring excluded paths), map each
clone pair's two sides to their chunks and split the results in
`clones.json`:

- **Within-chunk pairs** (both sides in the same chunk) → that chunk's
  Simplification agent in Step 3.
- **Cross-chunk pairs** → the Reuse agent in Step 4.

If no detector can run, record that; Step 7 reports it as an angle that did
not run.

Show the user the chunk list with the estimated agent count (2 per chunk + the
reduce agents) and use **AskUserQuestion** to confirm: run all, run a pilot
chunk first, or adjust the split (plus "single-wave mode" when Step 1 found
the repo small).

**Success criteria**: `chunks.json` covers every in-scope file exactly once;
the user approved the split and the run size.

### 3. Map — review each chunk

**Execution**: Agent

Process chunks in waves of no more than ~8–10 concurrent agents (about 4–5
chunks per wave), finishing a wave before starting the next, until every chunk
is done. For each chunk launch **2 agents** — Simplification and Efficiency —
giving each the chunk's file list, its purpose, and the repo map, and giving
the Simplification agent that chunk's within-chunk clone pairs from
`clones.json`. Both read every line of the chunk.

Each agent returns findings as `file`, `line`, one-line `summary`, and the
concrete cost (what is wasted or harder to maintain). The Simplification agent
also writes the **chunk card** (below).

#### Simplification

Flag unnecessary complexity within the chunk: redundant or derivable state,
copy-paste with slight variation, deep nesting, needless indirection. Name the
simpler form that does the same job. Confirm each within-chunk clone pair
you were given by reading both sides, and report the real ones as
copy-paste findings; the detector only sees exact token matches, so keep
looking for near-copies it missed. Dead code may only be flagged after
Grepping the whole repo for every reference, including dynamic ones (string
names, reflection, config, routes, generated call sites, native-side callers
of channel methods). If reachability cannot be proven, do not flag it.

#### Efficiency

Flag wasted work: redundant computation or repeated I/O, independent
operations run sequentially, blocking work on startup or hot paths, rebuilding
or re-rendering more than needed. Also flag long-lived objects built from
closures or captured environments — they keep the entire enclosing scope alive
for the object's lifetime (a memory leak when that scope holds large values);
prefer a class/struct that copies only the fields it needs. Name the cheaper
alternative. If a hot path clearly continues into another chunk, note the
callee so the reduce pass can trace it.

#### Chunk card

At most ~60 lines, written to `cards/<chunk-id>.md`:

- **Purpose** — one or two sentences.
- **Public surface** — types, functions, and entry points other chunks use.
- **Helper inventory** — every general-purpose helper the chunk defines
  (formatting, parsing, retry, path handling, validation, mapping, logging
  wrappers…) with its signature and `file:line`.
- **Patterns** — how the chunk does recurring things (error handling, state
  updates, async/cancellation, persistence, platform branching).
- **Special cases** — every branch keyed on a specific provider, platform,
  mode, ID, or type name, with `file:line`.
- **Dependencies** — chunks it imports and chunks that import it.

After each wave, the main model **dedupes** that wave's findings per chunk:
merge findings pointing at the same line or mechanism, keep the clearest
wording and the union of sites, and write `findings/<chunk-id>.json`.

**Success criteria**: Every chunk has a card and a deduped findings file.
Report the finding count per chunk.

### 4. Reduce — review across chunks

**Execution**: Agent

Launch **2 agents** in one message, each given **every** chunk card and
permission to Read specific files the cards point at. Neither reads the whole
codebase.

#### Reuse

1. Hand the agent the cross-chunk clone pairs from `clones.json` (Step 2).
   It confirms each by reading both sides.
2. From the combined **Helper inventory** sections, find helpers that do the
   same job in different chunks, and Grep for inline re-implementations of
   helpers that already exist.
3. For each duplication cluster, name which copy should become the shared
   version, where it should live, and every site that should call it.

#### Altitude

From the combined **Patterns** and **Special cases** sections, check that
behavior is implemented at the right depth, not as bandaids: special cases
layered on shared infrastructure mean a fix wasn't deep enough — prefer
generalizing the underlying mechanism, and name that change. A special case
recurring in three or more places is a missing abstraction: report the
cluster as one finding. Also flag the same pattern implemented differently in
different chunks, and trace any cross-chunk hot paths the Efficiency agents
noted.

#### When the cards don't fit

If the combined cards are too large for one agent (roughly 3,000+ lines —
usually only past a few hundred chunks), split them into groups of related
chunks by dependency, run Reuse and Altitude per group, then run one final
pass of each over the group reports. Do not group otherwise — every extra
level hides relationships between chunks in different groups.

The main model dedupes and clusters the reduce output into
`findings/reduce.json`.

**Success criteria**: Reuse and Altitude each saw every card, and cross-chunk
findings are written.

### 5. Plan and approve

Merge map and reduce findings into one plan:

- **Resolve conflicts**: drop or rewrite any chunk finding whose code a
  cross-chunk finding moves, merges, or deletes. Note each such drop.
- **Cluster** the same underlying problem across chunks into one finding that
  names every site.
- Drop anything whose fix would change intended behavior, touch excluded
  files, or that you judge a false positive — note the drop.
- **Order** for application: cross-chunk restructuring first (shared helpers,
  new abstractions, moved special cases), then local simplification, then
  efficiency. Within that, rank by value.
- Group into **batches** of related findings that can be verified together.

Write `plan.md`, present the ranked list to the user, and use
**AskUserQuestion** to confirm which findings to apply — all, a subset, or
none. Apply nothing before this answer.

**Success criteria**: A ranked, conflict-free, deduplicated plan where each
entry names every site, and an explicit user decision on what to apply.

### 6. Apply in verified batches

Before editing, detect the project's build/test/lint/analyze commands
(`package.json`, `pubspec.yaml`, `Makefile`, `Cargo.toml`, `pyproject.toml`,
CI config, build scripts, `CLAUDE.md`/`AGENTS.md`) and run them to record a
baseline. If the suite already fails, or there is none, say so plainly and
continue only if the user accepts the missing safety net.

Work the approved plan in order, one batch at a time. For each batch: apply
the fixes, re-run the baseline commands, and if they fail, revert **that
batch** and record it as skipped in `plan.md`. Do not chase the failure — a
failed check means the change wasn't behavior-preserving, which is out of
scope here. Before a later batch touches code an earlier batch changed,
re-read the current file rather than trusting the finding's line numbers.

**Success criteria**: Every batch either applied-and-verified or
reverted-and-recorded. The verification suite ends in the state it started.

### 7. Report

Summarize:

- Scope: hand-written lines and files reviewed, chunks, and agents run.
- Fixed, grouped by cluster, with sites touched — cross-chunk first.
- Skipped, and why (failed verification, conflict, behavior change, declined).
- **Never looked at**: excluded files, chunks the user declined or deferred,
  and angles that could not run (e.g. no clone detector). An unreviewed area
  must never read as clean.
- Correctness bugs noticed in passing, listed but not fixed — point the user
  at a separate correctness review.

**Success criteria**: The user can tell exactly what changed, what didn't, and
what this run never examined.

## Hard constraints

- **Behavior must not change.** Every fix is a refactor. If a change alters
  observable behavior, skip it and report it.
- **No bug hunting.** Correctness defects go in the report, never get fixed here.
- **No edits before Step 5 approval.** Steps 1–4 are read-only on the repo.
- **Never apply anything the user has not approved.**
- **Never flag dead code without repo-wide proof** of unreachability.
- **Never edit generated, bundled, vendored, or ignored files.**
- **Never write run state into the repo.** Use the run directory.
