# 05 — Dynamic test-dep installation

For libraries that have many optional "reference" or "benchmark" dependencies
(e.g. to test against a reference implementation), hardcoding them in workflow
YAML creates maintenance burden. Instead, auto-detect them from `pyproject.toml`.

## Pattern

Define optional extras with a consistent prefix in `pyproject.toml`:

```toml
[project.optional-dependencies]
bench-yaml    = ["PyYAML>=6.0"]
bench-http    = ["httpx>=0.28"]
bench-protobuf = ["protobuf>=5.0"]
# ... one per module

dev = [
    "ruff>=0.15.0",
    "ty>=0.0.24",
    "pytest>=8.0",
    "zerodep[bench-yaml,bench-http,bench-protobuf]",  # include all in dev
]
```

Then in CI, dynamically extract and install all matching extras:

```yaml
- name: Install reference libraries
  continue-on-error: true          # don't fail CI if one optional dep is unavailable
  run: |
    EXTRAS=$(python -c "
    import tomllib, pathlib
    data = tomllib.loads(pathlib.Path('pyproject.toml').read_text())
    extras = [k for k in data['project']['optional-dependencies']
              if k.startswith('bench-') and k != 'bench-special']
    print(','.join(extras))
    ")
    echo "Installing extras: $EXTRAS"
    pip install ".[$EXTRAS]"
```

## Why `continue-on-error: true`?

Optional reference libraries may:
- Not yet be released for a new Python version
- Have binary wheels missing for the runner architecture
- Be temporarily unavailable (PyPI outage, git dependency)

Tests that import them are typically guarded with `pytest.importorskip()`, so
the test suite still runs — just skips the comparison tests.

## Handling git-only deps

Some packages aren't on PyPI:

```yaml
- name: Install git-only reference packages
  continue-on-error: true
  run: |
    pip install "special_pkg @ git+https://github.com/org/repo.git" || true
```

The `|| true` ensures the step doesn't fail if the git repo is unreachable.

## Benefits

- Adding a new module with a reference library only requires updating `pyproject.toml`
- No workflow YAML changes ever needed
- `dev` extra always installs everything, so local dev is fully reproducible
- CI and local dev use the same dependency specification
