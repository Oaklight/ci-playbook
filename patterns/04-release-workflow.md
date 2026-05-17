# 04 — Release workflow (CalVer)

Automated release process for Python libraries using CalVer (`YYYY.M.D`).

## Flow

```
1. Merge all changes → master, CI green
2. Update changelogs (move [Unreleased] → [YYYY.M.D])
3. Trigger Release workflow (workflow_dispatch)
4. Workflow: lint+test → version bump → commit → tag → GitHub Release
5. On release publish: benchmark + PyPI workflows trigger automatically
```

## Full release.yml

```yaml
name: Release

on:
  workflow_dispatch:
    inputs:
      version:
        description: "Release version (leave empty for auto CalVer YYYY.M.D)"
        required: false
        default: ""

permissions:
  contents: write

jobs:
  quality:
    uses: ./.github/workflows/lint-test.yml
    with:
      python-versions: '["3.12"]'

  release:
    needs: quality
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Determine version
        id: version
        run: |
          if [ -n "${{ github.event.inputs.version }}" ]; then
            VER="${{ github.event.inputs.version }}"
          else
            VER="$(date +%Y.%-m.%-d)"
          fi
          if git tag -l "v${VER}" | grep -q .; then
            echo "::error::Tag v${VER} already exists"
            exit 1
          fi
          echo "version=${VER}" >> "$GITHUB_OUTPUT"
          echo "Release version: ${VER}"

      - name: Bump project version
        run: |
          VER="${{ steps.version.outputs.version }}"
          sed -i "s/^version = .*/version = \"${VER}\"/" pyproject.toml

      - name: Commit and tag
        run: |
          VER="${{ steps.version.outputs.version }}"
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add pyproject.toml
          if git diff --cached --quiet; then
            echo "No changes to commit"
          else
            git commit -m "release: v${VER}"
          fi
          git tag "v${VER}"
          git push origin master --tags

      - name: Create GitHub Release
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          VER="${{ steps.version.outputs.version }}"
          gh release create "v${VER}" \
            --title "v${VER}" \
            --generate-notes
```

## Cascade triggers

Downstream workflows trigger on `release: published` — no changes needed
when adding new modules or extras:

```yaml
# benchmark.yml / pypi.yml
on:
  release:
    types: [published]
```

## Notes

- `fetch-depth: 0` is required for `git tag -l` to see existing tags
- The duplicate-tag guard (`git tag -l | grep -q`) prevents re-releasing the same day
- `permissions: contents: write` is required for push + tag
- CalVer handles same-day releases via manual `version` input (e.g. `2026.5.17.1`)
- Always quality-gate before the release job using `needs: quality`
