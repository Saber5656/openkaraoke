# Title

ffmpeg/ffprobe runner and media probe with validation

## Summary

Implement `openkaraoke.media.ffmpeg` (binary discovery + safe subprocess
runner) and `openkaraoke.media.probe` (ffprobe JSON → validated
`MediaInfo`, input classification, limit enforcement).

## Context

All parsing of untrusted media happens inside ffmpeg/ffprobe subprocesses
(DESIGN §8 T1/T2). This module is the only place that spawns them.

## Scope

- `src/openkaraoke/media/ffmpeg.py`, `probe.py`
- Synthetic fixtures generator `scripts/make_fixtures.py` (first version)
- Unit + `@pytest.mark.ffmpeg` integration tests

## Detailed Requirements

1. `ffmpeg.py`:
   - `find_binary(name: Literal["ffmpeg","ffprobe"], cfg) -> Path`: config
     override (`ffmpeg_path`/`ffprobe_path`) else `shutil.which`; missing ⇒
     `EnvMissingError` with per-OS install remedy strings (exact strings in
     code: macOS `brew install ffmpeg`, Debian/Ubuntu
     `sudo apt install ffmpeg`).
   - `run(argv: list[str], *, timeout_s: float, log: StageLog | None,
     capture_stdout: bool) -> CompletedProcess`: `subprocess.run` with
     `shell=False`, `stdin=DEVNULL`, env passthrough minus `LD_PRELOAD`/
     `DYLD_*`; user-supplied file paths must be passed by callers prefixed
     with `./` when relative (helper `safe_path_arg(p)` neutralizes leading
     `-`); on timeout kill the process group and raise `StageError`.
   - `version(cfg) -> tuple[int, int]` parsed from `ffmpeg -version` first
     line; `has_subtitles_filter(cfg) -> bool` via `-hide_banner -filters`
     stdout containing a line matching `^ .{3} subtitles `.
2. `probe.py`:
   - `probe(path, cfg) -> MediaInfo` running
     `ffprobe -v error -print_format json -show_format -show_streams ./<path>`
     with `timeout_s=30`, parsing into pydantic `MediaInfo`:
     `container: str`, `duration_ms: int`, `streams: list[StreamInfo]`
     (`kind: audio|video|other`, `codec: str`, `sample_rate: int | None`,
     `channels: int | None`, `width/height/fps: … | None`).
     Non-zero exit or unparsable JSON ⇒ `InputError` including the first
     200 chars of stderr.
   - `classify(info) -> Literal["audio","video"]`: video iff ≥ 1 video
     stream whose codec is not an attached picture (`mjpeg`/`png` cover art
     with `disposition.attached_pic=1` counts as audio file).
   - `validate_input(info, cfg, *, audio_stream: int | None) -> SelectedInput`:
     duration within `1_000 ms … cfg.max_duration_min*60_000` else
     `InputError` (message cites the config key); at least one audio stream;
     `audio_stream` index in range (default 0 = first audio stream);
     video-size cap 4096×2304 (T1); fps cap 120.
3. `scripts/make_fixtures.py` (stdlib + numpy + soundfile only): writes
   `tests/assets/tone-5s.wav` (5 s, 440 Hz sine + noise, 44.1 kHz stereo,
   ≈ 200 KiB), `tests/assets/colorbar-2s.mp4` (generated via ffmpeg lavfi
   `testsrc2`+`sine`, 320×240), `tests/assets/silence-3s.wav`. Idempotent;
   assets committed. Total committed size < 500 KiB (verify).
4. Integration tests (`@ffmpeg`): probe both fixtures, classify correctly,
   duration within ±100 ms; corrupted file (`tests/assets/garbage.bin` =
   1 KiB of `\x00`) yields `InputError`.
5. Unit tests: `safe_path_arg("-evil.wav") == "./-evil.wav"`; env scrubbing;
   `validate_input` limit matrix.

## Acceptance Criteria

- [ ] No other module in `src/` calls `subprocess` (grep test, extended from
      issue 05 pattern, allowing only `media/ffmpeg.py`).
- [ ] All requirements' behaviors covered by tests; `@ffmpeg` suite passes
      locally and in CI on both OSes.
- [ ] Missing ffmpeg produces exit-5 error with the correct per-OS remedy.
- [ ] mypy strict clean.

## Validation

```bash
uv run python scripts/make_fixtures.py && git status --short tests/assets
uv run pytest tests/media -q -m "not ml"
```

## Dependencies

- 04 (StageLog type), 05 (errors)

## Non-goals

- Audio extraction (08), rendering (15/16), doctor UI (26).

## Design References

- DESIGN §5.1 (probe rules), §8 T1/T2, §10 (media matrix), §11 (fixtures)
