# Title

Enhanced LRC import/export

## Summary

Implement `openkaraoke.timing.lrc`: export of `TimingDocument` to enhanced
LRC (line tags + `<...>` word tags) and import back, including line-only LRC,
with the text-matching rules of DESIGN §4.7.

## Context

LRC is the interoperability path for users who prefer external LRC editors
over editing `timing.json` directly.

## Scope

- `src/openkaraoke/timing/lrc.py`
- Unit tests with golden files

## Detailed Requirements

1. Export `to_lrc(doc: TimingDocument) -> str`:
   - Header lines in order: `[re:openkaraoke]`, `[ve:1]`,
     `[offset:<offset_ms>]` (omitted when 0).
   - Per line: `[mm:ss.xx]<mm:ss.xx>unit1<mm:ss.xx>unit2…<mm:ss.xx>`
     where `[..]` = line start, each `<..>` precedes its unit (unit start),
     and the trailing `<..>` = line end. Times centisecond-rounded
     (`floor(ms/10 + 0.5)`); `mm` zero-padded 2+, minutes may exceed 59.
   - Unit text emitted verbatim (display text). Newline `\n`, file ends with
     one `\n`. UTF-8.
2. Import `from_lrc(text: str, lyrics: LyricsDocument) -> TimingDocument`:
   - Accept enhanced and line-only LRC; ignore standard metadata tags
     (`[ar:] [ti:] [al:] [by:] [la:]` etc.); parse `[offset:±N]` into
     `offset_ms`.
   - Multiple line-tags on one physical line (repeated lyrics) are expanded
     into separate entries.
   - Match each LRC lyric line to `lyrics.lines` **in order** after
     `normalize_line` (issue 09): a mismatch is an `InputError` reporting up
     to 3 diffs as `line N: expected «…» got «…»`. Extra/missing lines are
     mismatches.
   - Enhanced import: word-tag count must equal unit count (from
     `segment_line`) — else fall back to line-level timing for that line
     with a `warning` diagnostic returned alongside
     (`from_lrc` returns `(TimingDocument, list[Diagnostic])`).
   - Line-only import: one unit per line spanning `[start, next_start or
     start+4000]`, `timed: True`, plus `lrc-line-only` warning per line.
   - End times: line end = trailing word tag if present, else next line
     start clamped to +30 ms minimum, last line +4000 ms.
3. Round-trip guarantee test: `from_lrc(to_lrc(doc), lyrics)` reproduces all
   times within ±10 ms (centisecond resolution) and identical unit texts.
4. Reject files > 256 KiB, invalid timestamps (`InputError` with line no).
5. mypy strict, pure functions.

## Acceptance Criteria

- [ ] Golden export file for the `valid-ja.json` fixture matches
      byte-for-byte.
- [ ] Round-trip property holds on all timing fixtures.
- [ ] Line-only and enhanced imports produce documented warnings.
- [ ] Text mismatch produces the 3-diff error format.
- [ ] Coverage ≥ 95%.

## Validation

```bash
uv run pytest tests/timing/test_lrc.py -q
```

## Dependencies

- 10 (models), 09 (normalize/segment)

## Non-goals

- CLI commands (25); SRT or other formats.

## Design References

- DESIGN §4.7, §2.3
