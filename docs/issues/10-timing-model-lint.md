# Title

Timing document model and lint

## Summary

Implement `openkaraoke.timing.model` (pydantic models for `timing.json`
schema v1, load/save) and `openkaraoke.timing.lint` (the invariant checker
with structured diagnostics) — the contract for the user edit loop.

## Context

`timing.json` is written by alignment (20), hand-edited by users, imported
from LRC (11), and consumed by the ASS builder (12). DESIGN §4.3 fixes the
schema and the five invariants.

## Scope

- `src/openkaraoke/timing/model.py`, `lint.py`
- Unit tests + JSON fixtures

## Detailed Requirements

1. `model.py` (pydantic, `extra="forbid"`):
   - `TimingUnit{text: str, start_ms: int, end_ms: int, confidence: float | None, timed: bool}`
     (`ge=0` on times, `0.0–1.0` on confidence).
   - `TimingLine{id: str (pattern ^L\d{3,}$), text: str,
     section_break_after: bool = False, start_ms: int, end_ms: int,
     units: list[TimingUnit] (min 1)}`.
   - `TimingDocument{schema_version: Literal[1], language: str, offset_ms: int = 0,
     generator: GeneratorInfo | None, lines: list[TimingLine] (min 1)}`.
   - `load_timing(path) -> TimingDocument` (`InputError` on JSON/validation
     failure quoting the pydantic error paths);
     `save_timing(path, doc)` atomic, `indent=2`, `ensure_ascii=False`.
   - `apply_offset(doc) -> TimingDocument`: returns a copy with `offset_ms`
     folded into all times (clamped ≥ 0) and `offset_ms=0` — used by the
     subtitle stage only; the stored file keeps the offset explicit.
2. `lint.py`: `lint(doc, *, media_duration_ms: int | None) -> list[Diagnostic]`
   with `Diagnostic{severity: Literal["error","warning"], code: str,
   line_id: str | None, message: str}` and codes:

   | code | severity | rule (DESIGN §4.3) |
   |---|---|---|
   | `unit-order` | error | units sorted, start ≤ end, no overlap |
   | `line-order` | error | lines sorted by start_ms |
   | `line-overlap` | warn ≤ 500 ms / error > 2000 ms | inter-line overlap bands |
   | `duration-exceeded` | error | last end > media duration + 1000 (only when duration given) |
   | `text-drift` | warning | line.text ≠ concat(units.display) |
   | `unit-short` | warning | duration < 60 ms (error < 30 ms → code `unit-invalid`) |
   | `untimed-ratio` | warning | > 30% units `timed: false` in a line |

   Deterministic ordering: by (line index, code).
3. `lint_errors(diags) -> bool` helper (any severity == error).
4. Fixtures: `tests/assets/timing/valid-ja.json`, plus one fixture per
   diagnostic code, hand-written, referenced by table-driven tests.
5. mypy strict; no I/O besides load/save.

## Acceptance Criteria

- [ ] Valid fixture loads and round-trips byte-identically (module owns
      canonical serialization).
- [ ] Each diagnostic code triggered by exactly its fixture; severities per
      table.
- [ ] `apply_offset` clamps at 0 and zeroes the stored offset.
- [ ] Unknown JSON keys rejected with a path-bearing message.
- [ ] Coverage ≥ 95%.

## Validation

```bash
uv run pytest tests/timing -q
```

## Dependencies

- 05; 09 (uses the same normalized-text convention for `text-drift`)

## Non-goals

- LRC (11), CLI wiring of `timing lint` (25), alignment output writing (20).

## Design References

- DESIGN §4.3, §2.3 (edit loop), §7 (subtitle failure row)
