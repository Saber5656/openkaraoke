# Title

Console output and logging module

## Summary

Implement `openkaraoke.logging`: rich-based console output respecting
`-v/-vv/--quiet`, plus per-stage file logging into the workspace `logs/`
directory, and a progress-reporting helper used by long stages.

## Context

Every stage needs consistent human output (status lines, warnings, progress
bars) and a persisted log file whose path is referenced from the manifest and
error messages (DESIGN §6.2 "log path on failure").

## Scope

- `src/openkaraoke/logging.py`
- Unit tests

## Detailed Requirements

1. `setup_console(verbosity: int, quiet: bool) -> Console` — returns a rich
   `Console` writing to **stderr** (stdout is reserved for `--json` payloads).
   Levels: quiet ⇒ errors only; default ⇒ info; `-v` ⇒ debug; `-vv` ⇒ debug +
   tracebacks enabled (see `errors.py` integration, issue 05).
2. `stage_logger(workspace: Path, stage: str) -> StageLog` context manager:
   - Creates `logs/<stage>-<yyyymmdd-HHMMSS>.log` (workspace-relative,
     directory created if missing), UTF-8, line-buffered.
   - `StageLog.path` (relative Path for manifest), `.write(line)`,
     `.tee(line, level)` = console + file.
   - On `__exit__` with exception: writes the traceback to the file always
     (regardless of console verbosity) and flushes.
3. `progress(console, total: float | None, label: str)` wrapper around
   `rich.progress.Progress` yielding an `update(completed: float)` callable;
   renders a spinner when `total is None`. Disabled automatically when the
   console is not a TTY or `--quiet` (falls back to periodic log lines at
   most every 5 s).
4. Standard message helpers: `info/warn/error(console, msg)` producing the
   fixed prefixes `•` / `⚠` / `✖` so tests can assert output stably.
5. No global mutable state: everything flows from explicitly passed objects
   (`Console`, `StageLog`). No `logging` stdlib root-logger configuration.
6. Third-party library noise (torch/transformers warnings) suppression is
   NOT handled here (that's per-backend, issues 18/19).

## Acceptance Criteria

- [ ] Verbosity matrix behaves per req 1 (asserted via capsys/StringIO
      console injection).
- [ ] `stage_logger` creates the file, tees, and records tracebacks on error.
- [ ] Progress falls back to log lines when not a TTY.
- [ ] stdout stays empty in all logging paths (asserted in tests).
- [ ] mypy strict clean.

## Validation

```bash
uv run pytest tests/logging -q
```

## Dependencies

- 01, 03 (Path conventions; no config values are read here, only types)

## Non-goals

- ffmpeg progress parsing (issue 16); JSON output modes (per-command issues).

## Design References

- DESIGN §6 (stdout/stderr contract), §6.2 (log path in errors), §4.1 (`logs/`)
