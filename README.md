# my-custom-skill

Custom [Claude Code](https://claude.com/claude-code) skills. Each skill is
self-contained: its folder holds everything it needs, and no skill depends on
or points to another.

| Skill | Scope | Use when |
|---|---|---|
| `simplify` | the current diff, PR, branch, or uncommitted work | cleaning up changes you just made |
| `full-simplify` | the whole repo (or a path) | a codebase-wide cleanup of a small-to-medium repo |
| `map-simplify` | the whole repo (or a path), map-reduce | a codebase-wide cleanup of a large repo (~50k+ lines) |

All three review code from the same four angles (Reuse, Simplification,
Efficiency, Altitude) and are quality-only: they do not hunt for bugs.

## simplify

Diff-scoped cleanup. Gathers the diff (upstream range plus uncommitted
changes, or a PR/branch/path you pass), runs four review agents in parallel —
one per angle — then dedups their findings and applies the fixes, skipping
anything that would change behavior. Without a subagent tool it does a single
pass over all four angles and says so.

Claude Code ships a built-in `/simplify`; this is a standalone copy of that
recipe, so you can read it, change it, or use it in other harnesses. If you
install it into `~/.claude/skills/` alongside the built-in, rename the folder
and `name:` (e.g. `my-simplify`) to avoid the name clash.

## full-simplify

Whole-repo cleanup. Lists the tracked files (tests included), excludes
generated, vendored, and binary files, splits scopes larger than ~40 files into
review units along the repo's module boundaries, and runs the four angle agents
per unit (at most ~8–10 agents at once). Findings are deduped and applied
directly, skipping behavior changes and generated or vendored files.

## map-simplify

Map-reduce quality cleanup for large codebases. It splits the repo into
cohesive chunks built from the import graph, then:

- **Clone detection**: one run of a clone detector (jscpd) over all the code,
  split into copies within a chunk and copies across chunks.
- **Map**: two agents per chunk (Simplification and Efficiency) review every
  line and write a compact chunk card. Simplification also confirms that
  chunk's within-chunk clones.
- **Reduce**: two agents (Reuse and Altitude) read all the cards together,
  plus the cross-chunk clones, to find cross-chunk duplication and missing
  abstractions.

Findings are merged into one plan. Nothing is applied until you approve it,
and fixes are applied in batches that are reverted if the build or tests fail.
Quality only: it does not hunt for bugs.

For a small repo it offers a single-wave mode that skips chunking and runs the
four angles over the whole scope at once.

## Install

Copy any skill folder into your skills directory:

```sh
mkdir -p ~/.claude/skills
cp -r full-simplify map-simplify ~/.claude/skills/
```

Then run it in Claude Code, optionally with a path, e.g. `/map-simplify lib/foo/`.
