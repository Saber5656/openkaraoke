# ADR-003: In-house CTC forced alignment (torchaudio + per-language wav2vec2), not whisperX

- **Status:** Accepted (2026-07-06)

## Context

v1 requirement is **forced alignment of user-provided lyrics** to the isolated
vocal stem — no ASR. whisperX solves alignment well (char/word timestamps,
`ja` via `jonatasgrosman/wav2vec2-large-xlsr-53-japanese`, wildcard handling
for out-of-vocab chars) but pulls the full ASR stack (faster-whisper /
ctranslate2 / pyannote) we don't need. `ctc-forced-aligner` (MMS) is Latin-
vocabulary only → romanization round-trip is fragile for Japanese. MFA is the
accuracy gold standard but conda/dictionary-based — wrong shape for an
end-user CLI. torchaudio's `forced_align` API was explicitly **preserved**
through torchaudio's maintenance-phase migration (research §3).

## Decision

Implement a thin alignment core in-repo (DESIGN §5.3):

- Emissions via HF `transformers` wav2vec2 CTC models from a **pinned
  per-language registry** (`ja` → `jonatasgrosman/wav2vec2-large-xlsr-53-japanese`
  @ pinned revision; `en` → torchaudio `WAV2VEC2_ASR_BASE_960H`), computed in
  overlapping 30 s chunks and concatenated.
- DP alignment via `torchaudio.functional.forced_align` + `merge_tokens`.
- Out-of-vocab chars → wildcard emission column (max non-blank per frame) —
  technique from whisperX (BSD-2-Clause), **reimplemented, not copied**, with
  attribution in the module docstring.
- Low-confidence/wildcard units marked `timed: false` and interpolated.

## Alternatives considered

- **whisperX as dependency** — rejected: dependency weight, ASR stack unused,
  install friction; kept as documented fallback backend if the in-house
  aligner misses the quality gate (issue 21 go/no-go).
- **ctc-forced-aligner** — rejected: CJK romanization detour.
- **MFA** — rejected: install/dictionary UX.

## Consequences

- We own the tricky bits (chunk stitching, wildcard, interpolation) — spec'd
  to mechanical detail in issues 19/20; quality gated by the issue-21 spike
  before `align` is considered done.
- Audio I/O must avoid torchaudio (decode APIs removed in 2.9): ffmpeg →
  wav → `soundfile`.
- `AlignmentBackend` protocol keeps whisperX/MFA insertable later without CLI
  changes.
