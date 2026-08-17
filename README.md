# wyre-claude-plugins

Curated, publicly available Claude Code plugins for **line-of-business
systems** — the counterpart to
[msp-claude-plugins](https://github.com/wyre-technology/msp-claude-plugins)
for systems that live behind the customer's firewall instead of behind a
cloud API.

## The model: generic connectors + curated skills

Vendor-specific MCP servers are the wrong shape for on-prem databases. A
Sage 100, an IQMS, or a custom ERP is ultimately a SQL database with a
schema — there is no proprietary protocol to wrap, so a dedicated server
adds maintenance surface without adding capability. What *is*
vendor-specific is knowledge: which tables answer which business
questions, which composite keys make joins correct, which dialect quirks
break naive queries.

So this marketplace pairs:

- **Generic connectors** (Conduit on-prem tunnel: `mssql`, `postgres`,
  `mysql`; ODBC via linked-server bridges) that expose a minimal read-only
  tool surface — `query`, `list_tables`, `describe_table`, and
- **Curated skills** that teach Claude the vendor's schema, canonical
  queries, and setup path on top of those tools.

## Plugins

| Plugin | System | Status |
|---|---|---|
| [`sage100`](./plugins/sage/sage100) | Sage 100 ERP (Standard / Advanced / Premium) | 0.1.0 — six skills, read-only |

Planned next (matching Conduit's connector roadmap): IQMS / DELMIAworks
(Oracle), generic Postgres/MSSQL starter packs, and further ERP verticals.

## Installation

Plugins are consumed two ways:

1. **Conduit skills catalog** — Conduit syncs this marketplace and serves
   the skills over MCP to connected clients (skills show up alongside the
   org's connector tools).
2. **Claude Code marketplace** — `claude plugin marketplace add
   wyre-ai/wyre-claude-plugins`, then install individual plugins.

## Repository layout

```
.claude-plugin/marketplace.json   # canonical registry
plugins/<vendor>/<product>/       # one directory per plugin
  .claude-plugin/plugin.json      # version authority
  .mcp.json                       # Conduit MCP endpoint (no credentials)
  skills/<domain>/SKILL.md        # + flat sibling *.md references
  README.md / GOVERNANCE.md
docs/research/                    # verified research each plugin is built on
scripts/                          # CI validators
_templates/                       # authoring templates
```

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md). The short version: every factual
claim in a skill traces to a research doc under `docs/research/`, every
skill anchors to the connector's real tool names, and the CI validators
must pass.

## License

Apache-2.0
