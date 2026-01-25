# pgroll GitHub Action - AI Assistant Guidelines

## Project Overview

This is a GitHub Action **wrapper** for [pgroll](https://pgroll.com), a PostgreSQL zero-downtime migration tool by Xata.
The action provides a convenience layer for running pgroll in CI/CD workflows - all migration functionality comes from pgroll itself.
Implemented as a composite action (YAML-based, no compiled code).

## Architecture

```
actions-pgroll/
├── action.yml          # Single action file (setup + command routing)
```

### Design Principles

1. **Thin wrapper** - Mirrors pgroll CLI exactly, adds no extra orchestration
2. **Convenience layer** - Handles binary download, caching, and cross-platform setup
3. **Secure by default** - Checksum verification, credential masking
4. **Attribution** - All migration features are pgroll's; this action just makes it easy to use in GitHub Actions

## Key Technologies

- GitHub Actions (composite actions)
- Shell scripting (bash)
- pgroll CLI (Go binary from xataio/pgroll)
- YAML for action definitions

## Coding Standards

- Use YAML for action definitions
- Follow GitHub Actions best practices
- Keep shell scripts POSIX-compatible where possible
- Use `set -e` in shell scripts for fail-fast behavior
- Always mask sensitive values (database URLs) in logs

## Common Commands

```bash
# Test actions locally with act
act -j test-setup

# Validate action.yml syntax
python3 -c "import yaml; yaml.safe_load(open('action.yml'))"

# Run specific CI job
act -j test-integration --secret-file .secrets
```

## File Patterns

- `action.yml` - Main action definition
- `.github/workflows/*.yml` - CI workflows
- `examples/*.yml` - Example workflows for users

## pgroll CLI Reference

| Command | Description |
|---------|-------------|
| `pgroll init` | Initialize pgroll internal state |
| `pgroll start <file>` | Start a migration (creates dual schemas) |
| `pgroll complete` | Complete an active migration |
| `pgroll rollback` | Roll back an active migration |
| `pgroll status` | Show current migration status |
| `pgroll validate <file>` | Validate migration file syntax |
| `pgroll migrate <dir>` | Apply all pending migrations |
| `pgroll baseline <name>` | Create baseline for existing DB |

## Security Notes

- **Never log database URLs or credentials** - Use `env:` to pass sensitive values
- **Always use GitHub Secrets** for `postgres-url` in real workflows
- Test against PostgreSQL 14+ (pgroll requirement)
