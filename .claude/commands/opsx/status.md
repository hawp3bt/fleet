# Command: opsx status

Check the operational status of Fleet infrastructure components, services, and deployments.

## Usage

```
/opsx status [component] [--env <environment>] [--verbose]
```

## Arguments

- `component` (optional): Specific component to check. One of: `fleet`, `db`, `redis`, `s3`, `all` (default: `all`)
- `--env`: Target environment. One of: `dev`, `staging`, `prod` (default: `dev`)
- `--verbose`: Show detailed output including logs and metrics

## What This Command Does

This command performs a comprehensive health check across Fleet infrastructure components and reports their current operational status.

### Steps Performed

1. **Gather environment context**
   - Identify the target environment from `--env` flag or current kubectl context
   - Load relevant configuration from `.claude/config/` or environment variables

2. **Check Fleet server**
   - Query `/healthz` endpoint on the Fleet server
   - Verify expected pod count matches running pod count
   - Check recent error rates from logs (last 15 minutes)
   - Report Fleet version currently deployed

3. **Check database (MySQL)**
   - Verify database pod/service is reachable
   - Check replication lag if applicable
   - Report connection pool utilization
   - Flag any recent migration failures

4. **Check Redis**
   - Ping Redis service
   - Report memory utilization vs. configured max
   - Check for any keyspace anomalies

5. **Check S3 / object storage**
   - Verify bucket accessibility using configured credentials
   - Check for any recent write/read failures in Fleet logs

6. **Summarize and report**
   - Print a status table with component, status (✅ OK / ⚠️ WARN / ❌ ERROR), and brief notes
   - Exit with non-zero code if any component is in ERROR state

## Example Output

```
Fleet Infrastructure Status — env: staging
──────────────────────────────────────────
Component   Status   Notes
──────────  ───────  ────────────────────────────────────
fleet       ✅ OK    v4.52.1 · 3/3 pods running
db          ⚠️ WARN  replication lag: 8s (threshold: 5s)
redis       ✅ OK    memory 42% · 0 evictions
s3          ✅ OK    bucket reachable · last write: 2m ago
──────────────────────────────────────────
Overall: WARN — 1 component needs attention
```

## Implementation Notes

- Use `kubectl` for Kubernetes-based deployments; fall back to `docker compose ps` for local dev
- Credentials and endpoints should be sourced from environment variables or Vault — never hardcoded
- When `--verbose` is set, tail the last 50 log lines per component and include them in output
- This command is read-only; it makes no changes to infrastructure
- Suitable for use in CI pipelines as a pre-deploy gate (exits non-zero on ERROR)

## Related Commands

- `/opsx apply` — Apply infrastructure or config changes
- `/opsx diff` — Preview changes before applying
- `/opsx explore` — Investigate a specific component in depth
- `/opsx archive` — Archive old deployment artifacts
