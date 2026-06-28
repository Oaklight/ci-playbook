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
    rev: v0.15.20
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: local
    hooks:
      - id: ty
        name: ty check
        description: Type check src/ with ty.
        entry: ty check src/
        language: system
        pass_filenames: false
        require_serial: true
        types: [python]

  - repo: https://github.com/rohaquinlop/complexipy-pre-commit
    rev: v5.1.0
    hooks:
      - id: complexipy
        files: ^src/
        exclude: ^src/_vendor/
        args: [-mx, "20"]
```

Replace `files`/`exclude` patterns to match your package layout (e.g.
`files: ^src/mypackage/`, `exclude: ^src/mypackage/_vendor/`).

## Variants

### ruff only (small/simple projects)

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.15.20
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
```

### ruff + ty (without complexipy)

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.15.20
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
```

## System hooks need PATH

`ty` is a `language: system` hook — it relies on whatever is on your `PATH`.
Before running pre-commit, activate your virtualenv or conda env:

```bash
conda activate myproject
pre-commit run --all-files
```

If you get `Executable 'ty' not found`, you're in the wrong environment.

Note: `ruff` and `complexipy` use their official pre-commit repos, which
download their own binaries — no PATH dependency for those.

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
    "ruff==0.15.20",
    "ty==0.0.54",
]
```

Pin exact versions (`==`) for dev tools to ensure every contributor and CI
run the same linter/type-checker. Bump these together with `rev` in
`.pre-commit-config.yaml` when upgrading.

## complexipy: complexity as a signal, not a gate

We use the [official complexipy pre-commit hook](https://github.com/rohaquinlop/complexipy-pre-commit)
instead of a local system hook. This avoids PATH/environment issues and lets
pre-commit manage the complexipy binary automatically.

**Philosophy**: high complexity is a code smell, not a bug. complexipy should
be enabled by default so you see the signal, but treated as advisory — it
should inform, not block.

### Making complexipy advisory

**Locally**: complexipy runs as part of `pre-commit run` and will block
commits by default. To make it advisory, use `SKIP`:

```bash
# Run all hooks except complexipy
SKIP=complexipy git commit -m "..."

# Or run complexipy separately to see the report
pre-commit run complexipy --all-files
```

If a function legitimately needs to be complex (parser, state machine, CLI
dispatcher), skip it consciously — don't disable the check entirely.

**In CI**: split into a blocking lint step and an advisory complexity step:

```yaml
- name: Lint (ruff + ty)
  run: SKIP=complexipy pre-commit run --all-files

- name: Complexity check (advisory)
  run: pre-commit run complexipy --all-files
  continue-on-error: true
```

### Choosing a threshold (`-mx`)

The `-mx` flag sets the maximum cognitive complexity per function. Pick a
threshold that matches where the project is, not where you wish it were:

| Scenario | `-mx` | Rationale |
|----------|-------|-----------|
| New greenfield project | `15` | Enforce clean design from day 1. If a function hits 15, it's time to extract helpers. |
| Active project, good hygiene | `20` | The sweet spot — catches genuinely tangled functions while allowing reasonable branching. |
| Legacy project being onboarded | `30–45` | Get visibility into the worst offenders without drowning in noise. Ratchet down over time. |
| Fast-moving prototype / research | `45` or disable | Don't slow down iteration. Re-evaluate when the project stabilizes. |

**Ratcheting strategy for legacy projects**: start with a high threshold that
passes today, then lower it by 5 each quarter. New code is written to the
lower bar; old code gets refactored when touched.

### Config example

```yaml
  - repo: https://github.com/rohaquinlop/complexipy-pre-commit
    rev: v5.1.0
    hooks:
      - id: complexipy
        files: ^src/mypackage/
        exclude: ^src/mypackage/_vendor/
        args: [-mx, "20"]
```

Replace `files`/`exclude` patterns to match your package layout.

To disable complexipy entirely, comment out the block — don't raise `-mx`
to an absurdly high value, which hides the signal without removing the cost.
