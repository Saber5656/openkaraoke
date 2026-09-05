# ADR-004: Wipe rendering via ASS karaoke tags burned in with ffmpeg/libass

- **Status:** Accepted (2026-07-06)

## Context

The confirmed v1 visual is the classic karaoke display: two alternating lyric
zones, per-unit color wipe following the sung position, next-line preview,
intro/gap countdown. Candidate render paths: (a) ASS subtitles with `\kf`
karaoke tags burned in by ffmpeg's `subtitles` filter (libass); (b)
programmatic frame rendering (PIL/moviepy/custom compositor); (c) browser/
canvas rendering (headless Chromium).

## Decision

**(a) ASS + libass burn-in.**

- The `subtitle` stage is a pure function `timing.json + style.toml →
  karaoke.ass` (deterministic, golden-testable text artifact).
- `\kf` (smooth fill) implements the wipe natively; two-zone rotation,
  preview, and countdown are expressed as ASS events (DESIGN §5.4).
- `render` burns subtitles via ffmpeg `subtitles=` filter with
  `cwd = workspace` and fixed relative paths (no filter-escaping surface).

## Alternatives considered

- **Programmatic frame rendering** — rejected: reimplements text shaping,
  CJK line layout, outline/shadow, timing interpolation; slow in Python;
  large bug surface.
- **Headless browser** — rejected: heavyweight dependency, hard to make
  deterministic, security surface.

## Consequences

- Karaoke visuals are constrained to what libass can express — acceptable for
  v1 scope; ruby/furigana (v2) is representable in ASS if needed later.
- ffmpeg must be built with libass → `doctor` check + documented prerequisite.
- A CJK font must exist at render time; we do not bundle fonts (repo size,
  licensing surface); `doctor` detects and recommends Noto Sans CJK.
- The intermediate `.ass` is itself a user-inspectable artifact — debugging
  and manual tweaking are possible with standard tools (Aegisub etc.).
