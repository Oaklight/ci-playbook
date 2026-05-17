# ci-playbook

Documented CI/CD patterns and templates I use across my Python projects.

Everything here is battle-tested across 10+ repos and follows a single
philosophy: **pre-commit is the single source of truth for code quality** —
the same hooks run locally and in CI.

## Patterns

| Pattern | Description |
|---------|-------------|
| [01 — Pre-commit setup](patterns/01-pre-commit-setup.md) | Standard `.pre-commit-config.yaml` for Python projects |
| [02 — CI lint via pre-commit](patterns/02-ci-lint-with-pre-commit.md) | Run pre-commit in GitHub Actions |
| [03 — Reusable lint+test workflow](patterns/03-reusable-lint-test-workflow.md) | `workflow_call` to share lint+test across CI and release |
| [04 — Release workflow (CalVer)](patterns/04-release-workflow.md) | Automated tagging, version bumping, GitHub Release creation |
| [05 — Dynamic test-dep installation](patterns/05-dynamic-deps.md) | Auto-detect optional extras from `pyproject.toml` |

## Templates

Ready-to-copy workflow files live in [`templates/`](templates/):

```
templates/
├── .pre-commit-config.yaml          # ruff + ty + complexipy
├── .pre-commit-config-ruff-only.yaml
└── workflows/
    ├── ci-simple.yml                # Single-repo CI (lint + test)
    ├── lint-test.yml                # Reusable workflow (workflow_call)
    ├── ci-with-reusable.yml         # CI that calls lint-test.yml
    └── release.yml                  # CalVer release with quality gate
```

## Tool versions

| Tool | Role | Notes |
|------|------|-------|
| [ruff](https://github.com/astral-sh/ruff) | Lint + format | Replaces flake8, isort, black |
| [ty](https://github.com/astral-sh/ty) | Type check | Astral's fast type checker |
| [complexipy](https://github.com/rohaquinlop/complexipy) | Complexity check | Cognitive + cyclomatic |
| [pre-commit](https://pre-commit.com) | Hook runner | Orchestrates all of the above |

## Philosophy

- `pre-commit run --all-files` is the canonical lint gate — not raw tool invocations
- System hooks (`ty`, `complexipy`) need the project virtualenv on `PATH`; install dev deps before running
- CI uses `pre-commit/action@v3.0.1` for repos that already install dev deps, or `pip install pre-commit ty complexipy && pre-commit run --all-files` for simpler setups
- Test jobs are always separate from lint — fail fast on lint before spending time on tests
