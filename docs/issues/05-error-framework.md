# Title

Error framework and exit codes

## Summary

Implement `openkaraoke.errors`: the exception hierarchy mapped 1:1 to the
exit-code table of DESIGN §6.2, plus the top-level CLI error handler that
renders cause → remedy → log path and controls traceback visibility.

## Context

Every subsequent issue raises these exceptions instead of ad-hoc ones; the
exit-code table is a public contract (used in docs and tests).

## Scope

- `src/openkaraoke/errors.py`
- CLI integration in `cli/__init__.py` (wrap command invocation)
- Unit tests

## Detailed Requirements

1. Exception classes (all subclass `OpenKaraokeError(Exception)` which
   carries `message: str`, `remedy: str | None`, `log_path: Path | None`):

   | Class | Exit code |
   |---|---|
   | `ConfigError` | 3 |
   | `InputError` | 4 |
   | `EnvironmentError_` (avoid builtin clash; name `EnvMissingError`) | 5 |
   | `ModelError` | 6 |
   | `StageError` | 7 |
   | `WorkspaceError` | 8 |
   | (any other exception) | 10 |

   Click usage errors keep click's exit code 2.
2. `EXIT_CODES: dict[str, int]` constant + `exit_code_for(exc) -> int`.
3. CLI handler (decorator or group-level `invoke` override):
   - Catches `OpenKaraokeError`: prints via issue-04 `error()` one block:
     `✖ <message>` / optional `→ <remedy>` / optional `log: <log_path>`;
     exits with the mapped code.
   - Catches bare `Exception`: prints `✖ internal error: <repr>` + a hint to
     re-run with `-vv` and file a bug at the repository issues URL; exit 10.
   - With `-vv`: full rich traceback printed before the block, for both cases.
   - `KeyboardInterrupt`: print `interrupted`, exit 130, no traceback.
4. All output to stderr. No `sys.exit` calls anywhere else in the codebase
   (enforced by a grep-based unit test scanning `src/` for `sys.exit(` —
   allowed only in `errors.py` and `cli/__init__.py`).
5. Docstring table in `errors.py` mirrors DESIGN §6.2 verbatim.

## Acceptance Criteria

- [ ] Raising each class from a dummy command exits with its table code and
      prints the message/remedy/log lines when set.
- [ ] Unknown exceptions exit 10 with the bug-report hint.
- [ ] `-vv` shows tracebacks; default does not.
- [ ] Grep test passes (no stray `sys.exit`).
- [ ] mypy strict clean.

## Validation

```bash
uv run pytest tests/errors -q
```

## Dependencies

- 01 (04 for pretty printing is a soft dependency: use plain stderr prints if
  04 has not landed, then adopt `error()` in 04's PR)

## Non-goals

- Per-stage failure detection logic (each stage issue).

## Design References

- DESIGN §6.2 (exit codes), §7 (failure modes)
