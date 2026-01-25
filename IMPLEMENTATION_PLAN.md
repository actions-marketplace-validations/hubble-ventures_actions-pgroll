# Implementation Plan: GitHub Action for pgroll

This document outlines a comprehensive plan for implementing an open source GitHub Action for [pgroll](https://pgroll.com), the PostgreSQL zero-downtime migration tool by Xata.

## Executive Summary

The goal is to create a GitHub Action that enables users to run pgroll migrations as part of their CI/CD pipelines. The action will follow GitHub Actions best practices, align with pgroll's CLI patterns, and provide a modular architecture similar to other successful database migration actions like [ariga/atlas-action](https://github.com/ariga/atlas-action).

---

## 1. Understanding pgroll

### What pgroll Does

pgroll is a migration tool for PostgreSQL that enables zero-downtime schema changes by:
- Serving multiple schema versions simultaneously during migrations
- Automatically handling complex migration operations (backfills, triggers)
- Supporting instant rollbacks during active migrations
- Working with existing database schemas

### Core CLI Commands

| Command | Description | Key Use Case |
|---------|-------------|--------------|
| `pgroll init` | Initialize pgroll internal state | First-time setup |
| `pgroll start <file>` | Start a migration (creates dual schemas) | Begin migration |
| `pgroll complete` | Complete an active migration | Finalize migration |
| `pgroll rollback` | Roll back an active migration | Revert changes |
| `pgroll status` | Show current migration status | Check state |
| `pgroll validate <file>` | Validate migration file syntax | CI validation |
| `pgroll migrate <dir>` | Apply all pending migrations | Batch deployment |
| `pgroll baseline <name>` | Create baseline for existing DB | Adoption |

### Configuration Options

pgroll accepts configuration via CLI flags or environment variables:

| CLI Flag | Environment Variable | Default | Description |
|----------|---------------------|---------|-------------|
| `--postgres-url` | `PGROLL_PG_URL` | - | Database connection URL |
| `--schema` | `PGROLL_SCHEMA` | `public` | Target schema |
| `--pgroll-schema` | `PGROLL_STATE_SCHEMA` | `pgroll` | Internal state schema |
| `--lock-timeout` | `PGROLL_LOCK_TIMEOUT` | `500` | Lock timeout (ms) |
| `--role` | `PGROLL_ROLE` | - | Postgres role for DDL |

---

## 2. Architecture Decision

### Recommended Approach: Composite Actions

After analyzing similar projects (atlas-action, liquibase-action, flyway), the recommended approach is **Composite Actions** for the following reasons:

**Advantages:**
- Faster execution (no Docker build overhead)
- Simpler maintenance (YAML-based, no compiled code)
- Easy to understand and contribute to
- Native shell script support
- Better caching support for the pgroll binary

**Alternative Considered:**
- Docker Actions: Better isolation but slower and more complex
- JavaScript Actions: More powerful but requires Node.js tooling and compilation

### Modular Structure

Following the atlas-action pattern, the action should be split into:

1. **Setup Action** - Installs and configures pgroll
2. **Command Actions** - One action per pgroll command

This allows users to:
- Use only the commands they need
- Compose complex workflows
- Benefit from clear separation of concerns

---

## 3. Repository Structure

```
actions-pgroll/
├── README.md                    # Main documentation
├── LICENSE                      # Apache-2.0 (matching pgroll)
├── CONTRIBUTING.md              # Contribution guidelines
├── CODE_OF_CONDUCT.md           # Community standards
├── SECURITY.md                  # Security policy
├── .github/
│   ├── workflows/
│   │   ├── ci.yml               # CI for the action itself
│   │   └── release.yml          # Automated releases
│   ├── ISSUE_TEMPLATE/
│   │   ├── bug_report.md
│   │   └── feature_request.md
│   └── PULL_REQUEST_TEMPLATE.md
├── setup/
│   └── action.yml               # Setup/install pgroll
├── init/
│   └── action.yml               # pgroll init
├── start/
│   └── action.yml               # pgroll start
├── complete/
│   └── action.yml               # pgroll complete
├── rollback/
│   └── action.yml               # pgroll rollback
├── status/
│   └── action.yml               # pgroll status
├── validate/
│   └── action.yml               # pgroll validate
├── migrate/
│   └── action.yml               # pgroll migrate
├── baseline/
│   └── action.yml               # pgroll baseline
└── examples/
    ├── basic-migration.yml      # Simple migration workflow
    ├── ci-validation.yml        # PR validation workflow
    └── production-deploy.yml    # Production deployment
```

---

## 4. Action Specifications

### 4.1 Setup Action (`hubble-ventures/actions-pgroll/setup`)

**Purpose:** Install pgroll binary and add to PATH

**Inputs:**
| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `version` | No | `latest` | pgroll version to install |
| `token` | No | `${{ github.token }}` | GitHub token for API requests |

**Outputs:**
| Output | Description |
|--------|-------------|
| `version` | Installed pgroll version |
| `path` | Path to pgroll binary |

**Implementation Notes:**
- Download from GitHub releases (xataio/pgroll)
- Support Linux (amd64, arm64), macOS (amd64, arm64), Windows
- Cache binary using actions/cache for faster subsequent runs
- Verify checksum for security

### 4.2 Init Action (`hubble-ventures/actions-pgroll/init`)

**Purpose:** Initialize pgroll internal state in the database

**Inputs:**
| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `postgres-url` | Yes | - | Database connection URL |
| `pgroll-schema` | No | `pgroll` | Schema for internal state |

**Outputs:** None (success/failure only)

### 4.3 Start Action (`hubble-ventures/actions-pgroll/start`)

**Purpose:** Start a migration (creates dual schema versions)

**Inputs:**
| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `postgres-url` | Yes | - | Database connection URL |
| `migration-file` | Yes | - | Path to migration file |
| `schema` | No | `public` | Target schema |
| `pgroll-schema` | No | `pgroll` | Internal state schema |
| `lock-timeout` | No | `500` | Lock timeout in ms |
| `role` | No | - | Postgres role for DDL |
| `complete` | No | `false` | Also complete the migration |
| `backfill-batch-size` | No | `1000` | Rows per backfill batch |
| `backfill-batch-delay` | No | `0s` | Delay between batches |

**Outputs:**
| Output | Description |
|--------|-------------|
| `version` | New schema version name |

### 4.4 Complete Action (`hubble-ventures/actions-pgroll/complete`)

**Purpose:** Complete an active migration

**Inputs:**
| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `postgres-url` | Yes | - | Database connection URL |
| `schema` | No | `public` | Target schema |
| `pgroll-schema` | No | `pgroll` | Internal state schema |

**Outputs:**
| Output | Description |
|--------|-------------|
| `version` | Completed schema version |

### 4.5 Rollback Action (`hubble-ventures/actions-pgroll/rollback`)

**Purpose:** Roll back an active migration

**Inputs:**
| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `postgres-url` | Yes | - | Database connection URL |
| `schema` | No | `public` | Target schema |
| `pgroll-schema` | No | `pgroll` | Internal state schema |

**Outputs:**
| Output | Description |
|--------|-------------|
| `version` | Rolled back to version |

### 4.6 Status Action (`hubble-ventures/actions-pgroll/status`)

**Purpose:** Get current migration status

**Inputs:**
| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `postgres-url` | Yes | - | Database connection URL |
| `schema` | No | `public` | Target schema |
| `pgroll-schema` | No | `pgroll` | Internal state schema |

**Outputs:**
| Output | Description |
|--------|-------------|
| `schema` | Schema name |
| `version` | Current version |
| `status` | Status: "No migrations", "In progress", or "Complete" |
| `json` | Full JSON output |

### 4.7 Validate Action (`hubble-ventures/actions-pgroll/validate`)

**Purpose:** Validate migration file syntax

**Inputs:**
| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `migration-file` | Yes | - | Path to migration file |
| `postgres-url` | No | - | Database URL (for schema validation) |
| `schema` | No | `public` | Target schema |

**Outputs:**
| Output | Description |
|--------|-------------|
| `valid` | `true` or `false` |
| `errors` | Validation errors (if any) |

### 4.8 Migrate Action (`hubble-ventures/actions-pgroll/migrate`)

**Purpose:** Apply all pending migrations (start + complete)

**Inputs:**
| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `postgres-url` | Yes | - | Database connection URL |
| `migrations-dir` | Yes | - | Directory containing migrations |
| `schema` | No | `public` | Target schema |
| `pgroll-schema` | No | `pgroll` | Internal state schema |
| `lock-timeout` | No | `500` | Lock timeout in ms |
| `role` | No | - | Postgres role for DDL |
| `backfill-batch-size` | No | `1000` | Rows per backfill batch |
| `backfill-batch-delay` | No | `0s` | Delay between batches |

**Outputs:**
| Output | Description |
|--------|-------------|
| `applied-count` | Number of migrations applied |
| `version` | Final schema version |

### 4.9 Baseline Action (`hubble-ventures/actions-pgroll/baseline`)

**Purpose:** Create baseline migration for existing database

**Inputs:**
| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `postgres-url` | Yes | - | Database connection URL |
| `migration-name` | Yes | - | Name for baseline migration |
| `migrations-dir` | No | `.` | Output directory |
| `schema` | No | `public` | Target schema |
| `pgroll-schema` | No | `pgroll` | Internal state schema |

**Outputs:**
| Output | Description |
|--------|-------------|
| `file` | Path to generated baseline file |

---

## 5. Example Workflows

### 5.1 Basic CI Validation

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
      
      - uses: hubble-ventures/actions-pgroll/setup@v1
      
      - name: Validate all migration files
        run: |
          for file in migrations/*.json; do
            pgroll validate "$file"
          done
```

### 5.2 Development Database Migration

```yaml
name: Apply Migrations (Dev)
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
      
      - uses: hubble-ventures/actions-pgroll/setup@v1
      
      - uses: hubble-ventures/actions-pgroll/migrate@v1
        with:
          postgres-url: ${{ secrets.DEV_DATABASE_URL }}
          migrations-dir: ./migrations
```

### 5.3 Production Deployment with Staged Rollout

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
      
      - uses: hubble-ventures/actions-pgroll/setup@v1
      
      # Start migration (creates dual schema)
      - uses: hubble-ventures/actions-pgroll/start@v1
        id: start
        with:
          postgres-url: ${{ secrets.PROD_DATABASE_URL }}
          migration-file: ${{ inputs.migration-file }}
      
      - name: Output new schema version
        run: echo "New schema version: ${{ steps.start.outputs.version }}"
      
      # Manual approval gate would go here in real workflow
      
      # Complete migration
      - uses: hubble-ventures/actions-pgroll/complete@v1
        with:
          postgres-url: ${{ secrets.PROD_DATABASE_URL }}
```

### 5.4 Migration with Rollback on Failure

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
      
      - uses: hubble-ventures/actions-pgroll/setup@v1
      
      - uses: hubble-ventures/actions-pgroll/start@v1
        id: start
        with:
          postgres-url: ${{ secrets.DATABASE_URL }}
          migration-file: ./migrations/latest.json
      
      - name: Run application tests
        id: tests
        run: npm test
        continue-on-error: true
      
      - name: Rollback on test failure
        if: steps.tests.outcome == 'failure'
        uses: hubble-ventures/actions-pgroll/rollback@v1
        with:
          postgres-url: ${{ secrets.DATABASE_URL }}
      
      - name: Complete on test success
        if: steps.tests.outcome == 'success'
        uses: hubble-ventures/actions-pgroll/complete@v1
        with:
          postgres-url: ${{ secrets.DATABASE_URL }}
```

---

## 6. Implementation Phases

### Phase 1: Foundation (Week 1)

**Tasks:**
1. Set up repository structure
2. Create LICENSE (Apache-2.0)
3. Create initial README.md
4. Implement `setup` action
   - Binary download from GitHub releases
   - Platform detection (Linux/macOS/Windows)
   - Caching support
5. Create CI workflow for the action itself

**Deliverables:**
- Working `setup` action
- Basic documentation
- CI pipeline

### Phase 2: Core Commands (Week 2)

**Tasks:**
1. Implement `init` action
2. Implement `start` action
3. Implement `complete` action
4. Implement `rollback` action
5. Implement `status` action
6. Add integration tests using PostgreSQL service container

**Deliverables:**
- All core migration commands working
- Integration test suite

### Phase 3: Extended Commands (Week 3)

**Tasks:**
1. Implement `validate` action
2. Implement `migrate` action
3. Implement `baseline` action
4. Add comprehensive error handling
5. Add detailed logging/output

**Deliverables:**
- Complete action suite
- Error handling documentation

### Phase 4: Documentation & Polish (Week 4)

**Tasks:**
1. Write comprehensive README with examples
2. Create CONTRIBUTING.md
3. Create example workflow files
4. Add GitHub Marketplace metadata (branding)
5. Create release workflow with semantic versioning
6. Security review and SECURITY.md

**Deliverables:**
- Production-ready action
- Complete documentation
- Published to GitHub Marketplace

---

## 7. Testing Strategy

### Unit Tests
- Validate action.yml syntax
- Test input validation logic
- Test output formatting

### Integration Tests
- Use PostgreSQL service container in GitHub Actions
- Test each command against real database
- Test error scenarios (invalid migrations, connection failures)

### Example Test Workflow
```yaml
name: Integration Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: ./setup
        with:
          version: latest
      
      - uses: ./init
        with:
          postgres-url: postgres://postgres:postgres@localhost:5432/postgres
      
      # ... additional test steps
```

---

## 8. Open Source Best Practices

### Documentation
- Clear README with quick start guide
- API reference for all inputs/outputs
- Multiple example workflows
- Troubleshooting guide

### Community
- Issue templates for bugs and features
- Pull request template
- Code of conduct
- Contributing guidelines

### Releases
- Semantic versioning (v1.0.0, v1.1.0, etc.)
- Changelog for each release
- Major version tags (v1, v2) for stability
- Automated release workflow

### Security
- No secrets in logs
- Secure handling of database URLs
- Dependabot for dependency updates
- Security policy for vulnerability reporting

---

## 9. AI Coding Assistant Rules (Cursor, Claude Code, Copilot)

Modern open source projects benefit from including AI coding assistant configuration files. These help contributors using AI tools (Cursor, Claude Code, GitHub Copilot, Cline, etc.) get consistent, project-aware assistance.

### The Challenge: Multiple Formats

Different AI coding tools use different configuration formats:

| Tool | Configuration File | Format |
|------|-------------------|--------|
| **Claude Code** | `CLAUDE.md` | Markdown (auto-loaded) |
| **Cursor** | `.cursor/rules/*.mdc` | MDC with YAML frontmatter |
| **GitHub Copilot** | `.github/copilot-instructions.md` | Markdown |
| **Cline** | `.cline-rules` | Text/Markdown |
| **Devin** | `devin-guidelines.md` | Markdown |

### Recommended Approach: Single Source of Truth

To avoid duplication and keep rules in sync, use a **canonical source file** that other tool-specific files reference or are generated from.

#### Option 1: Symlink Approach (Simplest)

```
actions-pgroll/
├── .ai/
│   └── RULES.md              # Canonical source (all rules here)
├── CLAUDE.md -> .ai/RULES.md # Symlink for Claude Code
├── .cursor/
│   └── rules/
│       └── pgroll.mdc        # Generated or manually synced
└── .github/
    └── copilot-instructions.md -> ../.ai/RULES.md
```

**Pros:** Simple, no build step needed
**Cons:** Symlinks may not work on Windows, MDC format differs from Markdown

#### Option 2: Generation Script (More Flexible)

Create a script that generates tool-specific files from the canonical source:

```bash
#!/bin/bash
# scripts/sync-ai-rules.sh

# Source file
SOURCE=".ai/RULES.md"

# Generate CLAUDE.md (direct copy)
cp "$SOURCE" "CLAUDE.md"

# Generate Copilot instructions
cp "$SOURCE" ".github/copilot-instructions.md"

# Generate Cursor MDC (add frontmatter)
cat > ".cursor/rules/pgroll.mdc" << EOF
---
description: "pgroll GitHub Action development guidelines"
globs: ["**/*.yml", "**/*.yaml", "**/*.sh", "**/*.md"]
alwaysApply: true
---

$(cat "$SOURCE")
EOF

echo "AI rules synced successfully"
```

Add to CI to verify sync:
```yaml
- name: Verify AI rules are in sync
  run: |
    ./scripts/sync-ai-rules.sh
    git diff --exit-code || (echo "AI rules out of sync!" && exit 1)
```

#### Option 3: Use Airulefy (Emerging Tool)

[Airulefy](https://github.com/airulefy/Airulefy) is a new tool that automates this:

```bash
pip install airulefy
airulefy generate  # Generates all tool-specific files from .ai/
airulefy watch     # Auto-regenerate on changes
```

**Note:** Airulefy is very new (as of Jan 2025) - evaluate stability before adopting.

### Recommended Repository Structure with AI Rules

```
actions-pgroll/
├── .ai/
│   └── RULES.md              # Canonical AI assistant rules
├── CLAUDE.md                 # For Claude Code users
├── .cursor/
│   └── rules/
│       └── pgroll.mdc        # For Cursor users
├── .github/
│   ├── copilot-instructions.md  # For GitHub Copilot users
│   └── workflows/
│       └── ...
├── scripts/
│   └── sync-ai-rules.sh      # Sync script
└── ...
```

### Example Canonical Rules File (`.ai/RULES.md`)

```markdown
# pgroll GitHub Action - AI Assistant Guidelines

## Project Overview
This is a GitHub Action for pgroll, a PostgreSQL zero-downtime migration tool.
The action is implemented as composite actions (YAML-based, no compiled code).

## Architecture
- `setup/action.yml` - Installs pgroll binary
- `*/action.yml` - Individual command actions (init, start, complete, etc.)
- Each action wraps a pgroll CLI command

## Key Technologies
- GitHub Actions (composite actions)
- Shell scripting (bash)
- pgroll CLI (Go binary from xataio/pgroll)

## Coding Standards
- Use YAML for action definitions
- Follow GitHub Actions best practices
- Keep shell scripts POSIX-compatible where possible
- Use `set -e` in shell scripts for fail-fast behavior

## Common Commands
```bash
# Test actions locally with act
act -j test

# Validate action.yml syntax
yamllint */action.yml

# Run integration tests
act -j integration-test --secret-file .secrets
```

## File Patterns
- `*/action.yml` - Action definitions
- `examples/*.yml` - Example workflows
- `scripts/*.sh` - Helper scripts

## Important Notes
- Never log database URLs or credentials
- Always mask sensitive outputs
- Test against PostgreSQL 14+ (pgroll requirement)
```

### What to Include in AI Rules

Based on best practices from Claude Code and Cursor documentation:

1. **Project Context**
   - What the project does
   - Key technologies and frameworks
   - Architecture overview

2. **Coding Standards**
   - Style guidelines
   - Naming conventions
   - Error handling patterns

3. **Common Commands**
   - Build/test commands
   - Development workflow
   - Debugging tips

4. **File Organization**
   - Key directories and their purposes
   - File naming patterns
   - Where to find specific functionality

5. **Project-Specific Warnings**
   - Common pitfalls
   - Security considerations
   - Breaking change risks

### Keeping Rules Updated

1. **Add to PR template**: Remind contributors to update AI rules if they change project structure
2. **CI validation**: Check that rules files are in sync
3. **Periodic review**: Review rules quarterly or after major changes
4. **Contributor feedback**: Encourage contributors to suggest rule improvements

## 10. Considerations & Risks

### Technical Risks
1. **Binary compatibility**: pgroll releases may not cover all platforms
   - Mitigation: Document supported platforms, provide fallback to source build
   
2. **Version compatibility**: pgroll CLI may change between versions
   - Mitigation: Pin to specific versions, test against multiple versions

3. **Database connection security**: URLs contain credentials
   - Mitigation: Use GitHub secrets, mask in logs

### Adoption Risks
1. **Discoverability**: Users may not find the action
   - Mitigation: Publish to Marketplace, contribute to pgroll docs

2. **Maintenance burden**: Action needs updates with pgroll releases
   - Mitigation: Automated testing, clear contribution process

---

## 11. Success Metrics

- GitHub stars and forks
- Weekly downloads/usage
- Issue response time
- Community contributions
- Integration with pgroll documentation

---

## 12. References

- [pgroll Documentation](https://pgroll.com/docs/latest/getting-started)
- [pgroll GitHub Repository](https://github.com/xataio/pgroll)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Creating Composite Actions](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action)
- [atlas-action (Reference Implementation)](https://github.com/ariga/atlas-action)
- [Open Source Guides](https://opensource.guide/best-practices/)

---

## Next Steps

1. Review this plan and provide feedback
2. Decide on any scope adjustments
3. Begin Phase 1 implementation
4. Set up project board for tracking progress
