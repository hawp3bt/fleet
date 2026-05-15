# opsx diff

Analyze and summarize differences between Fleet configurations, database migrations, or code changes.

## Usage

```
/opsx diff [target] [options]
```

## Arguments

- `target` — What to diff: `migrations`, `config`, `schema`, `api`, or a specific file path
- `--from` — Base ref, branch, tag, or version (default: `main`)
- `--to` — Target ref, branch, tag, or version (default: current working tree)
- `--format` — Output format: `summary`, `detailed`, `json` (default: `summary`)
- `--filter` — Filter by type: `breaking`, `additive`, `deprecation`

## What This Command Does

This command helps you understand what has changed between two states of the Fleet codebase, with a focus on operationally significant changes:

1. **Database migrations** — Lists new migration files, their purpose, and whether they are reversible
2. **API changes** — Identifies added, removed, or modified endpoints; flags breaking changes
3. **Configuration** — Highlights new required or optional config keys, changed defaults
4. **Schema** — Summarizes structural changes to database tables or JSON payloads

## Steps

### 1. Identify the diff scope

Determine what the user wants to compare:
- If `target` is `migrations`, look in `server/datastore/mysql/migrations/tables/` and `server/datastore/mysql/migrations/data/`
- If `target` is `config`, look in `server/config/` and any `*.yml` / `*.yaml` files
- If `target` is `schema`, look at migration files and `server/fleet/` structs
- If `target` is `api`, look at `server/service/` and `server/fleet/service.go`
- If `target` is a file path, diff that specific file

### 2. Run git diff

Use git to get the raw diff:

```bash
git diff --name-status $FROM..$TO -- [path-filter]
```

For detailed content:

```bash
git diff $FROM..$TO -- [path-filter]
```

### 3. Analyze the changes

For each changed file, determine:
- **Type of change**: added, modified, deleted, renamed
- **Risk level**: breaking, safe, unknown
- **Operational impact**: requires migration, config update, restart, etc.

#### Migration analysis
- New `*.up.sql` files → deployment requires running migrations
- Check for `ALTER TABLE ... DROP COLUMN` or `DROP TABLE` → potentially destructive
- Check for `ALTER TABLE ... ADD COLUMN ... NOT NULL` without DEFAULT → may fail on large tables
- Reversible if a corresponding `*.down.sql` exists

#### API analysis
- Removed routes or changed HTTP methods → breaking
- New required request fields → breaking for existing clients
- New optional fields or new endpoints → additive (safe)
- Changed response fields → check if omitempty or versioned

#### Config analysis
- New keys with no default → operators must update config before deploying
- Changed defaults → behavior change, document clearly
- Removed keys → breaking for operators using them

### 4. Format the output

**Summary format** (default):
```
Diff: main..feature/my-branch

Migrations (2 new):
  + 20240315120000_add_policy_automations.up.sql  [safe, reversible]
  + 20240315120001_backfill_host_display_names.up.sql  [safe, data-only]

API Changes (1 breaking, 3 additive):
  ! PATCH /api/v1/fleet/config — removed field `smtp_settings.enable_ssl_tls` (breaking)
  + GET /api/v1/fleet/software/versions — new endpoint
  + POST /api/v1/fleet/hosts/transfer — new endpoint
  + GET /api/v1/fleet/labels/{id}/hosts — new query param `order_direction`

Config Changes (1 new required):
  ! server.private_key — new required field, no default

Operational Notes:
  - Run database migrations before deploying this version
  - Update fleet.yml to include `server.private_key` before restart
  - Clients using PATCH /api/v1/fleet/config must remove `smtp_settings.enable_ssl_tls`
```

**JSON format**: Emit a structured JSON object with arrays for each category.

**Detailed format**: Include full git diff output alongside the analysis.

### 5. Highlight action items

Always end with a prioritized list of **Operational Notes** — things an operator or developer must do before or after deploying the diff.

## Examples

```
/opsx diff migrations --from v4.50.0 --to v4.51.0
/opsx diff api --from main --filter breaking
/opsx diff config --from v4.50.0
/opsx diff server/datastore/mysql/migrations/tables/
```

## Notes

- When in doubt about whether an API change is breaking, err on the side of flagging it
- Migration files with `_test` in the name are not operational migrations — skip them
- If `--from` or `--to` are not provided and git context is unavailable, ask the user to specify refs explicitly
- For very large diffs (>500 changed files), summarize by directory rather than file
