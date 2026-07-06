# Title

CLI timing tools and `inspect`

## Summary

Implement `openkaraoke timing lint|export-lrc|import-lrc` and
`openkaraoke inspect` — the user-facing edit-loop and status commands,
including `--json` machine output.

## Context

DESIGN §2.3 (edit loop) and §6 rows. Pure wiring over issues 06/10/11; the
`--json` contract (stdout = single JSON doc, human text = stderr) is set by
DESIGN §6.

## Scope

- `src/openkaraoke/cli/timing.py`, `inspect.py`
- CLI tests

## Detailed Requirements

1. `timing lint DIR [--json]`:
   - Load workspace + timing.json; run 10's `lint` with
     `media_duration_ms` from the manifest.
   - Human output: one line per diagnostic
     `<severity> <code> <line-id>: <message>`, sorted; summary line
     `N errors, M warnings`.
   - `--json`: `{"diagnostics": [...], "errors": N, "warnings": M}`.
   - Exit 0 when no errors (warnings ok), 4 when any error.
2. `timing export-lrc DIR [-o FILE]`: default FILE =
   `<workspace>/lyrics/lyrics.lrc`; `-o` may be outside the workspace
   (explicit user intent). Refuse overwrite without `--force`.
3. `timing import-lrc DIR FILE [--force]`:
   - Parse lyrics from the workspace (09), run 11's `from_lrc`; print its
     warnings; write timing.json (backup previous to
     `lyrics/timing.json.bak` first — single rotating backup).
   - `--force` required when an existing timing.json has `generator`
     metadata (protects auto-aligned data from accidental clobber);
     message explains.
4. `inspect DIR [--json]`:
   - Table columns: stage, status, stale?, params summary (model/device or
     style hash prefix), finished_at, duration, artifact (path + size),
     log. Row per pipeline stage + a header block (slug, language, input
     file, duration, media type).
   - `--json`: full manifest dump + computed `stale` map + artifact sizes.
   - Exit 8 on unloadable workspace (standard), else 0.
5. All human output to stderr when `--json` is active (stdout carries only
   JSON — assert in tests). JSON keys snake_case, stable ordering
   (`sort_keys=True`).
6. Tests: lint outputs and exit codes on fixture workspaces (reuse timing
   fixtures from 10 dropped into a scaffolded workspace); export→import
   round trip inside a workspace; import backup + `--force` gate; inspect
   table renders all stages incl. stale flags after a manual
   timing.json edit; `--json` schema assertions for all three commands.

## Acceptance Criteria

- [ ] Edit loop demo passes as an integration test: align(fake) → edit one
      unit's start_ms → `timing lint` ok → subtitle stage sees staleness
      (via inspect JSON).
- [ ] stdout/stderr discipline under `--json` proven.
- [ ] Backup + generator-guard semantics per req 3.
- [ ] mypy strict clean.

## Validation

```bash
uv run pytest tests/cli/test_timing_inspect.py -q
```

## Dependencies

- 06, 10, 11

## Non-goals

- Interactive/TUI editing (v2 web editor idea); SRT export.

## Design References

- DESIGN §2.3, §4.7, §6 (timing/inspect rows)
