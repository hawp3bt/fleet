# Rollback Command

Rollback a previously applied change or deployment in the Fleet infrastructure.

## Usage

```
/opsx rollback [target] [options]
```

## Arguments

- `target` — The deployment, migration, or configuration to roll back. Can be a deployment name, migration version, or config key.

## Options

- `--dry-run` — Preview what would be rolled back without making changes
- `--to <version>` — Roll back to a specific version or state
- `--force` — Skip confirmation prompts (use with caution)
- `--namespace <ns>` — Kubernetes namespace to target (default: `fleet`)

## Examples

```bash
# Roll back the most recent fleet-server deployment
/opsx rollback fleet-server

# Roll back to a specific deployment revision
/opsx rollback fleet-server --to 42

# Dry-run a database migration rollback
/opsx rollback migration --to 20240115120000 --dry-run

# Roll back a config change in a specific namespace
/opsx rollback fleet-config --namespace fleet-staging
```

## Behavior

1. **Identify target** — Resolves the target to a concrete resource (Kubernetes deployment, DB migration, Helm release, etc.).
2. **Fetch history** — Retrieves the revision history for the target.
3. **Confirm** — Unless `--force` is passed, displays a diff of the proposed rollback and prompts for confirmation.
4. **Execute** — Performs the rollback using the appropriate mechanism:
   - Kubernetes deployments: `kubectl rollout undo`
   - Helm releases: `helm rollback`
   - Database migrations: runs down-migrations to the target version
   - Config maps: restores from the last known-good snapshot
5. **Verify** — Waits for the rollback to stabilize and reports status.

## Safety Checks

- Will refuse to roll back a migration if active Fleet server pods are running against the current schema, unless `--force` is set.
- Warns if rolling back would affect more than one resource simultaneously.
- Logs all rollback actions to the audit trail.

## Related Commands

- [`/opsx apply`](apply.md) — Apply a change or deployment
- [`/opsx status`](status.md) — Check current deployment status
- [`/opsx diff`](diff.md) — Show differences between versions
- [`/opsx archive`](archive.md) — Archive a resource before destructive changes
