# Contributing

Issues and PRs are welcome — this repo holds the `allmcp` skill and its docs.

## Quality bar

The skill is our public voice to every agent, so every change must hold the
same discipline:

- **Every fact matches the live platform.** Tool names, auth shapes, quota
  numbers, and error behavior are verified against production before merge —
  nothing lands from memory.
- **No secrets, no internal references.** Placeholder credentials only
  (`allmcp_...`), public URLs only.
- **Imperative instructions to the agent, not documentation for humans.**
  One default per job, boundaries stated explicitly, no option menus.
- **Pure markdown.** No scripts, nothing executable — installs must stay
  all-green on security scans.
- **Lean.** Every line is a recurring token tax in the agent's context.
  If a fact lives better in the platform's own tool descriptions or
  `describe_category` playbooks, it doesn't belong here.

## What to report

If an agent following the skill word-for-word fails against the live
platform, that's the highest-value bug you can file — please include the
tool calls it made and what came back.
