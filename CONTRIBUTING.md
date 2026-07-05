# Contributing

Thanks for caring about these skills — here's how changes actually land.

## This repo is generated — don't hand-edit skill content

Everything under `skills/` is authored, verified against the live AllMCP
platform, and maintained in the AllMCP backend repository. A sync job exports
it here whenever the source changes. That means:

- **PRs that edit `skills/**` directly will be closed** — not because we don't
  want the fix, but because the next sync would silently overwrite it.
- **The right path is an issue.** Report the mistake (wrong tool name, stale
  enum, missing gotcha, unclear recipe) in
  [GitHub Issues](https://github.com/Perception-Dynamics-Inc/allmcp-skills/issues).
  We fix it at the source, re-verify it against the live platform, and the fix
  syncs back here — usually within a day.

## What we gladly take PRs for

- `README.md` improvements (typos, clearer setup steps)
- Install/agent-compatibility notes

## Quality bar for skill content

Every skill in this repo follows the same discipline before it ships:

- Every tool name, parameter, and enum is verified against the live platform —
  no facts from memory.
- Capability boundaries are stated explicitly: a skill never promises an
  action the platform can't perform.
- Recipes are written as outcome-first playbooks with realistic data, not API
  reference dumps.
- Pure markdown only — no scripts, nothing executes at install time.

If a skill ever contradicts what the platform actually does, that's a bug —
please file it.
