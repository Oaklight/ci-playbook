# 03 — Reusable lint+test workflow

When multiple workflows need the same lint+test logic (e.g. CI on every push
*and* a release quality gate), extract it into a reusable workflow.

## Problem

Without a reusable workflow, you end up copy-pasting lint+test steps into
both `ci.yml` and `release.yml`. When you change a hook or add a Python
version, you have to update two files.

## Solution: `workflow_call`

Create `lint-test.yml` as a reusable workflow triggered by `on: workflow_call`.
All other workflows call it with `uses: ./.github/workflows/lint-test.yml`.

### `lint-test.yml` (reusable)

```yaml
name: Lint & Test

on:
  workflow_call:
    inputs:
      python-versions:
        description: "JSON array of Python versions to test"
        type: string
        default: '["3.10", "3.11", "3.12", "3.13"]'

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install pre-commit ty complexipy
      - run: pre-commit run --all-files

  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        python-version: ${{ fromJson(inputs.python-versions) }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: pip install -e ".[dev,test]"
      - run: pytest tests/ -v
```

### `ci.yml` (caller — full matrix)

```yaml
name: CI

on:
  push:
    branches: [master]
  pull_request:
    branches: [master]
  workflow_dispatch:

jobs:
  ci:
    uses: ./.github/workflows/lint-test.yml
    # uses default python-versions: ["3.10", "3.11", "3.12", "3.13"]
```

### `release.yml` (caller — single version quality gate)

```yaml
name: Release

on:
  workflow_dispatch:

jobs:
  quality:
    uses: ./.github/workflows/lint-test.yml
    with:
      python-versions: '["3.12"]'   # CI already covers the full matrix

  release:
    needs: quality
    runs-on: ubuntu-latest
    steps:
      # ... release-specific steps only
```

## When to use

Use the reusable pattern when:
- More than one workflow needs lint+test (typically CI + release)
- You want the release quality gate to be identical to CI

Skip it for simple repos with only a single CI workflow — the extra file
adds indirection without benefit.

## Limitations

- Reusable workflows must be in the same repo (for `./.github/workflows/` paths)
  or in a public repo (for `org/repo/.github/workflows/file.yml@ref`)
- `workflow_call` inputs are string-typed; use `fromJson()` for arrays
- Secrets must be explicitly forwarded: `secrets: inherit`
