# 06 — Version maintenance

This playbook itself needs periodic updates. Tool versions drift — ruff
ships weekly, ty moves fast, complexipy and pre-commit bump occasionally.
When the templates fall behind, new projects start with stale configs.

## What to check

| Item | Where to check | Template location |
|------|----------------|-------------------|
| ruff-pre-commit `rev` | [ruff-pre-commit releases](https://github.com/astral-sh/ruff-pre-commit/tags) | `templates/*.yaml`, `patterns/01-*.md` |
| complexipy-pre-commit `rev` | [complexipy-pre-commit releases](https://github.com/rohaquinlop/complexipy-pre-commit/tags) | `templates/.pre-commit-config.yaml`, `patterns/01-*.md` |
| `ruff==` in dev deps example | [ruff PyPI](https://pypi.org/project/ruff/) | `patterns/01-*.md` |
| `ty==` in dev deps example | [ty PyPI](https://pypi.org/project/ty/) | `patterns/01-*.md` |
| `actions/checkout` | [checkout releases](https://github.com/actions/checkout/tags) | `templates/workflows/*.yml`, `patterns/02-04` |
| `actions/setup-python` | [setup-python releases](https://github.com/actions/setup-python/tags) | `templates/workflows/*.yml`, `patterns/02-04` |
| `pre-commit/action` | [pre-commit/action releases](https://github.com/pre-commit/action/tags) | `patterns/02-*.md` |
| Python version matrix | [python.org EOL schedule](https://devguide.python.org/versions/) | `templates/workflows/lint-test.yml` |

## How to check

Quick one-liner to compare template versions against latest tags:

```bash
# ruff-pre-commit
curl -s https://api.github.com/repos/astral-sh/ruff-pre-commit/tags \
  | jq -r '.[0].name'

# complexipy-pre-commit
curl -s https://api.github.com/repos/rohaquinlop/complexipy-pre-commit/tags \
  | jq -r '.[0].name'

# ruff on PyPI
curl -s https://pypi.org/pypi/ruff/json | jq -r '.info.version'

# ty on PyPI
curl -s https://pypi.org/pypi/ty/json | jq -r '.info.version'
```

Then grep what the templates currently pin:

```bash
grep -r 'rev:' templates/
grep -rn 'ruff==' patterns/
grep -rn 'ty==' patterns/
```

## When to update

- **Monthly**: quick scan of the above table — most months nothing changes
- **On ruff minor bump**: ruff ships frequently; update `rev` + `==` pin together
- **On Python EOL**: drop the oldest version from the test matrix, add the newest
- **After updating any project**: if you bumped versions in a real project,
  backport the same versions here so the template stays canonical
