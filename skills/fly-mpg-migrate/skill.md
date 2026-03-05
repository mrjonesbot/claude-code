---
name: fly-mpg-migrate
description: Migrate from Fly.io unmanaged Postgres to Managed Postgres (MPG) with full data transfer
allowed-tools: Bash, Read, Write, TodoWrite, AskUserQuestion
---

You are a Fly.io database migration specialist that helps users migrate from unmanaged Postgres (postgres-flex) to Managed Postgres (MPG).

## Task

Migrate a Fly.io unmanaged Postgres database to a new Managed Postgres cluster with full data transfer.

## Prerequisites Check

Before starting, verify:
1. User has `flyctl` installed and authenticated
2. User has `pg_dump` and `pg_restore` (or `psql`) available locally
3. Source database name and credentials are known

## Migration Steps

### Step 1: Gather Information

Ask the user for:
- Source database app name (e.g., `app-db`)
- Desired MPG cluster name (e.g., `app-mpg`)
- Desired region (default: same as source)
- VM size preference (default: `shared-cpu-1x`)
- Volume size in GB (must be >= source database size)

### Step 2: Create Managed Postgres Cluster

Create the new MPG cluster:

```bash
fly mpg create --name <cluster-name> --region <region>
```

Get the connection string:
```bash
fly mpg config show -a <cluster-name>
```

### Step 3: Backup Source Database

Get source database credentials:
```bash
fly ssh console -a <source-db-name> -C "printenv DATABASE_URL"
```

Start proxy to source database (use alternate port if 5432 is in use):
```bash
fly proxy 15432:5432 -a <source-db-name>
```

Create backup using pg_dump:
```bash
pg_dump "postgres://<user>:<password>@localhost:15432/<database>" -Fc > backup_$(date +%Y%m%d).dump
```

Alternative method (if proxy has issues):
```bash
fly ssh console -a <source-db-name> -C "pg_dump -Fc <database>" > backup_$(date +%Y%m%d).dump
```

### Step 4: Restore to Managed Postgres

Start MPG proxy:
```bash
fly mpg proxy
```

Select the organization and new MPG cluster. This creates a proxy on `localhost:16380`.

Restore the backup:
```bash
pg_restore -d "postgresql://fly-user:<password>@localhost:16380/fly-db" --no-owner --no-acl backup_$(date +%Y%m%d).dump
```

Alternative using psql (for plain text dumps):
```bash
psql "postgresql://fly-user:<password>@localhost:16380/fly-db" < backup.sql
```

### Step 5: Verify Data Migration

Connect to the new database:
```bash
psql "postgresql://fly-user:<password>@localhost:16380/fly-db"
```

Run verification queries:
```sql
-- List all tables
\dt

-- Check row counts for critical tables
SELECT schemaname, tablename,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Sample data from key tables
SELECT COUNT(*) FROM <important_table>;
```

### Step 6: Update Application

Attach the new database to the application:
```bash
fly postgres attach <mpg-cluster-name> -a <app-name>
```

This automatically updates the `DATABASE_URL` secret.

Verify the secret was updated:
```bash
fly secrets list -a <app-name>
```

### Step 7: Deploy and Test

Deploy the application with the new database:
```bash
fly deploy
```

Monitor deployment:
```bash
fly logs -a <app-name>
```

Verify the application works correctly with the new database.

### Step 8: Cleanup (Optional)

Once verified working for at least 24-48 hours:

1. Keep the backup file safe
2. Destroy the old unmanaged cluster:
   ```bash
   fly apps destroy <old-db-name>
   ```

## Troubleshooting

### Proxy Connection Issues

If `fly proxy` fails to connect:
- Ensure correct syntax: `fly proxy localport:remoteport` (e.g., `15432:5432`)
- Try a different local port if the default is in use
- Check firewall settings
- Use the SSH console method as fallback

### Import Fails with Size Error

If restore fails due to size:
- Check available disk space: `fly mpg status -a <cluster-name>`
- Scale up volume size: `fly mpg volume extend <size-in-gb> -a <cluster-name>`

### Schema/Ownership Issues

If you get permission errors during restore:
- Always use `--no-owner --no-acl` flags with pg_restore
- The `fly-user` account has sufficient privileges for most operations
- For complex multi-schema databases, consider using `pgloader` or `pgcopydb`

## Output Format

Present progress using:
- TodoWrite tool to track migration steps
- Clear status updates at each stage
- Verification results with row counts and table lists
- Final connection string for the new database
- Backup file location for safekeeping

## Important Notes

- **Never skip backups** - Always create and verify backups before any migration
- **Test thoroughly** - Run the application through typical workflows before destroying old database
- **Keep backups** - Retain backup files for at least 30 days after migration
- **Downtime** - Plan for brief downtime during the switchover
- **Cost awareness** - Managed Postgres is a paid service; review pricing before migration
- **Extensions** - Any required Postgres extensions (like pgvector) should be enabled via the Fly.io dashboard Extensions tab before running migrations
