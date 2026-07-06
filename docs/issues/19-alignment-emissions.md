# Title

Alignment emissions: chunked wav2vec2 log-probs

## Summary

Implement `openkaraoke.alignment.emissions` and `alignment/registry.py`:
loading the per-language wav2vec2 CTC model (via issue 17's cache), running
chunked inference over `analysis.wav`, and returning the concatenated
log-probability matrix plus vocabulary metadata — the input to issue 20's
DP core.

## Context

DESIGN §5.3 "Audio side" steps 1–4 and ADR-003. Chunking with overlap
trimming bounds memory for full songs.

## Scope

- `src/openkaraoke/alignment/{registry,emissions}.py`
- Unit tests (mocked model) + numeric tests with a tiny real model optional

## Detailed Requirements

1. `registry.py`: `AlignModelSpec{key: str, kind: "hf"|"torchaudio",
   ref: str, revision: str | None, char_level: bool}`;
   entries `ja` (hf, `jonatasgrosman/wav2vec2-large-xlsr-53-japanese`,
   pinned revision, char_level=True), `en` (torchaudio
   `WAV2VEC2_ASR_BASE_960H`, char_level=False). Unsupported language ⇒
   `ModelError`.
2. `load_model(spec, *, device, offline, cfg) -> LoadedAligner{model,
   vocab: dict[str, int], blank_id: int, sample_rate: 16000}`:
   - hf: `transformers.Wav2Vec2ForCTC.from_pretrained(ref,
     revision=spec.revision, use_safetensors=True (fall back with a logged
     warning if the pinned revision lacks safetensors), local_files_only=offline)`
     + `Wav2Vec2Processor` for the vocab; env `HF_HOME` pointed at our cache
     (§17) during the call.
   - torchaudio: bundle API; vocab from `bundle.get_labels()`.
   - Vocab normalization: keys lowercased; word-delimiter token (`|`)
     identified and exposed as `word_sep_id`; blank from config
     (`pad_token_id`).
   - `model.eval()`, `torch.inference_mode()` everywhere;
     `torch.manual_seed(0)` at load.
3. `compute_emissions(aligner, wav_path, *, device, console, log) ->
   Emissions{log_probs: np.ndarray [T, V] float32, frame_ms: float}`:
   - Load wav via `soundfile` (assert 16 kHz mono; else `StageError` —
     caller bug).
   - Window 30.0 s, hop 28.0 s. For each window: tensor → device → forward →
     `log_softmax` → cpu float32 numpy.
   - Concatenate with edge trimming per DESIGN §5.3.3: interior windows drop
     frames corresponding to the first/last 1.0 s; first window keeps its
     head, last keeps its tail. Frame bookkeeping in *frames*, computed from
     the model's measured ratio, not hard-coded.
   - `frame_ms = (n_samples / 16000 * 1000) / T_total`; assert
     `15 ≤ frame_ms ≤ 25` else `StageError` (unexpected architecture).
   - Progress via issue-04 helper (windows count); peak-memory guard: catch
     OOM `RuntimeError` → `StageError` remedy `--device cpu` /
     smaller chunk env `OPENKARAOKE_ALIGN_CHUNK_S` (int, default 30,
     min 10 — read once, documented as tuning knob for U1/U5).
4. Determinism: two runs over the same file produce bitwise-equal arrays on
   cpu (test with mocked small model; real-model equality not asserted).
5. No torchaudio I/O; no network beyond `models_cache.ensure`/HF cache path.

## Acceptance Criteria

- [ ] Mock-model tests verify: window/hop frame math (exact expected T for
      synthetic durations 5 s, 65 s, 90 s), trimming correctness (stitch a
      ramp signal and assert no duplicated/missing frames), frame_ms assert.
- [ ] Vocab normalization exposes `blank_id`, `word_sep_id`, lowercase keys.
- [ ] Offline honored end-to-end (socket-block test with cached mock).
- [ ] Lazy ML imports (module importable without torch).
- [ ] mypy passes (module in relaxed set), coverage ≥ 85% via mocks.

## Validation

```bash
uv run pytest tests/alignment/test_emissions.py -q
```

## Dependencies

- 08 (analysis.wav production), 17 (model ensure)

## Non-goals

- DP alignment/spans (20), quality evaluation (21), VAD segmentation (v2).

## Design References

- DESIGN §5.3 (audio side), §12 U1/U5; ADR-003
