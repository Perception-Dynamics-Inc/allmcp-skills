# AllMCP Agent Skills

Expert-level [Agent Skills](https://skills.sh) for [AllMCP](https://allmcp.co) — the universal
integration hub that gives your AI agent one MCP endpoint for CRMs, spreadsheets, ads
platforms, telephony, restaurants, and project tools.

Install into Claude Code, Codex, Cursor, Goose, and 70+ other agents with one command:

```bash
npx skills add Perception-Dynamics-Inc/allmcp-skills
```

Pure markdown, no scripts — nothing executes at install time.

## Why

Agents work dramatically better with AllMCP when they carry its workflow guides:
which tool to call first, which values are unguessable, what each provider can
and cannot do. These skills are the same playbooks the AllMCP team ships inside
the platform, packaged in the standard Agent Skills format.

If you install only one skill, install **`allmcp`** — it teaches the whole
platform loop (discover → connect → unlock → call). Add a provider skill for
each app you actually use.

## Skills

| Skill | What it covers |
|---|---|
| [`allmcp`](skills/allmcp) | The platform loop: endpoint setup, connecting providers, unlocking hidden tools, error handling, quota etiquette |
| [`allmcp-bitrix24`](skills/allmcp-bitrix24) | Bitrix24 CRM & business platform — CRM, tasks, calendar, chats, drive, telephony, and more |
| [`allmcp-google-sheets`](skills/allmcp-google-sheets) | Google Sheets — values, formatting, tabs, charts, filters, protection, sharing |
| [`allmcp-google-docs`](skills/allmcp-google-docs) | Google Docs — find documents, read content, export text |
| [`allmcp-google-ads`](skills/allmcp-google-ads) | Google Ads — accounts, campaigns, keywords, budgets, metrics, diagnostics |
| [`allmcp-amocrm`](skills/allmcp-amocrm) | amoCRM — leads, contacts, pipelines, tasks, catalogs, unsorted inbox |
| [`allmcp-kommo`](skills/allmcp-kommo) | Kommo — the international amoCRM platform, same surface on kommo.com |
| [`allmcp-yougile`](skills/allmcp-yougile) | YouGile — projects, boards, columns, tasks, chat, stickers |
| [`allmcp-salesdrive`](skills/allmcp-salesdrive) | SalesDrive — orders, products, categories, payments, reference data |
| [`allmcp-binotel`](skills/allmcp-binotel) | Binotel telephony — calls, stats, outbound dialing, employees, customers |
| [`allmcp-altegio`](skills/allmcp-altegio) | Altegio — appointments, services, clients, staff, finances |
| [`allmcp-iiko`](skills/allmcp-iiko) | iiko restaurant platform — organizations, menu, delivery orders |

Each provider skill is a lean `SKILL.md` (connect + orientation) plus
`references/*.md` — per-category playbooks the agent reads only when it works
in that category.

## Getting started with AllMCP

1. Get an API key at [allmcp.co/login](https://allmcp.co/login) — free up to
   20,000 provider tool calls a month, no card required.
2. Point your MCP client at `https://go.allmcp.co/mcp/` with header
   `X-API-Key: <your key>` ([client setup guides](https://docs.allmcp.co/documentation/quickstart)).
3. Install these skills and ask your agent to connect a provider.

## Contributing

This repository is a **build artifact**: skill content is authored and verified
against the live platform in the AllMCP backend, then synced here. Please don't
open PRs that hand-edit skill content — they'll be overwritten by the next sync.
Found a mistake or want a new provider covered? [Open an issue](https://github.com/Perception-Dynamics-Inc/allmcp-skills/issues)
and we'll fix it at the source. See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

[MIT](LICENSE) © Perception Dynamics Inc.
