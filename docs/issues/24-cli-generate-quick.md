# Title

CLI `generate` orchestrator and `quick`

## Summary

Implement `openkaraoke generate` (runs pending/stale stages in pipeline
order with `--until`/`--force`) and `openkaraoke quick` (one-shot: create a
workspace next to the input, generate, optionally copy the output and
optionally discard the workspace).

## Context

DESIGN §2.2/§2.4 and §6 rows. Reuses `run_stage` from 23 and
`stages_to_run` from 06 — no new stage logic.

## Scope

- `src/openkaraoke/cli/generate.py`, `quick.py`
- CLI tests with fake ML backends

## Detailed Requirements

1. `generate DIR [--force] [--until STAGE]`:
   - Compute `stages_to_run(workspace, manifest, until, force)`; empty ⇒
     print `everything up to date` and exit 0.
   - Print the plan first (`will run: separate → align → …`), then execute
     sequentially via `run_stage`; stop at first failure (its exit code
     propagates); summary block at the end (per-stage ✓/✖/skip + total
     wall time + output path when render ran).
   - `--until` validates against `STAGE_ORDER` (usage error 2 otherwise).
2. `quick INPUT LYRICS [-o OUT.mp4] [--language L] [--keep/--no-keep]`:
   - Workspace dir = `<INPUT dir>/<INPUT stem>.okproj` (slugified stem);
     existing dir ⇒ exit 8 with remedy "use `new`/`generate` on it or
     remove it" (never auto-reuse — predictability over convenience).
   - Runs the `new` flow (22) then full `generate`.
   - `-o`: copy final mp4 to OUT (parent must exist); default: leave in
     workspace and print its path.
   - `--no-keep`: on **success only**, delete the workspace after copying
     the output (requires `-o`; usage error 2 if `--no-keep` without `-o`);
     deletion uses the confinement guard and refuses if the dir does not
     look like a workspace (`project.json` present) — safety against
     misdirected deletion.
   - On failure: workspace always kept + hint printed
     (`inspect`/stage rerun guidance).
3. Both commands honor `--offline`/`--device`; interrupted runs (SIGINT)
   leave a consistent manifest (in-flight stage `failed` or untouched —
   covered by 23's wrapper; test here at orchestrator level).
4. Tests: fresh generate runs all 4 stages in order (fake backends; assert
   call order); `--until align` stops; second run skips all; mid-pipeline
   failure propagates exit 7 and later stages don't run; quick happy path
   produces OUT.mp4 and keeps/deletes per flags; quick on existing dir ⇒ 8;
   `--no-keep` without `-o` ⇒ 2; `--no-keep` refuses non-workspace dir.

## Acceptance Criteria

- [ ] Plan/summary output format stable (golden-ish assertions on lines).
- [ ] Stage execution strictly sequential and ordered; failure short-
      circuits.
- [ ] `quick` end-to-end passes with fake ML + real ffmpeg in CI.
- [ ] Workspace deletion safety checks proven by tests.
- [ ] mypy strict clean.

## Validation

```bash
uv run pytest tests/cli/test_generate_quick.py -q
```

## Dependencies

- 23

## Non-goals

- Parallel stage execution; watch/daemon modes; batch input lists (v2).

## Design References

- DESIGN §2.2, §2.4, §6 (generate/quick rows); ADR-005
