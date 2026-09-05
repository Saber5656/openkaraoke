# Title

Audio extraction/resample module

## Summary

Implement `openkaraoke.media.extract`: ffmpeg-based derivation of the two
working audio files — `audio/source.wav` (44.1 kHz stereo, from the input
media) and `audio/analysis.wav` (16 kHz mono, from an arbitrary wav, used on
`vocals.wav` by the align stage).

## Context

DESIGN §5.1 step 4 and §5.3 audio-side step 1. torchaudio must not be used
for I/O (ADR-003); ffmpeg is the single decoder.

## Scope

- `src/openkaraoke/media/extract.py`
- `@ffmpeg` integration tests

## Detailed Requirements

1. `extract_source_wav(input_media: Path, out_wav: Path, *, audio_stream: int, cfg, log) -> None`:
   `ffmpeg -y -i ./<input> -map 0:a:<n> -ac 2 -ar 44100 -c:a pcm_s16le -f wav ./<out>`
   via `media.ffmpeg.run`, `timeout_s = 600`. Out path parent must exist.
2. `to_analysis_wav(in_wav: Path, out_wav: Path, *, cfg, log) -> None`:
   `-ac 1 -ar 16000 -c:a pcm_s16le`, `timeout_s = 300`.
3. Both verify the output afterwards with `soundfile.info`: correct
   samplerate/channels, frames > 0; mismatch ⇒ `StageError` (message includes
   ffmpeg log tail of 30 lines).
4. Overwrite semantics: write to `<out>.tmp.wav`, verify, `os.replace`.
5. Mono/multichannel inputs: `-ac 2` upmix/downmix is accepted behavior
   (document in docstring).
6. Tests (`@ffmpeg`): source extraction from `colorbar-2s.mp4` (video) and
   `tone-5s.wav` (audio); analysis conversion of `tone-5s.wav` → assert
   16000 Hz mono via soundfile; interrupted-write simulation leaves no
   partial final file.

## Acceptance Criteria

- [ ] Both functions produce verified files with exact sample formats.
- [ ] tmp+replace semantics proven by test.
- [ ] No decoding library besides ffmpeg/soundfile used (no torchaudio import
      — grep test).
- [ ] mypy strict clean.

## Validation

```bash
uv run pytest tests/media/test_extract.py -q
```

## Dependencies

- 07

## Non-goals

- Loudness/gain (render stage option), stem separation (18).

## Design References

- DESIGN §5.1 (step 4), §5.3 (audio side), §3.2 (stage table)
- ADR-003 (no torchaudio I/O)
