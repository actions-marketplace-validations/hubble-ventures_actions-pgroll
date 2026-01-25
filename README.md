# GitHub Action for pgroll

[![CI](https://github.com/hubble-ventures/actions-pgroll/actions/workflows/ci.yml/badge.svg)](https://github.com/hubble-ventures/actions-pgroll/actions/workflows/ci.yml)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)

A GitHub Action wrapper for running [pgroll](https://pgroll.com) in your CI/CD workflows. pgroll is the PostgreSQL zero-downtime migration tool by [Xata](https://xata.io).

## What This Action Does

This action is a thin convenience layer that:

- **Installs pgroll** - Downloads and caches the official pgroll binary from [xataio/pgroll](https://github.com/xataio/pgroll) releases
- **Verifies integrity** - Validates binary checksums for security
- **Wraps CLI commands** - Provides GitHub Actions-friendly inputs and outputs for all pgroll commands
- **Handles cross-platform setup** - Works on Linux, macOS, and Windows runners

All migration functionality is provided by pgroll itself. For details on how pgroll works (zero-downtime migrations, dual schema versions, instant rollback, etc.), see the [pgroll documentation](https://pgroll.com/docs/latest/getting-started).

## Quick Start

```yaml
- uses: hubble-ventures/actions-pgroll@v1
  with:
    command: migrate
    postgres-url: ${{ secrets.DATABASE_URL }}
    migrations-dir: ./migrations
```

## Available Commands

| Command | Description |
|---------|-------------|
| `init` | Initialize pgroll internal state in the database |
| `start` | Start a migration (creates dual schema versions) |
| `complete` | Complete an active migration |
| `rollback` | Roll back an active migration |
| `status` | Get current migration status |
| `validate` | Validate migration file syntax |
| `migrate` | Apply all pending migrations |
| `baseline` | Create baseline for existing database |

## Inputs

### Common Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `command` | Yes | - | pgroll command to run |
| `version` | No | `latest` | pgroll version to install |
| `verify-checksum` | No | `true` | Verify binary checksum |
| `postgres-url` | Varies | - | PostgreSQL connection URL |

### Command-Specific Inputs

| Input | Used By | Description |
|-------|---------|-------------|
| `migration-file` | start, validate | Path to migration file |
| `migrations-dir` | migrate, baseline | Path to migrations directory |
| `migration-name` | baseline | Name for baseline migration |
| `schema` | All except validate | Target schema (default: `public`) |
| `pgroll-schema` | All except validate | Internal state schema (default: `pgroll`) |
| `lock-timeout` | start, migrate | Lock timeout in ms (default: `500`) |
| `role` | start, migrate | PostgreSQL role for DDL |
| `complete` | start | Complete migration after start |
| `backfill-batch-size` | start, migrate | Rows per backfill batch (default: `1000`) |
| `backfill-batch-delay` | start, migrate | Delay between batches (default: `0s`) |

## Outputs

| Output | Command | Description |
|--------|---------|-------------|
| `version` | start, complete, rollback, migrate | Schema version |
| `status` | status | Migration status |
| `json` | status | Full JSON output |
| `valid` | validate | Validation result (`true`/`false`) |
| `applied-count` | migrate | Number of migrations applied |
| `file` | baseline | Path to generated baseline file |
| `pgroll-version` | All | Installed pgroll version |
| `pgroll-path` | All | Path to pgroll binary |

## Examples

### Validate Migrations in PR

```yaml
name: Validate Migrations
on:
  pull_request:
    paths:
      - 'migrations/**'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: hubble-ventures/actions-pgroll@v1
        with:
          command: validate
          migration-file: migrations/latest.json
```

### Apply Migrations on Push

```yaml
name: Apply Migrations
on:
  push:
    branches: [main]
    paths:
      - 'migrations/**'

jobs:
  migrate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: hubble-ventures/actions-pgroll@v1
        with:
          command: migrate
          postgres-url: ${{ secrets.DATABASE_URL }}
          migrations-dir: ./migrations
```

### Production Deployment with Staged Rollout

```yaml
name: Production Migration
on:
  workflow_dispatch:
    inputs:
      migration-file:
        description: 'Migration file to apply'
        required: true

jobs:
  migrate:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      
      # Start migration (creates dual schema)
      - uses: hubble-ventures/actions-pgroll@v1
        id: start
        with:
          command: start
          postgres-url: ${{ secrets.PROD_DATABASE_URL }}
          migration-file: ${{ inputs.migration-file }}
      
      - name: Output new schema version
        run: echo "New schema version: ${{ steps.start.outputs.version }}"
      
      # Complete migration after verification
      - uses: hubble-ventures/actions-pgroll@v1
        with:
          command: complete
          postgres-url: ${{ secrets.PROD_DATABASE_URL }}
```

### Migration with Rollback on Failure

```yaml
name: Safe Migration
on:
  push:
    branches: [main]

jobs:
  migrate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: hubble-ventures/actions-pgroll@v1
        id: start
        with:
          command: start
          postgres-url: ${{ secrets.DATABASE_URL }}
          migration-file: ./migrations/latest.json
      
      - name: Run application tests
        id: tests
        run: npm test
        continue-on-error: true
      
      - name: Rollback on test failure
        if: steps.tests.outcome == 'failure'
        uses: hubble-ventures/actions-pgroll@v1
        with:
          command: rollback
          postgres-url: ${{ secrets.DATABASE_URL }}
      
      - name: Complete on test success
        if: steps.tests.outcome == 'success'
        uses: hubble-ventures/actions-pgroll@v1
        with:
          command: complete
          postgres-url: ${{ secrets.DATABASE_URL }}
      
      - name: Fail if tests failed
        if: steps.tests.outcome == 'failure'
        run: exit 1
```

## Security

- **Checksum verification** - Binary integrity verified via SHA256 (enabled by default)
- **Never log database URLs** - Connection strings are masked in logs
- **Use GitHub Secrets** - Always store `postgres-url` in repository secrets
- **Least privilege** - Use a dedicated database role with minimal permissions

## Requirements

- PostgreSQL 14 or later (pgroll requirement)
- GitHub-hosted runners: Linux, macOS, or Windows

## License

Apache-2.0 - See [LICENSE](LICENSE)

## Contributing

Contributions are welcome! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## Related

- [pgroll](https://github.com/xataio/pgroll) - The pgroll CLI tool
- [pgroll Documentation](https://pgroll.com/docs/latest/getting-started) - Official documentation
