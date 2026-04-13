---
name: commandbox-migrations
description: >
  Use this skill when running database migrations from the CommandBox CLI using commandbox-migrations,
  including migrate up/down/reset/refresh, creating migration files, running seeders, and checking
  migration status. Covers CI/CD usage, configuration, and all CLI commands for cfmigrations.
applyTo: "**/*.{bx,cfc,cfm,bxm,json}"
---

# CommandBox Migrations Skill

## When to Use This Skill

Load this skill when:
- Running `migrate up/down/reset/refresh` from the CLI
- Creating new migration files via `migrate create`
- Running database seeders with `migrate seed`
- Checking migration status with `migrate status`
- Setting up CI/CD pipelines with database migrations
- Configuring migration connection settings in `box.json` or `.cfmigrations`

> Note: For the CFC-level Schema Builder and migration writing API, see the `cfmigrations` skill.

## Installation

```bash
box install commandbox-migrations
```

## Configuration

### Option 1 — `.cfmigrations` file (recommended)

```json
{
    "default": {
        "driver": "MSSQL",
        "host": "${DB_HOST}",
        "port": "${DB_PORT:1433}",
        "database": "${DB_DATABASE}",
        "username": "${DB_USERNAME}",
        "password": "${DB_PASSWORD}",
        "migrationsTable": "cfmigrations",
        "migrationsDirectory": "resources/database/migrations"
    }
}
```

Supported drivers: `MSSQL`, `MySQL`, `PostgreSQL`, `SQLite`, `Generic`

### Option 2 — `box.json` settings

```json
{
    "cfmigrations": {
        "connectionInfo": {
            "driver": "MySQL",
            "host": "${DB_HOST}",
            "port": "3306",
            "database": "${DB_DATABASE}",
            "username": "${DB_USERNAME}",
            "password": "${DB_PASSWORD}"
        },
        "migrationsTable": "cfmigrations",
        "migrationsDirectory": "resources/database/migrations"
    }
}
```

## CLI Commands

### Install Migration Table

```bash
# Create the cfmigrations tracking table (run once)
box migrate install
```

### Create a Migration

```bash
# Create a new up/down migration file
box migrate create create_users_table

# Creates: resources/database/migrations/YYYY_MM_DD_HHMMSS_create_users_table.cfc
```

### Run Migrations

```bash
# Run all pending migrations
box migrate up

# Run a specific number of migrations
box migrate up count=1

# Run a single named migration
box migrate up migration=create_users_table

# Rollback the last migration
box migrate down

# Rollback a specific number of steps
box migrate down count=3

# Rollback all migrations
box migrate reset

# Rollback then re-run all migrations (destructive!)
box migrate refresh

# Rollback then re-run with seeders
box migrate refresh --seed
```

### Check Status

```bash
# Show which migrations have been run
box migrate status
```

Output example:
```
+----------------------------------------------------------------------+
| Migration                                         | Ran    | Order   |
+----------------------------------------------------|--------|---------|
| 2024_01_15_120000_create_users_table              | YES    | 1       |
| 2024_01_16_090000_create_posts_table              | YES    | 2       |
| 2024_02_01_143000_add_avatar_to_users             | NO     | -       |
+----------------------------------------------------------------------+
```

### Seeders

```bash
# Run all seeders
box migrate seed

# Run a specific seeder
box migrate seed seedFile=UserSeeder
```

### Using a Named Configuration

```bash
# Use a specific connection from .cfmigrations
box migrate up --configuraton=reporting
```

## Production Patterns

### CI/CD Pipeline (GitHub Actions)

```yaml
- name: Run Migrations
  run: |
    box migrate up
  env:
    DB_HOST: localhost
    DB_DATABASE: myapp
    DB_USERNAME: ${{ secrets.DB_USER }}
    DB_PASSWORD: ${{ secrets.DB_PASS }}
```

### CommandBox Task Runner

```js
// tasks/MigrationTask.bx
class {

    function fresh( args = {} ) {
        print.yellowLine( "Refreshing database..." )
        command( "migrate refresh" ).params( seed: true ).run()
        print.greenLine( "Done!" )
    }

    function ci( args = {} ) {
        print.line( "Running CI migrations..." )
        command( "migrate install" ).run()
        command( "migrate up" ).run()
    }
}
```

### Safe Production Deployment

```bash
# Only run pending - never rollback automatically in production
box migrate up

# Verify after deployment
box migrate status
```

## Best Practices

- **Never run `migrate reset` or `migrate refresh` in production** — these are destructive. Use only `migrate up`
- **Use environment variables** for all connection credentials — never hardcode in `box.json`
- **Run `migrate install` once** during initial server setup — idempotent, safe to run multiple times
- **Run `migrate status`** before and after deployments to confirm state
- **Pin the box.json dependency version** — ensures consistent CLI behavior across environments
- **Use named configurations** (`.cfmigrations`) when multiple databases are needed (app vs. reporting)
- **Create seeders separately** from migrations — seeders are for dev data, not schema state

## Documentation

- commandbox-migrations: https://github.com/Ortus-Solutions/commandbox-migrations
- cfmigrations: https://cfmigrations.ortusbooks.com
- Migration files API: see `cfmigrations` skill
