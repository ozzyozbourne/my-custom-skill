# my-custom-skill

Custom [Claude Code](https://claude.com/claude-code) skills.

## map-simplify

Map-reduce quality cleanup for large codebases. It splits the repo into
cohesive chunks built from the import graph, then:

- **Map**: two agents per chunk (Simplification and Efficiency) review every
  line and write a compact chunk card.
- **Reduce**: two agents (Reuse and Altitude) read all the cards together,
  plus clone-detector results, to find cross-chunk duplication and missing
  abstractions.

Findings are merged into one plan. Nothing is applied until you approve it,
and fixes are applied in batches that are reverted if the build or tests fail.
Quality only: it does not hunt for bugs.

### Install

```sh
mkdir -p ~/.claude/skills
cp -r map-simplify ~/.claude/skills/
```

Then run `/map-simplify` (optionally with a path, e.g. `/map-simplify lib/foo/`)
in Claude Code.
