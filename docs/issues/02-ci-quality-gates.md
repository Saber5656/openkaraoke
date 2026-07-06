# Title

CI workflows: lint, type-check, tests (Linux + macOS)

## Summary

Add GitHub Actions workflows that gate every PR with ruff, mypy, and pytest on
a Linux + macOS matrix (Python 3.11 and 3.13), with ffmpeg installed so
`@pytest.mark.ffmpeg` integration tests run, plus a scheduled dependency-audit
job. Actions pinned by commit SHA.

## Context

DESIGN §11 defines the PR gates; DESIGN §8 T8/T9 requires supply-chain
hygiene in CI. This issue sets the guardrails before feature code lands.

## Scope

- `.github/workflows/ci.yml` (PR + push to non-main branches)
- `.github/workflows/audit.yml` (weekly schedule + manual dispatch)
- `.github/dependabot.yml`

## Detailed Requirements

1. `ci.yml` jobs:
   - `lint` (ubuntu-latest): `uv sync --extra dev` → `ruff check .` →
     `ruff format --check .`
   - `typecheck` (ubuntu-latest): `uv run mypy src`
   - `test`: matrix `os: [ubuntu-latest, macos-14]` ×
     `python: ["3.11", "3.13"]`; steps: install uv (official
     `astral-sh/setup-uv` action, SHA-pinned), `uv python install
     ${{ matrix.python }}`, install ffmpeg (ubuntu: `apt-get install -y
     ffmpeg`; macos: `brew install ffmpeg`), `uv sync --extra dev`,
     `uv run pytest -m "not ml" --cov=openkaraoke --cov-report=xml`,
     enforce coverage: fail if total coverage of the pure modules
     (`lyrics, timing, subtitles, render.command, workspace, config,
     models_cache.registry`) is below 85% (use `--cov-fail-under=85` with
     `[tool.coverage.run] source` narrowed to those packages — configure in
     `pyproject.toml` here).
   - ML marker excluded from CI (`-m "not ml"`).
2. All third-party actions referenced by **full commit SHA** with a
   version comment (T9). `permissions: contents: read` at workflow level.
3. `audit.yml`: weekly cron + `workflow_dispatch`; runs `uv run pip-audit`
   (add `pip-audit` to dev extras) and fails on known vulnerabilities;
   `permissions: contents: read`.
4. `dependabot.yml`: ecosystems `github-actions` (weekly) and `pip` (weekly,
   versioning-strategy: lockfile-only is not supported for uv — use
   `package-ecosystem: "uv"` if available on the platform at implementation
   time, otherwise `pip` against `pyproject.toml`; leave a comment noting the
   check).
5. Concurrency group cancels superseded runs per ref.
6. Badge for CI status added to README (single line; do not otherwise edit
   README in this issue).

## Acceptance Criteria

- [ ] A PR touching Python code triggers lint, typecheck, and 4 test matrix
      legs; all green on the scaffolding code.
- [ ] `pytest -m ffmpeg` tests execute (not skipped) on both OSes in CI once
      such tests exist (verify wiring with a trivial ffmpeg-version test
      added under `tests/test_ci_ffmpeg.py` in this issue).
- [ ] Every `uses:` line is SHA-pinned with a version comment.
- [ ] audit workflow runs green via manual dispatch.
- [ ] Workflow files pass `actionlint` locally (document the command in the
      PR description; do not add actionlint to CI in this issue).

## Validation

```bash
gh workflow run audit.yml && gh run watch
gh pr checks   # on the PR introducing this — all required checks green
actionlint .github/workflows/*.yml
```

## Dependencies

- 01 (scaffolding, dev extras)

## Non-goals

- Release/publish workflows, CodeQL, secret scanning, branch-protection
  settings (issue 32).
- Nightly ML integration job (post-v1 follow-up if wanted).

## Design References

- DESIGN §11 (testing strategy), §8 T8/T9
- ADR-001
