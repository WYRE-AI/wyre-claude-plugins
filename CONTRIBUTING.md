# Contributing

## Adding a plugin

1. Create `plugins/<vendor>/<product>/` with:
   - `.claude-plugin/plugin.json` — `name` (must equal the marketplace
     entry name), `version` (sole version authority — never put `version`
     on a marketplace entry), `description`, `author`, `homepage`,
     `repository`, `license: "Apache-2.0"`.
   - `.mcp.json` — points at Conduit (`https://conduit.wyre.ai/v1/mcp`),
     **no headers, no credentials** — Conduit brokers credentials
     server-side.
   - `README.md` — what it covers, prerequisites, customer-side setup.
   - `GOVERNANCE.md` — required for anything read-only-by-design or
     risk-bearing: what it connects as, how read-only actually holds, and
     the honest enforcement gaps.
   - `skills/<domain>/SKILL.md` — see authoring rules below.
2. Register it in `.claude-plugin/marketplace.json` (`source` starts with
   `./`; no `version` field on the entry) and bump the top-level
   marketplace `version`.
3. Update `CHANGELOG.md`.
4. Run the validators (below) and `claude plugin validate .` before
   opening a PR.

## Authoring skills

The Conduit skills catalog syncs this repo and enforces rules that are
easy to violate accidentally:

- Frontmatter: `name`, `description` (block scalar; what the skill
  covers), `when_to_use` (block scalar; trigger conditions ending with a
  "Use when: keyword, keyword" list). **Nothing else** — `allowed-tools`
  disables the skill in Conduit; `triggers:` is not a real field.
- Skill slug = directory name: `[a-z0-9-]`, ≤64 chars.
- Keep `SKILL.md` under ~350 lines; overflow goes into **flat sibling
  `*.md` files in the same directory** (Conduit syncs sibling Markdown
  only — no subdirectories, no `.sql`/`.json` files).
- **No external URLs inside `skills/**`** (Conduit's scanner flags any
  host outside `*.wyre.ai` / `docs.anthropic.com` /
  `modelcontextprotocol.io`). External references belong in the plugin
  README.
- No `$ARGUMENTS`, no inline-bash, no shell commands in skill bodies.
- **Anchor every skill to real tool names** in backticks — for the SQL
  connectors that is `mssql__query`, `mssql__list_tables`,
  `mssql__describe_table` (prefix follows the connector slug). The
  tool-anchoring validator fails skills that show operations without
  naming a tool.
- Every factual claim about a vendor's schema or behavior must trace to a
  research doc under `docs/research/`. Unverifiable details are framed as
  "verify with `mssql__describe_table` first", not asserted.
- Read-only stance throughout; writes are out of scope for this
  marketplace.

## Validators

```
node scripts/check-marketplace-drift.mjs
node scripts/check-doc-references.mjs
node scripts/check-tool-anchoring.mjs
claude plugin validate .
```

CI runs all four on every PR. Any change under a plugin directory
requires bumping that plugin's `plugin.json` version (new plugins exempt).

## Commit style

Conventional commits (`feat(sage100): ...`, `fix(docs): ...`) — releases
are cut by semantic-release conventions across this org.
