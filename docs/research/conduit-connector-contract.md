# Conduit on-prem SQL connector — tool contract for skill authors

Verified against the conduit + conduit-tunnel sources, 2026-08-17. Skills in this
repo are written against this exact surface; if the connector changes, update
this document and re-audit the skills.

## Tool surface

Every Conduit on-prem SQL connector (`mssql`, `postgres`, `mysql`) exposes the
same three MCP tools, implemented once in the tunnel binary (`sqlcommon`):

| Tool | Input | Behavior |
|---|---|---|
| `query` | `query` (required string), `max_rows` (optional int) | Runs a single read-only SELECT/WITH statement. Row cap default **100**, hard max **1000**. 15-second query timeout. |
| `list_tables` | `schema` (optional string) | Lists base tables (`TABLE_SCHEMA`, `TABLE_NAME`), optionally filtered by schema. |
| `describe_table` | `table` (required), `schema` (optional) | Returns `COLUMN_NAME`, `DATA_TYPE`, `IS_NULLABLE`, `CHARACTER_MAXIMUM_LENGTH` ordered by ordinal position. |

The gateway prefixes tool names with the connector slug: **`mssql__query`**,
`mssql__list_tables`, `mssql__describe_table` (likewise `postgres__*`,
`mysql__*`). A named connector instance uses its own slug: config
`{"sage100": {"type": "mssql", ...}}` surfaces **`sage100__query`** etc.
Skills should name the `mssql__*` forms and note that the prefix follows the
connector slug the org configured.

## Guardrails (what a skill can rely on)

1. **Read-only statement filter**: comments are stripped, at most one trailing
   `;` is tolerated, any other `;` is rejected ("only a single statement"), and
   the statement must start with `SELECT` or `WITH`. No EXEC, no sp_*, no
   multi-statement batches.
2. **Row caps**: `max_rows <= 0` → 100; `> 1000` → clamped to 1000. Results
   beyond the cap are not read; the result carries `"truncated": true`.
3. **Result shape**: one text content block containing JSON
   `{"rows": [...], "rowCount": N, "truncated": bool}`. Byte columns are
   coerced to strings.
4. **`describe_table` identifier rule**: `table`/`schema` must match
   `^[A-Za-z_][A-Za-z0-9_]*$`. `AR_Customer` passes; anything with a dot,
   space, or bracket does not — route those through `query` against
   `INFORMATION_SCHEMA.COLUMNS` instead. Linked-server four-part names can
   never go through `describe_table`.
5. **The real security boundary is the SQL principal**: a scoped read-only
   login (`db_datareader` or narrower). The connector cannot verify grants.
6. Connection is lazy — a down database surfaces on the first tool call, not at
   config time. Transport timeout to the tunnel is 30s (504 `tunnel_timeout`).

## Connector configuration (what setup instructions can reference)

Config is a slug → JSON map pushed to the tunnel from Conduit
(`PUT /api/orgs/:orgId/tunnel/config`):

```jsonc
{"connectors": {
  "mssql": {"host": "10.0.0.5", "port": 1433, "database": "MAS_ABC",
             "user": "conduit_readonly", "password": "...",
             "encrypt": "true", "auth": "sql"}
}}
```

- `mssql` fields: `host` (required), `port` (default 1433), `database`
  (required), `user`, `password`, `encrypt` (`"true"` default | `"disable"`),
  `auth` (`"sql"` default | `"integrated"` — Windows-only, gMSA service
  identity, takes no user/password).
- Named instances: `{"sage100": {"type": "mssql", ...}}` — one instance per
  Sage company database is the natural multi-company pattern
  (`sage100_abc` → `MAS_ABC`).
- OPENQUERY through a linked server counts as a SELECT and passes the filter,
  as long as the inner string contains no semicolons.

## Constraints on skill authoring (Conduit catalog sync + scanner)

The Conduit skills catalog syncs marketplace repos and enforces:

- Frontmatter must include `description` (1–1024 chars); slug = the skill's
  directory name, `[a-z0-9-]`, ≤64 chars. SKILL.md ≤ 256 KB.
- **`allowed-tools` must not appear** — it is stripped and RED-grades the skill
  (stored disabled). Same for `$ARGUMENTS`, `` !`cmd` `` inline-bash, and
  shell-execution patterns in the body.
- External URLs outside `docs.anthropic.com` / `modelcontextprotocol.io` /
  `*.wyre.ai` grade the skill YELLOW. Keep skills green: no external links in
  `skills/**`; put external references in the plugin README (not synced).
- Sibling `*.md` files in the skill directory are synced with the skill; other
  extensions (`.sql`, `.json`, `.csv`) are **not** — reference material must be
  Markdown, placed flat in the skill directory.
- Tool-name mentions in `lower_snake_case` backticks (e.g. `mssql__query`) are
  how skills anchor to the connector's real tool surface; the repo's
  `check-tool-anchoring` validator greps for exactly that.

## Known platform limitations (tracked upstream, not skill content)

1. With access-grant enforcement enabled, Conduit currently requires an
   admin-tier grant on the connector slug before members can call the SQL
   tools, despite the tools being read-only. Setup instructions should say
   so until the tools are classified read-tier upstream.
2. The Conduit skills catalog currently syncs a single marketplace repo;
   serving this marketplace through the catalog depends on multi-repo sync
   landing upstream.
3. Skill packs from this marketplace are not yet gated on connector access —
   a synced pack is visible to all org members whether or not they can call
   the connector.
