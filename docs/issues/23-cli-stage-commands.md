# Title

CLI stage commands: separate / align / subtitle / render

## Summary

Implement the four single-stage commands with shared stage-execution
plumbing: staleness check, lock, stage logger, manifest record update
(status/params/hashes/tool versions/log path), and consistent output —
delegating the work to issues 18/20/13/16.

## Context

DESIGN §6 table (stage rows) + §3.2 contracts + ADR-005 staleness. This
issue creates the reusable `run_stage` wrapper that `generate` (24) reuses.

## Scope

- `src/openkaraoke/cli/stages.py` (+ small `cli/_stage_runner.py` helper)
- CLI tests with fake backends

## Detailed Requirements

1. `run_stage(workspace, stage_name, worker: Callable, *, force, state) -> StageOutcome`:
   - Load manifest; `acquire_lock`; check upstream stages are `done` and
     not stale (else `WorkspaceError` listing what to run first);
     skip with `already up to date` (exit 0) when not stale and not forced.
   - Open `stage_logger`; mark record `pending→…`; on success write
     `status=done`, `inputs_hash` (recomputed via `stage_inputs`), params,
     `tool_versions`, timestamps, log path; on failure `status=failed` +
     `error` string; save manifest atomically in both cases.
   - Downstream stages: on success, leave records intact (staleness is
     computed, not stored — ADR-005); print a notice listing now-stale
     downstream stages.
2. `separate DIR [--model KEY] [--force]`: backend from 18 (real by
   default; `OPENKARAOKE_FAKE_ML=1` env selects fakes — test seam,
   documented as internal). Params recorded: model key, device.
3. `align DIR [--force] [--offset-ms N]`: runs 20's `run_align_stage`;
   `--offset-ms` post-writes `offset_ms` into timing.json (via 10 models);
   prints the untimed-unit summary and worst-lines warning list when
   non-empty.
4. `subtitle DIR [--force]`: 13's `build_subtitles`; write
   `subtitles/karaoke.ass` atomically; print planning diagnostics.
5. `render DIR [--force] [--output PATH]`: resolve background (14 helper) +
   plan (15) + execute (16); `--output` copies the final file to PATH after
   promotion (PATH parent must exist; outside-workspace write allowed here
   only — explicit user intent, DESIGN §8 T6).
6. All four honor global `--device`/`--offline` via state; exit codes per
   DESIGN §6.2.
7. Tests (fake ML backends, real ffmpeg for subtitle/render on fixtures):
   - Fresh project: `separate` runs; re-run ⇒ `already up to date`.
   - Editing `timing.json` marks subtitle+render stale (via `inspect`-level
     check) and `subtitle` runs while `separate` still skips.
   - Upstream-not-done error message lists the exact command to run.
   - Failure in worker records `status=failed` + error, exit 7, manifest
     still loadable.
   - `render --output` places a copy; `--offline` with missing model ⇒
     exit 6 (fake ensure raising ModelError).

## Acceptance Criteria

- [ ] Full stage lifecycle recorded in manifest for each command (asserted
      field-by-field).
- [ ] Staleness/skip/force matrix behaves per ADR-005 (table test).
- [ ] Exit codes and messages per DESIGN §6/§7.
- [ ] mypy strict clean.

## Validation

```bash
uv run pytest tests/cli/test_stages.py -q
```

## Dependencies

- 13, 16, 18, 20, 22

## Non-goals

- `generate`/`quick` orchestration (24); models/doctor (26).

## Design References

- DESIGN §3.2, §6, §6.2, §7; ADR-005
