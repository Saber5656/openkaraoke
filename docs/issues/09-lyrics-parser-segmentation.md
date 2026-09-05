# Title

Lyrics parser, normalization, ja/en unit segmentation

## Summary

Implement `openkaraoke.lyrics`: `lyrics.txt` loading/validation
(`parser.py`), text normalization (`normalize.py`), and deterministic
per-language wipe-unit segmentation (`segment.py`) producing the
`LyricsDocument` consumed by alignment and timing.

## Context

Unit segmentation defines what the karaoke wipe animates over; the rules are
fixed in DESIGN §5.3 (text side) and §4.4. This is pure, ML-free code and one
of the most test-heavy modules.

## Scope

- `src/openkaraoke/lyrics/{parser,normalize,segment}.py`
- Table-driven unit tests

## Detailed Requirements

1. `normalize.py`: `normalize_line(s) -> str` = `unicodedata.normalize("NFKC", s)`
   → replace runs of Unicode whitespace with a single ASCII space → strip.
   Exposed separately because LRC import (11) must match against it.
2. `parser.py`: `parse_lyrics_file(path) -> LyricsDocument`:
   - Read bytes; size > 65_536 ⇒ `InputError`. Decode UTF-8 (`utf-8-sig`);
     failure ⇒ `InputError` with byte offset and iconv hint.
   - Reject control chars other than `\n\t\r` (report line/col).
   - Split lines; drop `\r`; lines starting with `#` ignored; runs of ≥ 1
     blank line set `section_break_after=True` on the previous kept line.
   - ≥ 1 non-empty line required; > 300 lines ⇒ `InputError` (DESIGN §8 T3).
   - `LyricsDocument{language: str, lines: list[LyricsLine{index, raw,
     text (normalized), section_break_after}]}` — language passed in by
     caller (from manifest), stored for downstream.
3. `segment.py`: `segment_line(text: str, language: Literal["ja","en"]) -> list[Unit]`
   where `Unit{display: str, align_text: str}`:
   - **ja** (DESIGN §5.3 rules, implement exactly):
     a. Iterate grapheme clusters (use `unicodedata` + manual surrogate-safe
        iteration; graphemes beyond BMP treated as single clusters — no
        external grapheme lib in v1; document the simplification).
     b. New unit per base cluster.
     c. Merge into previous unit: `ぁぃぅぇぉゃゅょっゎァィゥェォャュョッヮーゝゞヽヾ`.
     d. Consecutive `[A-Za-z0-9']+` (after NFKC, i.e. fullwidth already
        folded) form one unit; digits stay grouped with letters run-wise.
     e. Punctuation/space (Unicode categories P*, Z*, S* except `'`) never
        starts a unit: append to previous unit's `display` (or prepend to
        the next unit when line-leading).
   - **en**: split on spaces; token = unit; punctuation stays attached;
     hyphenated tokens are single units.
   - `align_text` = `display` lowercased with all non-letter/kana/kanji/digit
     chars removed (used by alignment tokenization); may be empty (unit is
     then interpolation-only).
   - Per-line unit cap 200 ⇒ `InputError`.
4. Required test table (minimum; each row asserts exact unit `display`
   sequences):
   - `夜に駆ける` → `夜|に|駆|け|る`
   - `きっと` → `きっ|と`  (っ merges backward)
   - `スーパー` → `スー|パー`
   - `ずっとABC123で` → `ずっ|と|ABC123|で`
   - `「夢」を見た。` → `「夢」|を|見|た。`  (leading `「` prepends to the
     next unit; `」` and `。` append to the previous unit)
   - `Hello, world!` (en) → `Hello,|world!`
   - `don't stop` (en) → `don't|stop`
   - `mix ミックス mix` (ja) → `mix|ミッ|ク|ス|mix`
   - 300-line file and 201-unit line rejections.
5. Pure functions; no I/O besides `parse_lyrics_file`; mypy strict.

## Acceptance Criteria

- [ ] All table rows pass; property test: concatenated `display` equals the
      normalized line text for 1,000 random mixed-script lines (hypothesis
      or seeded random generator).
- [ ] Size/control/line-count validations produce exit-4 errors.
- [ ] Coverage ≥ 95% for the three files.

## Validation

```bash
uv run pytest tests/lyrics -q
```

## Dependencies

- 05

## Non-goals

- Ruby/furigana (v2), morphological analysis (v2, U7), LRC (11),
  alignment tokenization to model vocab ids (20).

## Design References

- DESIGN §4.4, §5.3 (text side), §8 T3, §12 U7
