# ADR-002: Vocal separation via `audio-separator` with a pinned BS-RoFormer default model

- **Status:** Accepted (2026-07-06)

## Context

We need 2-stem separation (vocals / instrumental) that runs locally on Apple
Silicon (MPS/CoreML), CUDA, and CPU. Research (docs/research §2) found:
`facebookresearch/demucs` archived 2025-01-01 (fork = bug fixes only);
`python-audio-separator` actively maintained (2026), wraps UVR-project models
(MDX, VR, Demucs, BS/Mel-RoFormer) behind one API, auto-downloads models,
supports CUDA + CoreML + CPU, Python ≥ 3.10, MIT with a UVR attribution
request. Best current general model: `model_bs_roformer_ep_317_sdr_12.9755.ckpt`
(vocals 12.9 SDR / instrumental 17.0 SDR).

## Decision

- Depend on **`audio-separator`** as the single v1 separation backend, behind
  our `SeparationBackend` protocol (DESIGN §5.2) so it stays swappable.
- Default model pinned: **`model_bs_roformer_ep_317_sdr_12.9755.ckpt`**;
  filename + sha256 recorded in our model registry; downloads land in the
  openkaraoke cache dir (not the library default `/tmp` path).
- Credit UVR and audio-separator in README/NOTICE per their request.

## Alternatives considered

- **demucs direct** — rejected: upstream archived; htdemucs SDR below current
  RoFormer models.
- **Own ONNX inference of MDX/RoFormer** — rejected for v1: large effort,
  reinvents a maintained wheel.
- **Cloud APIs (AudioShake etc.)** — rejected: violates fully-local principle.

## Consequences

- RoFormer `.ckpt` checkpoints are pickle-based → supply-chain mitigation via
  hash pinning + verification (ADR-006, DESIGN §8 T4); investigate safetensors
  availability (known unknown U4).
- CoreML acceleration effectiveness for RoFormer must be measured (U2);
  worst case documented CPU fallback on macOS.
- audio-separator API changes are absorbed in one adapter module (issue 18).
