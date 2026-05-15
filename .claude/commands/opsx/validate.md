# validate

Validate Fleet configuration, schema, and deployment state before applying changes.

## Usage

```
/opsx:validate [--config <path>] [--env <environment>] [--strict] [--format <json|text>]
```

## Description

Runs pre-flight validation checks against Fleet configuration files, database schema migrations, and environment state. Use this command before `apply` to catch issues early and ensure safe deployments.

Validation checks include:
- YAML/JSON configuration syntax and schema compliance
- Database migration ordering and integrity
- Environment variable presence and format
- Feature flag consistency
- Cross-reference checks between config sections
- Deprecated field detection

## Arguments

| Flag | Description | Default |
|------|-------------|--------|
| `--config` | Path to config file or directory | `./config` |
| `--env` | Target environment (`dev`, `staging`, `prod`) | `dev` |
| `--strict` | Treat warnings as errors | `false` |
| `--format` | Output format (`json` or `text`) | `text` |
| `--skip-db` | Skip database migration checks | `false` |
| `--skip-env` | Skip environment variable checks | `false` |

## Steps

1. **Identify scope** — Determine which config files and migrations are in scope based on `--config` path and recent git changes.

2. **Syntax validation** — Parse all YAML/JSON files and report any syntax errors with line numbers.

3. **Schema validation** — Validate config structure against Fleet's known schema. Flag unknown fields, missing required fields, and type mismatches.

4. **Migration checks** — Verify migration files:
   - Sequential version numbering (no gaps or duplicates)
   - Both `up` and `down` migrations present
   - No destructive operations without explicit `--allow-destructive` flag in prod

5. **Environment checks** — For the target `--env`, verify:
   - All referenced env vars are defined (or have defaults)
   - Secrets follow naming conventions (`FLEET_*`)
   - No dev-only values are set in prod configs

6. **Feature flag consistency** — Ensure feature flags referenced in code are declared in config and vice versa.

7. **Deprecation scan** — Warn about deprecated config keys that have known replacements.

8. **Report results** — Output a structured summary of errors, warnings, and passed checks.

## Output

In `text` format:
```
✓ Syntax valid: 12 files checked
✓ Schema valid: no unknown fields
⚠ Warning: 'smtp.legacy_tls' is deprecated, use 'smtp.tls_version'
✗ Error: Migration 0142 missing down migration
✗ Error: ENV var FLEET_REDIS_PASSWORD referenced but not set in staging

Result: FAILED (2 errors, 1 warning)
```

In `json` format:
```json
{
  "result": "failed",
  "errors": [
    {
      "type": "migration",
      "file": "db/migrations/tables/20240315142301_add_policy_results.go",
      "message": "Missing down migration function"
    },
    {
      "type": "env",
      "key": "FLEET_REDIS_PASSWORD",
      "env": "staging",
      "message": "Referenced in config but not defined"
    }
  ],
  "warnings": [
    {
      "type": "deprecation",
      "field": "smtp.legacy_tls",
      "message": "Deprecated, use smtp.tls_version instead"
    }
  ],
  "passed": 10
}
```

## Examples

```bash
# Validate all configs for staging
/opsx:validate --env staging

# Strict validation before prod deploy
/opsx:validate --env prod --strict

# Validate only a specific config file
/opsx:validate --config ./config/fleet.yml

# Skip DB checks (useful when DB is not accessible)
/opsx:validate --env dev --skip-db

# Machine-readable output for CI
/opsx:validate --env staging --format json
```

## Integration with Other Commands

- Run automatically as a pre-check in `/opsx:apply` (can be skipped with `--skip-validate`)
- Use `/opsx:diff` first to understand what changed before validating
- If validation fails, use `/opsx:status` to inspect current environment state

## Notes

- Validation is **read-only** and makes no changes to any environment
- In CI pipelines, exit code `0` = all checks passed, `1` = errors present, `2` = warnings only (with `--strict`, warnings also return `1`)
- For prod environments, destructive migration operations (DROP TABLE, DROP COLUMN) require explicit acknowledgment even if validation passes
