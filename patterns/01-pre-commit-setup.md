# 01 — Pre-commit setup

## Why pre-commit?

Running linters directly (`ruff check .`, `ty check`, etc.) works, but it creates
drift: CI and local dev end up using different flags, different versions, different
file scopes. Pre-commit fixes this — one config file governs both.

## Standard config

The standard config for Python projects using ruff + ty + complexipy:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.15.12
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: local
    hooks:
      - id: ty
        name: ty check
        description: Type check with ty.
        entry: ty check
        language: system
        pass_filenames: false
        require_serial: true
        types: [python]

      - id: complexipy
        name: complexipy (complexity check)
        entry: complexipy src/
        language: system
        pass_filenames: false
        always_run: true
```

## Variants

### ruff only (small/simple projects)

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.15.12
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
```

### ruff + ty (no complexity gate)

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.15.12
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: local
    hooks:
      - id: ty
        name: ty check
        entry: ty check
        language: system
        pass_filenames: false
        require_serial: true
        types: [python]
```

## System hooks need PATH

`ty` and `complexipy` are `language: system` hooks — they rely on whatever is
on your `PATH`. Before running pre-commit, activate your virtualenv or conda env:

```bash
conda activate myproject
pre-commit run --all-files
```

If you get `Executable 'ty' not found`, you're in the wrong environment.

## Install

```bash
pip install pre-commit
pre-commit install          # installs as git commit hook
pre-commit run --all-files  # run manually on everything
```

## Pinning ruff version

The `ruff-pre-commit` hooks download their own ruff binary (pinned to `rev`).
This means ruff version in pre-commit and ruff version in your virtualenv can
diverge. Keep them in sync by matching `rev` with what's in `pyproject.toml`:

```toml
[project.optional-dependencies]
dev = [
    "ruff>=0.15.12",
    "ty>=0.0.24",
    "complexipy>=5.2.0",
]
```
