# metro-architecture skill

An agent skill describing the core architecture of [Metro](https://github.com/react/metro), the JavaScript bundler for React Native, and the design principles behind it: the virtualized file system, in-memory resolution, transformation, the layered cache system and its invalidation model, incremental graph traversal, and cheap serialization.

It is written as the starting point for designing, reviewing or debugging changes to Metro core, and every claim is pinned to a public commit of `react/metro` (see the provenance note at the top of `SKILL.md`). When the code and the skill disagree, the code wins.

## Install

The repository root is the skill. Clone it into your agent's skills directory, e.g. for Claude Code:

```sh
git clone https://github.com/robhogan/metro-architecture-skill.git ~/.claude/skills/metro-architecture
```

## Layout

- `SKILL.md` — the pipeline, the seams table, nine principles, a design checklist and anti-patterns.
- `references/` — deep dives on the file map, resolution, transformation, caching, the incremental graph and server, and serialization and the runtime contract.
