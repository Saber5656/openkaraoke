# Title

ASS display program: zones, preview, countdown

## Summary

Implement `openkaraoke.subtitles.program`: the planner that turns a
`TimingDocument` + `StyleConfig` into the resolved `KaraokeEvent` list for
issue 12's builder — two-zone alternation, lead-in/lead-out with collision
shifting, and intro/gap countdowns — plus the top-level
`build_subtitles(timing, style) -> str` used by the subtitle stage.

## Context

DESIGN §5.4 "Two-zone rotation program" and "Countdown" specify the exact
scheduling rules. This is pure logic over integers — ideal for exhaustive
table tests.

## Scope

- `src/openkaraoke/subtitles/program.py`
- Unit tests

## Detailed Requirements

1. Zone assignment: line index 0 → `KaraokeLower`, 1 → `KaraokeUpper`,
   alternating (0-based even=lower). Layer always 0.
2. Windows: `show = start − lead_in_ms`, `hide = end + lead_out_ms`
   (post-`apply_offset` times; clamp show ≥ 0).
3. Collision rule per zone: if `show < previous_same_zone.hide`, set
   `show = previous.hide`; if that pushes `show > start`, set
   `show = start` and log a `warning` diagnostic (`zone-congestion`) —
   the wipe start is never delayed (wait_cs may be 0).
4. `wait_ms = start − show`; `gaps_ms[i] = units[i+1].start − units[i].end`
   (≥ 0 by lint; values < 10 ms folded into the previous unit's duration to
   avoid `\k0` spam — spec: add to previous `dur`, gap 0).
5. Countdown events (style `Countdown`):
   - Intro: when `countdown_intro` and first line `start ≥ 6000`:
     dots = `countdown_dots`, each dot `{\kf100}●` (1000 ms), wipe ends at
     first line `start`; event `show = start − dots*1000 − 500`,
     `wait_ms = 500`, `hide = start`.
   - Gap: for each adjacent pair with `next.start − prev.end ≥
     countdown_gap_threshold_ms`: same shape ending at `next.start`.
   - Dots use display `●` and are passed through the standard escaping.
6. `build_subtitles(timing: TimingDocument, style: StyleConfig) ->
   tuple[str, list[Diagnostic]]`: apply_offset → plan events → `build_ass`;
   diagnostics from planning (zone-congestion) returned to the caller
   (subtitle stage prints them as warnings).
7. Required test scenarios (exact expected event tables in the test file):
   - 4 lines, no overlaps: alternation, windows, waits.
   - Tight lines forcing the collision rule both branches.
   - Intro countdown at start=12_000 (dots 5): show=6_500, wait=500.
   - No intro countdown when start=4_000.
   - Gap countdown triggered at exactly the threshold; not at threshold−1.
   - Sub-10 ms gaps folded (no `\k0`).
   - offset_ms folding respected end-to-end.

## Acceptance Criteria

- [ ] All scenario tables pass; goldens for one full ASS via
      `build_subtitles` on the `valid-ja.json` fixture (byte-stable).
- [ ] No event has `show < 0`, `hide ≤ show`, or negative wait/gap.
- [ ] mypy strict; coverage ≥ 95%.

## Validation

```bash
uv run pytest tests/subtitles/test_program.py -q
```

## Dependencies

- 12

## Non-goals

- Ruby, per-line style overrides, duet colors (v2).

## Design References

- DESIGN §5.4 (rotation, countdown), §4.5 `[timing_display]`
