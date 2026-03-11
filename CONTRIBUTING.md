# Contributing to Vorion Systems Projects

Thank you for your interest in contributing! This guide applies to all repositories in the [vorionsys](https://github.com/vorionsys) organization.

## Getting Started

1. **Fork** the repository you want to contribute to
2. **Clone** your fork locally
3. **Create a branch** from `main` (`git checkout -b feat/your-feature`)
4. **Make your changes** with tests
5. **Commit** using [Conventional Commits](https://www.conventionalcommits.org/) (`feat:`, `fix:`, `docs:`, etc.)
6. **Push** and open a Pull Request

## Development

### TypeScript (vorion monorepo)

```bash
git clone https://github.com/vorionsys/vorion.git
cd vorion
npm install
npx turbo build
npx turbo test
```

### Python (cognigate)

```bash
git clone https://github.com/vorionsys/cognigate.git
cd cognigate
pip install -e .
pytest tests/
```

## Pull Request Guidelines

- All PRs must pass CI (lint, typecheck, build, test)
- Include tests for new functionality
- Update documentation for user-facing changes
- One logical change per PR
- Reference related issues with `Closes #N` or `Fixes #N`

## Reporting Bugs

Open an issue in the relevant repository with:
- Steps to reproduce
- Expected vs actual behavior
- Environment details (OS, Node/Python version)

## Security Vulnerabilities

**Do NOT open public issues for security vulnerabilities.** Email [security@vorion.org](mailto:security@vorion.org) instead.

## Code of Conduct

All contributors must follow our [Code of Conduct](CODE_OF_CONDUCT.md).

## Questions?

Open a discussion or email hello@vorion.org.
