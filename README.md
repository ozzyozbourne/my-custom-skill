# my-custom-skill

Custom [Claude Code](https://claude.com/claude-code) skills. Each skill is
self-contained: its folder holds everything it needs, and no skill depends on
or points to another.

| Skill | Scope | Use when |
|---|---|---|
| `simplify` | the current diff, PR, branch, or uncommitted work | cleaning up changes you just made |
| `full-simplify` | the whole repo (or a path) | a codebase-wide cleanup of a small-to-medium repo |
| `map-simplify` | the whole repo (or a path), map-reduce | a codebase-wide cleanup of a large repo (~50k+ lines) |
| `code-review` | the current diff, PR, branch, or path | hunting correctness bugs (plus cleanups) |
| `security-review` | the pending changes on the current branch | hunting exploitable security vulnerabilities |

The three simplify skills review code from the same four angles (Reuse,
Simplification, Efficiency, Altitude) and are quality-only: they do not hunt
for bugs. `code-review` hunts bugs; `security-review` hunts vulnerabilities.

`simplify`, `code-review`, and `security-review` are standalone copies of
Claude Code built-ins of the same names, taken from the Claude Code binary, so
you can read, change, or reuse them in other harnesses. If you install one into
`~/.claude/skills/` alongside the built-in, rename its folder and `name:`
(e.g. `my-code-review`) to avoid the name clash.

## simplify

Diff-scoped cleanup. Gathers the diff (upstream range plus uncommitted
changes, or a PR/branch/path you pass), runs four review agents in parallel —
one per angle — then dedups their findings and applies the fixes, skipping
anything that would change behavior. Without a subagent tool it does a single
pass over all four angles and says so.

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

## code-review

Bug-focused review of the current diff (or a PR, branch, or path) at an effort
level — `low`, `medium` (default), `high`, `xhigh`, or `max`:

- **low** — one pass over the diff hunks, no subagents, up to 4 findings.
- **medium / high** — 8 finder agents (3 correctness angles, Reuse,
  Simplification, Efficiency, Altitude, CLAUDE.md conventions), then one
  verifier per candidate. Medium favors precision (up to 8 findings), high
  favors recall (up to 10).
- **xhigh / max** — 10 finder agents (adds language pitfalls and
  wrapper/proxy correctness), verify, then a sweep for gaps; up to 15.

Flags: `--fix` applies the findings, `--comment` posts them to the GitHub PR or
GitLab MR, `--max-findings <n>|all` changes the cap. Its only
security-related check is SQL injection in the language-pitfall angle; use
`security-review` for a security pass.

## security-review

Security-focused review of the branch's pending changes against `origin/HEAD`.
A sub-agent hunts high-confidence, exploitable vulnerabilities (injection,
auth bypass, crypto and secrets, deserialization/RCE, XSS, data exposure);
then one sub-agent per finding filters false positives against a list of hard
exclusions and precedents, and only findings with confidence 8+ are reported,
as a markdown report.

## Install

Copy any skill folder into your skills directory:

```sh
mkdir -p ~/.claude/skills
cp -r full-simplify map-simplify ~/.claude/skills/   # plus any renamed built-in copies
```

Then run it in Claude Code, optionally with a path, e.g. `/map-simplify lib/foo/`.
