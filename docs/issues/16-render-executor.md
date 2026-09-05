# Title

Render executor: progress, timeout, output verification

## Summary

Implement `openkaraoke.render.executor`: run the argv from issue 15 with
`cwd = workspace`, live progress from ffmpeg's `-progress pipe:1` stream,
hard timeout, stderr capture to the stage log, tmp-file discipline, and
post-render ffprobe verification.

## Context

DESIGN §5.5 "Executor" — the last pipeline stage; failure UX matters
(exit 7 + log tail + no half-written outputs).

## Scope

- `src/openkaraoke/render/executor.py`
- `@ffmpeg` integration tests

## Detailed Requirements

1. `execute_render(workspace, plan, *, cfg, console, log) -> RenderResult{output: Path, duration_ms: int, sha256: str}`:
   - Spawn via `media.ffmpeg.run`-equivalent low-level popen (needs
     streaming; extend `media/ffmpeg.py` with `popen_stream()` if absent —
     keep the subprocess-only-in-that-module rule by placing the popen
     helper there and consuming it here).
   - Parse `key=value` progress lines from stdout; `out_time_us` drives the
     issue-04 progress bar against `plan.duration_ms`; update at most 5/s.
   - stderr → ring buffer (last 200 lines) and full copy into `log`.
   - Timeout: `max(600 s, 10 × duration)`; on expiry kill process group,
     remove tmp, raise `StageError` with remedy "try a faster preset".
   - Non-zero exit: remove tmp; `StageError` message includes last 30
     stderr lines; remedy hints keyed on common patterns
     (`No such filter: subtitles` → libass missing → point to doctor;
     `fontselect` warnings → CJK font guidance).
2. Verification (ffprobe on the tmp file before promote):
   container readable; exactly 1 video + 1 audio stream; duration within
   ±2000 ms of `plan.duration_ms`; else `StageError` and tmp removed.
3. Promote: `os.replace(tmp, output/<slug>.karaoke.mp4)`; compute sha256;
   return result (manifest update happens in the CLI stage wrapper,
   issue 23).
4. Cancellation: SIGINT during render kills ffmpeg group, removes tmp,
   re-raises KeyboardInterrupt (CLI maps to exit 130 — issue 05).
5. Integration tests (`@ffmpeg`):
   - Happy path: 3 s solid-background render with the issue-12 golden ass
     over `silence-3s.wav`; assert file exists, verification passes, sha256
     stable across two runs **except** encoder nondeterminism — assert only
     probe facts, not hash equality.
   - Failure path: corrupt ass file (garbage bytes) ⇒ exit-7 `StageError`,
     no output file, tmp cleaned.
   - Timeout path: patch timeout to 0.1 s ⇒ StageError, tmp cleaned.

## Acceptance Criteria

- [ ] Progress bar receives monotonically increasing values in the happy
      path (asserted via injected fake console).
- [ ] All failure paths leave `output/` free of partial files.
- [ ] Verification bounds enforced (test with a deliberately short `-t`).
- [ ] mypy strict (module in strict set); coverage ≥ 85% (subprocess paths
      partially covered by integration tests).

## Validation

```bash
uv run pytest tests/render/test_executor.py -q
```

## Dependencies

- 04 (progress/log), 15 (argv)

## Non-goals

- CLI wiring/staleness (23), background resolution (14).

## Design References

- DESIGN §5.5 (executor), §7 (render rows), §6.2
