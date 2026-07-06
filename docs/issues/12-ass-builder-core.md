# Title

ASS builder core: styles, wipe events, escaping

## Summary

Implement `openkaraoke.subtitles.ass`: deterministic generation of the ASS
document skeleton (Script Info, V4+ Styles from `StyleConfig`) and karaoke
Dialogue events with `\kf` wipe tags from timed lines — including the
security-relevant text escaping. Zone/preview/countdown planning is issue 13;
this issue renders whatever events it is given.

## Context

DESIGN §5.4 fixes the ASS structure; ADR-004 fixes the technology. The
builder must be a pure function producing byte-stable output (golden tests).

## Scope

- `src/openkaraoke/subtitles/ass.py`
- Golden-file unit tests incl. injection attempts

## Detailed Requirements

1. Public API:
   - `build_ass(events: list[KaraokeEvent], style: StyleConfig) -> str`
   - `KaraokeEvent{style_name: str, layer: int, show_ms: int, hide_ms: int,
     wait_ms: int, units: list[TimedUnit{display, start_ms, end_ms}] ,
     gaps_ms: list[int]}` — a fully resolved event (issue 13 produces these).
     For countdown events, units are the dots.
2. Header: `[Script Info]` with `ScriptType: v4.00+`, `PlayResX/PlayResY`
   from style video w/h, `WrapStyle: 2`, `ScaledBorderAndShadow: yes`,
   `YCbCr Matrix: TV.709`, generator comment `; openkaraoke <version>`.
3. `[V4+ Styles]` Format line exactly:
   `Name, Fontname, Fontsize, PrimaryColour, SecondaryColour, OutlineColour,
   BackColour, Bold, Italic, Underline, StrikeOut, ScaleX, ScaleY, Spacing,
   Angle, BorderStyle, Outline, Shadow, Alignment, MarginL, MarginR,
   MarginV, Encoding`. Emit styles `KaraokeLower`, `KaraokeUpper`,
   `Countdown` with values mapped from `StyleConfig` per DESIGN §5.4
   (Primary=sung, Secondary=upcoming, Outline/Back from colors; Alignment=2;
   MarginV lower = `margin_bottom_px`, upper = `margin_bottom_px +
   zone_gap_px`, countdown = upper + `zone_gap_px`; MarginL/R =
   `margin_side_px`; Encoding=1).
4. Color conversion: `#RRGGBB[AA]` → `&HAABBGGRR&`; ASS alpha = `0xFF -
   AA_in` (input AA FF=opaque; ASS 00=opaque); missing AA ⇒ opaque. Unit
   tests: `#33CCFF` → `&H00FFCC33&`; `#00000080` → `&H7F000000&`.
5. Time format `H:MM:SS.CS` (centiseconds, floor-with-carry per event so the
   sum of emitted `\k`/`\kf` durations equals `hide_ms − show_ms − lead
   residue` within 1 cs — implement carry accumulator exactly as DESIGN §5.4).
6. Event text assembly: `{\k<wait_cs>}` when `wait_ms > 0`, then per unit
   `{\kf<dur_cs>}<escaped display>` with `{\k<gap_cs>}` fillers from
   `gaps_ms` between units.
7. Escaping (`escape_display(s) -> str`): `{`→`｛`, `}`→`｝`, `\`→`＼`;
   strip C0/C1 controls; assert no `\n` remains. Applied to every display
   string, including countdown dots.
8. Output: `\n` line endings, UTF-8 (no BOM), sections separated by one
   blank line, `[Events]` Format:
   `Layer, Start, End, Style, Name, MarginL, MarginR, MarginV, Effect, Text`.
9. Determinism: identical inputs ⇒ identical bytes (no timestamps, no dict
   ordering hazards).

## Acceptance Criteria

- [ ] Golden test: fixture events + default style ⇒ committed `.ass` file,
      byte-identical.
- [ ] Injection tests: displays `{\an8}x`, `}{`, `\N`, `\kf9999`, `{i}`
      render as literal fullwidth-escaped text (assert output contains no
      `{\an8}` etc. outside our generated tags).
- [ ] Color and carry-rounding unit tests per reqs 4–5.
- [ ] Output renders without warnings in libass: `@ffmpeg` smoke test burns
      the golden file over `color=c=black:d=3` and asserts ffmpeg exit 0.
- [ ] mypy strict; coverage ≥ 95%.

## Validation

```bash
uv run pytest tests/subtitles/test_ass.py -q
```

## Dependencies

- 10 (TimedUnit source types), 14 (StyleConfig)

## Non-goals

- Zone rotation / preview / countdown timing decisions (13); rendering (15/16).

## Design References

- DESIGN §5.4, §8 T3
- ADR-004
