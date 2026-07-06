# Title

E2E smoke suite with fake ML backends + synthetic fixtures

## Summary

Build the end-to-end smoke test that exercises the entire product surface
(`quick` and the `new → stages → edit-loop → re-render` path) with fake ML
backends and real ffmpeg, finalize the synthetic fixture set, and wire it as
a required CI gate.

## Context

DESIGN §11 "E2E smoke" row. This is the whole-product regression net: it
proves the pipeline contract (ADR-005), the CLI UX, and the failure modes of
DESIGN §7 hold together — without ML weights or copyrighted media.

## Scope

- `tests/e2e/` suite
- `scripts/make_fixtures.py` finalization (all fixtures listed in DESIGN §11)
- `tests/fakes.py` completion (`FakeAlignmentBackend`)
- CI wiring (mark as required job)

## Detailed Requirements

1. `FakeAlignmentBackend`: given parsed lyrics + audio duration, produce a
   deterministic plausible `TimingDocument` (uniform distribution over
   [500 ms, duration−500 ms], units subdivided evenly, confidence 0.9,
   `timed: true`) — same seam as 23 (`OPENKARAOKE_FAKE_ML=1`).
2. Scenario A — quick happy path (audio input):
   `quick tone-5s.wav lyrics-ja-3lines.txt -o out.mp4` ⇒ exit 0; ffprobe
   assertions: h264+aac, yuv420p, duration 5 s ± 2 s, faststart
   (`moov` before `mdat` — probe via `ffprobe -v trace` grep or accept
   flag presence in command goldens from 15); workspace kept.
3. Scenario B — video input + edit loop:
   `new` with `colorbar-2s.mp4` → generate → assert background mode
   source_video was chosen (manifest params) → edit `timing.json` (shift
   one unit +100 ms) → `timing lint` ok → `generate` runs only
   subtitle+render (assert via manifest timestamps unchanged for
   separate/align) → output re-rendered.
4. Scenario C — failure UX: delete `audio/vocals.wav` from a done
   workspace → `generate` detects separate stale → runs it (fake) →
   completes; corrupt `project.json` (truncate) → any command exits 8 with
   the DESIGN §7 remedy string.
5. Scenario D — LRC round trip: `timing export-lrc` → `timing import-lrc`
   → subtitle stale → re-render; diff of ASS files shows only expected
   timestamp changes (none, since round-trip is ±10 ms and equal at cs
   resolution — assert byte-equal ASS).
6. Runtime budget: whole e2e suite ≤ 120 s on CI (fixtures are seconds
   long); mark `@pytest.mark.e2e`; CI runs it in the standard test job
   (both OSes).
7. Fixture finalization: ensure the complete set exists, committed, total
   < 500 KiB: `tone-5s.wav`, `silence-3s.wav`, `colorbar-2s.mp4`,
   `garbage.bin`, malformed-container corpus (27), `lyrics-ja-3lines.txt`,
   `lyrics-en-3lines.txt`, timing/LRC fixtures (10/11). `make_fixtures.py
   --check` verifies hashes of committed assets (regeneration
   reproducibility).

## Acceptance Criteria

- [ ] Scenarios A–D green on Linux + macOS CI with `-m e2e`.
- [ ] Fixture `--check` green; asset budget respected (test asserts
      cumulative size).
- [ ] No network, no ML imports during e2e (socket-block + import-recorder
      fixtures active).
- [ ] CI marks the e2e job required (branch protection note for issue 32).

## Validation

```bash
uv run pytest tests/e2e -q -m e2e
uv run python scripts/make_fixtures.py --check
```

## Dependencies

- 16, 24 (full CLI path); 27 recommended first (shares fixtures)

## Non-goals

- Real-model quality testing (21); performance benchmarking (30 docs).

## Design References

- DESIGN §11 (E2E row, fixture policy), §7, §2.2–§2.4; ADR-005
