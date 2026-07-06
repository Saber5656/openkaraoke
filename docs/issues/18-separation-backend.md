# Title

Separation backend (audio-separator adapter)

## Summary

Define the `SeparationBackend` protocol and implement the v1 adapter around
`audio_separator.separator.Separator` with pinned default model, forced
output naming, device selection, cache-dir integration, and error mapping —
plus a `FakeSeparationBackend` for tests.

## Context

ADR-002. The adapter isolates the third-party API so its changes never leak
past this module. ML imports must be lazy (DESIGN §3.3).

## Scope

- `src/openkaraoke/separation/base.py`, `audio_separator_backend.py`
- `tests/fakes.py` (FakeSeparationBackend)
- `@ml` integration test

## Detailed Requirements

1. `base.py`:
   - `SeparationResult{vocals: Path, instrumental: Path, model_key: str,
     tool_versions: dict[str, str]}`
   - `class SeparationBackend(Protocol): def separate(self, source_wav: Path,
     out_dir: Path, *, device: str, log: StageLog, console) -> SeparationResult`
   - `resolve_device(requested: Literal["auto","cpu","cuda","mps"]) -> str`:
     auto ⇒ first available of cuda, mps/coreml, cpu (probe lazily inside
     try/except ImportError; no torch import at module load).
2. `audio_separator_backend.py`:
   - Constructor takes `cfg` + model key (default
     `separation/bs_roformer_ep317`); calls `models_cache.ensure` first
     (honors offline), passes the cache dir as `model_file_dir`.
   - Configure `Separator(output_dir=…, output_format="wav",
     output_names={"Vocals": "vocals", "Instrumental": "instrumental"})`
     — verify exact parameter names against the installed version at
     implementation time; if the API differs, adapt here only and note it
     in the PR.
   - Silence the library's own logging into our `StageLog` (attach handler
     to its logger; nothing to stdout).
   - Post-conditions: both wav files exist, same duration as source
     ±500 ms (soundfile), samplerate 44100; else `StageError` with log tail.
   - Vocals-silence heuristic (DESIGN §7): RMS of vocals < −45 dBFS ⇒
     warning "input may already be instrumental; alignment may fail".
   - Map library exceptions: OOM/`RuntimeError` mentioning memory ⇒
     `StageError` with remedy `--device cpu` or shorter input; missing model
     ⇒ `ModelError`.
   - Record `tool_versions = {"audio_separator": <ver>, "torch": <ver>,
     "onnxruntime": <ver?>}` for the manifest.
3. `FakeSeparationBackend(tests/fakes.py)`: copies the source wav to
   `instrumental.wav` and writes 300 ms of shaped noise as `vocals.wav`
   (deterministic seed) — instant, ML-free, used by 23/24/29 tests.
4. `@ml` integration test (skipped unless `OPENKARAOKE_ML_TESTS=1`):
   run on `tests/assets/tone-5s.wav` with the real model on cpu; assert
   post-conditions and that measured device fallback is logged. Also record
   wall time + peak RSS into the test log (informational, feeds U2/U5).

## Acceptance Criteria

- [ ] Protocol + fake used by at least one green pipeline-level test.
- [ ] Post-conditions and error mapping covered by unit tests with a mocked
      Separator class.
- [ ] Lazy imports proven: `python -c "import openkaraoke.separation.base"`
      works without torch/audio-separator installed (CI job leg without
      `ml` extra).
- [ ] `@ml` test passes locally on maintainer hardware (documented in PR).

## Validation

```bash
uv run pytest tests/separation -q
OPENKARAOKE_ML_TESTS=1 uv run pytest tests/separation -m ml -q   # local
```

## Dependencies

- 08 (source wav exists upstream), 17 (ensure/cache)

## Non-goals

- Chunked separation for long songs (U5 follow-up), model selection UX (26),
  4-stem modes.

## Design References

- DESIGN §5.2, §7, §12 U2/U5; ADR-002
