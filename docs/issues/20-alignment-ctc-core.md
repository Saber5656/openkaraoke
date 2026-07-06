# Title

CTC forced alignment core: spans → timing.json

## Summary

Implement `openkaraoke.alignment.ctc`: tokenization of lyric units against
the model vocab with the wildcard-column technique, invocation of
`torchaudio.functional.forced_align` + `merge_tokens`, mapping spans back to
units, confidence + interpolation rules, sanity gates, and writing the final
`timing.json`.

## Context

DESIGN §5.3 "Alignment core" steps 1–6 defines the algorithm precisely;
ADR-003 documents the wildcard technique's origin (whisperX, BSD-2-Clause —
reimplementation with attribution, no code copying).

## Scope

- `src/openkaraoke/alignment/ctc.py`
- Unit tests with synthetic emissions (hand-crafted log-prob matrices)

## Detailed Requirements

1. `tokenize_units(lines: list[list[Unit]], vocab, char_level: bool) ->
   TokenPlan`:
   - Per unit, per character of `align_text`: vocab id if present, else
     **wildcard id `V`** (= vocab size; one virtual column).
   - en (char_level=False still uses char tokens for wav2vec2 ASR vocab —
     letters + `|`): join words with `word_sep_id` between units.
   - Empty `align_text` units contribute zero tokens (recorded for
     interpolation).
   - `TokenPlan` records per-unit token index ranges for reverse mapping.
2. `add_wildcard_column(log_probs) -> np.ndarray`: append column =
   `max` over non-blank real columns per frame (DESIGN §5.3.1).
3. `align(log_probs, plan, blank_id) -> list[UnitSpan]`:
   - `torchaudio.functional.forced_align(torch.from_numpy(lp)[None],
     targets[None], blank=blank_id)`; then
     `torchaudio.functional.merge_tokens(labels, scores)` → token spans.
   - Group spans to units via `TokenPlan` ranges; unit
     `start_ms/end_ms/confidence` per DESIGN §5.3.3 (round, clamp
     `end ≥ start + 30`).
4. Post-processing per DESIGN §5.3.4:
   - `timed=False` for zero-token units, wildcard-only units, or
     `confidence < 0.15`; linear interpolation between nearest timed
     neighbors **within the line**; whole-line untimed ⇒ uniform
     distribution between neighbor lines' boundaries and a per-line report
     entry.
   - Line `start_ms/end_ms` from first/last unit.
5. Sanity gate (DESIGN §5.3.5): untimed ratio > 30% or non-monotonic line
   starts ⇒ `StageError` whose message embeds a report of the 10 worst
   lines (`line id, text, untimed %, mean confidence`) and the three likely
   causes verbatim from DESIGN §7.
6. `run_align_stage(workspace, manifest, cfg, state) -> TimingDocument`:
   orchestrates: ensure `analysis.wav` (via 08 from `vocals.wav`), load
   model (19), emissions (19), steps 1–5, `save_timing` (10), returns doc +
   diagnostics list; called by CLI issue 23.
7. Synthetic tests (no ML): craft emissions where the correct alignment is
   known analytically:
   - 3 units, clean diagonal emissions ⇒ exact expected ms values.
   - OOV char in unit 2 ⇒ wildcard path taken, unit 2 `timed` semantics per
     threshold; interpolation between units 1 and 3 verified numerically.
   - Empty-align-text unit interpolated.
   - Gate test: emissions forcing > 30% wildcard ⇒ StageError with report.
   - Rounding/clamp edge: 10 ms span ⇒ end = start + 30.
8. Attribution docstring in `ctc.py` naming whisperX + BSD-2-Clause for the
   wildcard technique.

## Acceptance Criteria

- [ ] All synthetic scenarios produce exact expected numbers (tolerances
      ±1 ms from rounding only).
- [ ] `timing.json` written by `run_align_stage` passes `timing lint` with
      zero errors on the happy synthetic path.
- [ ] Gate triggers with the specified report format.
- [ ] Determinism: same inputs ⇒ identical timing.json bytes.
- [ ] mypy (relaxed set) + coverage ≥ 90% for pure functions.

## Validation

```bash
uv run pytest tests/alignment/test_ctc.py -q
```

## Dependencies

- 10 (timing model), 19 (emissions/vocab)

## Non-goals

- Model quality tuning (21), ASR fallback (v2), ruby (v2).

## Design References

- DESIGN §5.3 (core, steps 1–6), §7 (align rows); ADR-003
