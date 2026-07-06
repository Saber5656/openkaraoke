# Title

Project scaffolding: uv package, src layout, CLI entry stub

## Summary

Create the installable Python package skeleton for openkaraoke: pyproject
managed by uv, `src/openkaraoke/` layout, a click-based `openkaraoke` console
command with global flags and a working `--version`, plus lint/type/test
tooling configuration. No product features.

## Context

Everything else builds on this skeleton. Stack decisions are fixed by
ADR-001 (Python ≥ 3.11, uv, click, pydantic v2, rich, ruff, mypy, pytest).
The module map in DESIGN §3.3 defines the package layout; create the tree with
placeholder modules so later issues fill in files without restructuring.

## Scope

- `pyproject.toml`, `uv.lock`, `.gitignore`, `.python-version`
- `src/openkaraoke/` package tree (empty modules with docstrings only)
- CLI group with global options; `--version`
- ruff + mypy + pytest configuration; pre-commit config
- `LICENSE` (MIT, copyright holder: the repository owner)

## Detailed Requirements

1. `pyproject.toml`:
   - `[project]`: name `openkaraoke`, version `0.1.0.dev0`, description
     "Generate a karaoke video from any song", `requires-python = ">=3.11,<3.14"`,
     license `MIT`, `dependencies = ["click>=8.1", "pydantic>=2.7", "rich>=13", "platformdirs>=4", "soundfile>=0.12", "numpy>=1.26"]`.
   - `[project.optional-dependencies]`: `ml = ["audio-separator>=0.28", "torch>=2.8", "torchaudio>=2.8", "transformers>=4.48", "huggingface_hub>=0.30"]`
     (exact bounds re-verified in issue 30; keep these as initial values),
     `dev = ["pytest>=8", "pytest-cov", "mypy>=1.14", "ruff>=0.8", "pre-commit"]`.
   - `[project.scripts]`: `openkaraoke = "openkaraoke.cli:main"`.
   - Build backend: `hatchling`. src layout.
2. Package tree exactly as DESIGN §3.3 (create every listed module as a file
   containing a one-line module docstring; `cli/__init__.py` is real code).
3. `src/openkaraoke/__init__.py` exposes `__version__` via
   `importlib.metadata.version("openkaraoke")` with a `0.0.0-dev` fallback
   when the package is not installed.
4. `openkaraoke.cli:main`: click `Group` with global options stored in a
   `click.Context.obj` dataclass `AppState(verbosity: int, quiet: bool,
   device: str, offline: bool, config_path: Path | None)`:
   `-v/--verbose` (count), `--quiet`, `--device` choice
   `[auto,cpu,cuda,mps]` default `auto`, `--offline` flag, `--config PATH`,
   `--version`. Register a hidden placeholder subcommand `_noop` so the group
   is executable; later issues add real subcommands.
5. Tooling config (in `pyproject.toml`):
   - ruff: `line-length = 100`, enable `E,F,W,I,UP,B,S` (S = bandit rules;
     allow `S603/S607` only in `media/ffmpeg.py` via per-file ignore later),
     format = ruff-format.
   - mypy: `strict = true` for `openkaraoke.*` except overrides
     `openkaraoke.separation.*`, `openkaraoke.alignment.*` (non-strict, ML).
   - pytest: `testpaths = ["tests"]`, markers declared: `ffmpeg`, `ml`.
6. `tests/test_smoke.py`: asserts `openkaraoke --version` exits 0 via
   `click.testing.CliRunner` and version string matches `^\d+\.\d+\.\d+`.
   (dev0 suffix allowed: regex `^\d+\.\d+\.\d+`. Use `re.match`.)
7. `.pre-commit-config.yaml`: ruff (lint+format) hooks only.
8. Do not add product logic, network calls, or ML imports anywhere.

## Acceptance Criteria

- [ ] `uv sync --all-extras` succeeds on macOS arm64 and Linux x86_64.
- [ ] `uv run openkaraoke --version` prints the version, exit 0.
- [ ] `uv run openkaraoke --help` shows all global options from req 4.
- [ ] `uv run ruff check .`, `uv run ruff format --check .`,
      `uv run mypy src`, `uv run pytest` all pass.
- [ ] Package tree matches DESIGN §3.3 exactly (file-for-file).
- [ ] `uv.lock` committed; `git status` clean after build.

## Validation

```bash
uv sync --all-extras
uv run openkaraoke --version
uv run openkaraoke --help
uv run ruff check . && uv run ruff format --check .
uv run mypy src
uv run pytest -q
```

All commands exit 0; help output lists `-v, --quiet, --device, --offline, --config, --version`.

## Dependencies

None (first issue).

## Non-goals

- CI workflows (issue 02), config loading (03), logging (04), errors (05).
- Any subcommand behavior beyond `--version`/`--help`.

## Design References

- DESIGN §3.3 (module map), §3.4 (dependency policy)
- ADR-001 (stack)
