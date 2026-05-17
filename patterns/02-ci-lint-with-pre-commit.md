# 02 — CI lint via pre-commit

Two approaches depending on whether your CI job installs dev deps anyway.

## Approach A: `pre-commit/action` (recommended when you install dev deps)

Use the official `pre-commit/action` when the CI job already installs
project deps (which puts `ty` and `complexipy` on PATH as system hooks need).

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies         # needed for ty + complexipy system hooks
        run: pip install -e ".[dev]"

      - name: Run pre-commit checks
        uses: pre-commit/action@v3.0.1
```

`pre-commit/action` caches the pre-commit environments automatically (keyed by
`.pre-commit-config.yaml` hash), so ruff binary downloads are cached across runs.

## Approach B: manual install (no dev deps needed)

When the CI job doesn't need to install the full project (e.g. lint-only,
no test step in the same job), install pre-commit + system tools directly:

```yaml
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
```

This is used in the [zerodep reusable workflow](03-reusable-lint-test-workflow.md)
where the lint job is fully self-contained.

## Always separate lint from test

Keep lint and test as separate jobs. Benefits:
- Lint failures surface immediately without waiting for tests to run
- Test matrix (multiple Python versions) doesn't multiply lint cost
- `needs: [lint]` on the test job enforces order

```yaml
jobs:
  lint:
    ...

  test:
    needs: lint          # optional but makes ordering explicit
    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12", "3.13"]
    ...
```

## What about `ruff check --fix` in CI?

The ruff hook in `.pre-commit-config.yaml` uses `args: [--fix]`, which modifies
files in place. In CI there's no commit to update, so if ruff finds something to
fix, the hook exits non-zero (changes detected) and CI fails — which is the
correct behavior. You don't need to change the flag for CI.
