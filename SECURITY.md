# Security Policy

## Supported Versions

| Version | Supported          |
| ------- | ------------------ |
| 0.x.x   | :white_check_mark: |

## Reporting a Vulnerability

We take security vulnerabilities seriously. If you discover a security issue, please report it responsibly.

### How to Report

1. **Do NOT open a public issue** for security vulnerabilities
2. Email the maintainers directly or use GitHub's private vulnerability reporting feature
3. Include as much detail as possible:
   - Description of the vulnerability
   - Steps to reproduce
   - Potential impact
   - Suggested fix (if any)

### What to Expect

- **Acknowledgment**: We will acknowledge receipt within 48 hours
- **Assessment**: We will assess the vulnerability and determine its severity
- **Fix**: We will work on a fix and coordinate disclosure timing with you
- **Credit**: We will credit you in the release notes (unless you prefer anonymity)

## Security Best Practices for Users

When using this GitHub Action:

### 1. Protect Database Credentials

```yaml
# ✅ DO: Use GitHub Secrets
- uses: hubble-ventures/actions-pgroll@v1
  with:
    command: migrate
    postgres-url: ${{ secrets.DATABASE_URL }}

# ❌ DON'T: Hardcode credentials
- uses: hubble-ventures/actions-pgroll@v1
  with:
    command: migrate
    postgres-url: postgres://user:password@host:5432/db
```

### 2. Use Least Privilege Database Roles

Create a dedicated PostgreSQL role for migrations with minimal required permissions:

```sql
-- Create migration role
CREATE ROLE pgroll_migrations WITH LOGIN PASSWORD 'secure_password';

-- Grant necessary permissions
GRANT CREATE ON DATABASE yourdb TO pgroll_migrations;
GRANT USAGE, CREATE ON SCHEMA public TO pgroll_migrations;
GRANT ALL ON ALL TABLES IN SCHEMA public TO pgroll_migrations;
GRANT ALL ON ALL SEQUENCES IN SCHEMA public TO pgroll_migrations;
```

### 3. Use Environment Protection

For production databases, use GitHub environment protection rules:

```yaml
jobs:
  migrate:
    runs-on: ubuntu-latest
    environment: production  # Requires approval
    steps:
      - uses: hubble-ventures/actions-pgroll@v1
        with:
          command: migrate
          postgres-url: ${{ secrets.PROD_DATABASE_URL }}
```

### 4. Pin Action Versions

Use specific versions or commit SHAs for better security:

```yaml
# Good: Specific version
- uses: hubble-ventures/actions-pgroll@v1.0.0

# Better: Commit SHA
- uses: hubble-ventures/actions-pgroll@abc123def456
```

## Security Features

This action includes the following security measures:

- **Credential masking**: Database URLs are passed via environment variables and not logged
- **Binary verification**: pgroll binaries are downloaded from official GitHub releases with checksum verification
- **No persistent storage**: No credentials are stored between workflow runs

## Scope

This security policy covers the `hubble-ventures/actions-pgroll` GitHub Action wrapper only.

For security issues with pgroll itself, please report to the [xataio/pgroll](https://github.com/xataio/pgroll) repository.
