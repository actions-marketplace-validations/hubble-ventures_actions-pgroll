# Contributing to actions-pgroll

Thank you for your interest in contributing to the pgroll GitHub Action! This document provides guidelines and instructions for contributing.

## Getting Started

1. Fork the repository
2. Clone your fork locally
3. Create a new branch for your feature or fix

## Development Setup

### Prerequisites

- [act](https://github.com/nektos/act) - For local GitHub Actions testing
- Python 3.x - For YAML validation
- Docker - Required by act for running actions locally

### Testing Locally

```bash
# Validate YAML syntax
python3 -c "import yaml; yaml.safe_load(open('action.yml'))"

# Test the setup action
act -j test-setup

# Test with PostgreSQL (requires Docker)
act -j test-integration

# Run specific job with secrets
act -j test-integration --secret-file .secrets
```

Create a `.secrets` file for local testing (never commit this):

```
DATABASE_URL=postgres://user:pass@localhost:5432/db
```

## Project Structure

```
actions-pgroll/
├── action.yml          # Single action file (setup + commands)
├── .github/
│   └── workflows/
│       └── ci.yml      # CI workflow
└── examples/           # Example workflows
```

## Making Changes

### Adding a New Input

1. Add the input to `action.yml`
2. Update the README.md with the new input
3. Add tests in `.github/workflows/ci.yml`

### Modifying the Action

1. Keep changes minimal and focused
2. Ensure backwards compatibility
3. Test all affected commands

### Code Style

- YAML: 2-space indentation
- Shell: Use `set -e` for fail-fast
- Always mask sensitive values in logs
- Keep shell scripts POSIX-compatible where possible

## Testing

All changes must pass CI before merging:

- **Lint**: YAML syntax validation
- **Test Setup**: Verify binary installation on Linux and macOS
- **Test Validate**: Verify migration validation works
- **Integration Tests**: Full workflow against PostgreSQL

## Pull Request Process

1. Ensure all tests pass
2. Update documentation if needed
3. Add a clear description of changes
4. Reference any related issues

### Commit Messages

Follow conventional commits:

```
feat: add new input for backfill delay
fix: handle missing version in status output
docs: update README with new examples
ci: add macOS testing matrix
```

## Releasing

Releases are managed by maintainers:

1. Update version in relevant places
2. Create a new GitHub release with semantic version tag
3. Update major version tag (e.g., `v1` points to latest `v1.x.x`)

## Security

- Never log database URLs or credentials
- Report security issues privately to maintainers
- See [SECURITY.md](SECURITY.md) for our security policy

## Questions?

Open an issue with the `question` label or start a discussion.

## License

By contributing, you agree that your contributions will be licensed under the Apache-2.0 License.
