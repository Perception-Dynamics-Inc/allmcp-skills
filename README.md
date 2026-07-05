# AllMCP Agent Skill

The official [Agent Skill](https://skills.sh) for [AllMCP](https://allmcp.co) — the universal
integration hub that gives your AI agent one MCP endpoint for CRMs, spreadsheets, ads
platforms, telephony, restaurant platforms, and project tools.

Install into 70+ agents — Claude Code, Codex, Cursor, Goose, and more — with one command:

```bash
npx skills add Perception-Dynamics-Inc/allmcp-skills
```

Pure markdown, no scripts — nothing executes at install time.

## What you get

One skill, **`allmcp`**, that teaches your agent the whole platform loop:

- Endpoint setup (`https://go.allmcp.co/mcp/`, `X-API-Key` / Bearer auth, multi-tenant URL params)
- Discover → connect → unlock → call: `list_providers`, reading `connect_hint`,
  `connect_provider` for every auth shape (API key, multi-field, OAuth2, basic)
- Why a tool that "doesn't exist" is actually hidden, and how `describe_category`
  (or `?full_catalog=true`) reveals it
- What each platform error means and what to do about it (`action_required`,
  needs-reconnecting, rate limits, quota pause)
- Quota etiquette — 20,000 free provider calls a month, discovery always free,
  `get_usage` before heavy jobs

## Where are the per-provider skills?

You don't install them — **the platform serves them live.** When your agent opens
a category with `describe_category(provider, category)`, AllMCP returns that
category's expert playbook (tool sequencing, unguessable enums, capability
boundaries) right in the tool result, always in sync with the tools actually
deployed. Shipping static copies here would only let them rot. The one skill in
this repo teaches your agent exactly that discovery loop.

## Getting started with AllMCP

1. Get an API key at [allmcp.co/login](https://allmcp.co/login) — free up to
   20,000 provider tool calls a month, no card required.
2. Point your MCP client at `https://go.allmcp.co/mcp/` with header
   `X-API-Key: <your key>` ([client setup guides](https://docs.allmcp.co/documentation/quickstart)).
3. Install this skill and ask your agent to connect a provider.

## Contributing

Found a mistake, or the skill contradicts what the platform actually does?
[Open an issue](https://github.com/Perception-Dynamics-Inc/allmcp-skills/issues)
or a PR — see [CONTRIBUTING.md](CONTRIBUTING.md).

## License

[MIT](LICENSE) © Perception Dynamics Inc.
