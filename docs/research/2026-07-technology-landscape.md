# Technology Landscape Research — Karaoke Video Generation (2026-07)

- **Date of research:** 2026-07-06 (all links accessed on this date)
- **Purpose:** Verify the current state of the external components and comparable
  projects that materially affect the openkaraoke v1 design (see `docs/DESIGN.md`).
- **Method:** Web search + primary-source verification (GitHub repos, official docs).
  Facts below are dated; re-verify at implementation time where marked.

---

## 1. Comparable projects and positioning

| Project | What it does | Key dependencies | Why openkaraoke differs |
|---|---|---|---|
| [nomadkaraoke/karaoke-gen](https://github.com/nomadkaraoke/karaoke-gen) | Fully automated karaoke video generation (4K MP4, CDG+MP3), YouTube/Dropbox upload | **AudioShake paid cloud API** for transcription; fetches lyrics from Genius/Spotify/Musixmatch; MDX + Demucs separation | openkaraoke is **fully local, no paid APIs, no lyrics scraping** (user supplies lyrics), Japanese-first |
| [rakuri255/UltraSinger](https://github.com/rakuri255/UltraSinger) | Generates UltraStar Deluxe game files (txt + pitch + midi) from songs | Whisper (ASR), Demucs, pitch detection; >8 GB VRAM for large models | Different product category (sing-along game data, not karaoke videos); ASR-driven, not user-lyrics-driven |
| [Bowen951209/open-karaoke-toolkit](https://github.com/Bowen951209/open-karaoke-toolkit) | Karaoke lyrics animation maker (GUI, in development) | — | Animation editor, not an end-to-end audio→video pipeline |
| [OpenKJ](https://github.com/OpenKJ/OpenKJ), [KaraokeEternal](https://github.com/bhj/KaraokeEternal), [PiKaraoke](https://github.com/vicwomg/pikaraoke) | Karaoke hosting / party-queue systems | — | Playback/queueing category; they *consume* karaoke videos that openkaraoke *produces* |
| Youka, Moises (commercial) | Cloud karaoke/stem services | Proprietary | Closed, paid, cloud-processing |

**Positioning conclusion:** there is a clear gap for a *fully local, open-source,
user-lyrics-first* karaoke **video** generator with first-class Japanese support.
No GitHub project named `openkaraoke` / `open-karaoke` exists as of 2026-07-06
(closest: `open-karaoke-toolkit`), so the name is usable.

Sources:
- https://github.com/nomadkaraoke/karaoke-gen
- https://github.com/rakuri255/UltraSinger
- https://github.com/Bowen951209/open-karaoke-toolkit
- https://github.com/topics/karaoke

## 2. Vocal separation backends

| Fact | Evidence |
|---|---|
| `facebookresearch/demucs` was **archived 2025-01-01** (read-only); maintainer left Meta; fork `adefossez/demucs` receives important bug fixes only | [facebookresearch/demucs](https://github.com/facebookresearch/demucs), [issue #554](https://github.com/facebookresearch/demucs/issues/554) |
| [`python-audio-separator`](https://github.com/nomadkaraoke/python-audio-separator) (PyPI: `audio-separator`) is actively maintained as of mid-2026; wraps UVR-project models (MDX-Net, VR, Demucs, BS/Mel-RoFormer) behind one `Separator` API | GitHub repo + [PyPI](https://pypi.org/project/audio-separator/) |
| Recommended default model: `model_bs_roformer_ep_317_sdr_12.9755.ckpt` — vocals 12.9 SDR / instrumental 17.0 SDR; maintainer calls it the go-to for clean full-spectrum separation | [Discussion #133](https://github.com/nomadkaraoke/python-audio-separator/discussions/133) |
| Acceleration: CUDA (NVIDIA) and **CoreML on Apple Silicon** via ONNX Runtime providers; CPU fallback. Python ≥ 3.10. Models are auto-downloaded on first use to a configurable directory | README ([raw](https://raw.githubusercontent.com/nomadkaraoke/python-audio-separator/main/README.md)) |
| Licensing: package is MIT; UVR-trained model weights — maintainers ask to "honor the MIT license by providing credit to UVR and its developers". No non-commercial restriction found for the default BS-RoFormer checkpoints; the BS-RoFormer architecture implementation is MIT | audio-separator README, [lucidrains/BS-RoFormer LICENSE](https://github.com/lucidrains/BS-RoFormer/blob/main/LICENSE) |

**Decision input:** depend on `audio-separator`, pin the default model filename +
hash, give attribution to UVR in README/NOTICE. Do **not** depend on `demucs`
directly (archived upstream). Per-model license re-verification is an
implementation-time checklist item (see issue 17).

Risk to re-verify at implementation time: RoFormer checkpoints are pickle-based
torch checkpoints (`.ckpt`) — see the supply-chain section of the security model
(DESIGN.md §8) for the pinning/verification mitigation.

## 3. Lyrics-to-audio alignment (forced alignment)

Requirement recap: v1 aligns **user-provided lyrics text** to the **separated
vocal stem** (no ASR transcription in v1).

| Option | Findings | Verdict for v1 |
|---|---|---|
| [WhisperX](https://github.com/m-bain/whisperx) alignment stage | Active; aligns via wav2vec2 CTC models; **`ja` supported via `jonatasgrosman/wav2vec2-large-xlsr-53-japanese`**; produces char + word timestamps; handles out-of-vocabulary chars with a **wildcard emission column** (max non-blank score per frame); interpolates missing timestamps. License: **BSD-2-Clause**. | Algorithm blueprint — but the package drags in the full ASR stack (faster-whisper/ctranslate2, pyannote) that v1 does not need. Rejected as a dependency; reimplement the alignment technique (with attribution). |
| [ctc-forced-aligner](https://github.com/MahmoudAshraf97/ctc-forced-aligner) (MMS-300m based) | Alignment-only and light, but the MMS aligner vocabulary is **Latin-only**; CJK text must be romanized via `uroman`, and mapping romanized tokens back to Japanese characters is fragile | Rejected for a Japanese-first product |
| Montreal Forced Aligner | Most accurate word timestamps in comparisons ([whisperX issue #1247](https://github.com/m-bain/whisperX/issues/1247)) | Rejected: conda-centric install, dictionary/G2P workflow; poor fit for an end-user CLI |
| **In-house thin aligner** on `torchaudio.functional.forced_align` + Hugging Face wav2vec2 CTC models | torchaudio entered a maintenance phase (decode/encode APIs removed in 2.9, migrated to TorchCodec), **but `forced_align` was explicitly preserved after user feedback**; torchaudio 2.10 (Jan 2026) completed the migration with `forced_align` intact. Official CTC forced-alignment API tutorial exists. | **Chosen.** Alignment-only, small dependency surface (torch, torchaudio, transformers), full control over Japanese unit segmentation. |

Consequences for the design:

- Do **not** use torchaudio for audio I/O (those APIs were removed in 2.9);
  decode audio via ffmpeg into wav and load with `soundfile`.
- Japanese alignment model: `jonatasgrosman/wav2vec2-large-xlsr-53-japanese`
  (character-level vocabulary), pinned by revision. English: torchaudio bundled
  wav2vec2 ASR model or MMS-FA. Registry is per-language and extensible.
- Alignment quality on **singing** (vs speech) is the main open risk → dedicated
  evaluation-spike issue with go/no-go criteria (issue 21). Known mitigations:
  align against the separated vocal stem; interpolate low-confidence units.

Sources:
- https://github.com/m-bain/whisperX (README, `whisperx/alignment.py`, LICENSE)
- https://github.com/MahmoudAshraf97/ctc-forced-aligner
- https://github.com/pytorch/audio/issues/3902 ("Update on TorchAudio's future")
- https://docs.pytorch.org/audio/stable/tutorials/ctc_forced_alignment_api_tutorial.html
- https://huggingface.co/jonatasgrosman/wav2vec2-large-xlsr-53-japanese

## 4. Packaging: uv + PyTorch accelerator matrix

- Astral publishes an official guide for PyTorch with uv: accelerator-specific
  builds live on dedicated indexes (e.g. `https://download.pytorch.org/whl/cpu`,
  `/whl/cu130`), selected via optional-dependency extras + `[tool.uv]` index
  configuration with `explicit = true`. Requires uv ≥ 0.5.3.
- On macOS the default PyPI wheels include MPS support (no custom index needed).
- `audio-separator` ships its own extras for ONNX Runtime variants (CPU/GPU);
  the interplay between our torch extras and its onnxruntime extras must be
  verified in the packaging issue (issue 30).

Sources:
- https://docs.astral.sh/uv/guides/integration/pytorch/

## 5. Rendering (stable, low-risk facts)

- ASS subtitle karaoke tags (`\k`, `\kf`) burned in with ffmpeg's `subtitles`
  filter (libass) is the established technique for wipe-style karaoke text;
  ffmpeg must be built with libass (Homebrew/distro builds are).
- CJK display requires a CJK-capable font at render time; Noto Sans CJK (OFL)
  is the recommended install; font presence is a `doctor` check, not a bundled
  asset (repo stays small, no font licensing surface).

These are long-stable facts; no further research needed.

## 6. Summary of decisions fed into ADRs

| Decision | ADR |
|---|---|
| Python ≥ 3.11, uv-managed, src layout | ADR-001 |
| Separation via `audio-separator`, pinned BS-RoFormer default | ADR-002 |
| In-house CTC forced alignment (torchaudio + HF wav2vec2), whisperX rejected as dep | ADR-003 |
| ASS (`\kf`) + ffmpeg/libass burn-in renderer | ADR-004 |
| Per-song workspace directory + manifest as the pipeline contract | ADR-005 |
| Offline-first network policy; pinned model registry with hash verification | ADR-006 |
