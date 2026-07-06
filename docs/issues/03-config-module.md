# Title

Configuration module: layered config + platformdirs

## Summary

Implement `openkaraoke.config`: pydantic v2 schemas for the user-level
application config, layered loading (built-in defaults → user config file →
CLI overrides), and platform-correct paths for config and cache directories.

## Context

Multiple modules need configuration (default language, cache dir override,
ffmpeg path override, duration caps) and the standard directories (model
cache). Style config (`style.toml`) is a separate schema owned by issue 14;
this issue provides the shared loading machinery both use.

## Scope

- `src/openkaraoke/config/schema.py` — `AppConfig` model
- `src/openkaraoke/config/loader.py` — paths + layered loading
- Unit tests

## Detailed Requirements

1. `AppConfig` (pydantic `BaseModel`, `extra="forbid"`, all fields defaulted):
   - `default_language: Literal["ja", "en"] = "ja"`
   - `max_duration_min: int = 20` (range 1–60)
   - `cache_dir: Path | None = None` (None ⇒ platformdirs default)
   - `ffmpeg_path: Path | None = None`, `ffprobe_path: Path | None = None`
   - `device: Literal["auto","cpu","cuda","mps"] = "auto"`
   - `hf_home: Path | None = None` (exported as `HF_HOME` for transformers
     downloads when set)
2. Paths API (`loader.py`):
   - `config_file_path() -> Path` = `platformdirs.user_config_path("openkaraoke") / "config.toml"`
   - `cache_root(cfg) -> Path` = `cfg.cache_dir` or
     `platformdirs.user_cache_path("openkaraoke")`; subdirs `models/`, `hf/`.
   - Both created lazily with mode 0o700 on POSIX.
3. `load_app_config(explicit_path: Path | None) -> AppConfig`:
   - Order: defaults → TOML file (explicit path if given, else default path;
     missing file is fine, unreadable/invalid file is exit-3 error) → env
     override `OPENKARAOKE_CONFIG` (path, same as explicit) — explicit CLI
     path wins over env.
   - TOML parsed with `tomllib`. Validation errors are wrapped in
     `ConfigError` (from issue 05; until 05 lands, raise `ValueError` with
     the same message format and add a TODO referencing issue 05) reporting
     the file path and the pydantic error list (one line per error:
     `key.path: message`).
4. CLI global flags (`--device`, `--offline`, `--config`) do NOT mutate
   `AppConfig`; they live in `AppState` (issue 01) — provide
   `effective_device(cfg, state) -> str` helper that resolves precedence
   (CLI > config > "auto").
5. No I/O at import time. Loader functions are pure given inputs; directory
   creation only in `cache_root`.
6. Tests: defaults; file overrides; forbid-extra error message contains the
   unknown key path; precedence explicit-path > env > default path; invalid
   TOML syntax produces the exit-3-style error; `cache_root` creates dirs
   with 0o700.

## Acceptance Criteria

- [ ] `load_app_config(None)` returns defaults when no file exists.
- [ ] A config file with `default_language = "en"` changes the value; an
      unknown key `foo = 1` fails with a message containing `foo`.
- [ ] `cache_root` honors `cache_dir` override and creates `models/` lazily.
- [ ] mypy strict clean; coverage of the module ≥ 95%.

## Validation

```bash
uv run pytest tests/config -q
uv run mypy src/openkaraoke/config
```

## Dependencies

- 01

## Non-goals

- `style.toml` schema (issue 14); project manifest (issue 06); doctor
  reporting of config (issue 26).

## Design References

- DESIGN §4.5 (style is separate), §6 (global flags), §8 T7
- ADR-001, ADR-006 (cache dir role)
