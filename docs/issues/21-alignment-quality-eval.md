# Title

Alignment quality eval harness + go/no-go playbook

## Summary

Build the evaluation harness (`scripts/eval_alignment.py`) and the manual
annotation playbook (`docs/VALIDATION.md`) that measure real-song alignment
quality against hand-annotated references, and record the go/no-go decision
for the v1 aligner in a dated research report.

## Context

DESIGN §12 U1: alignment quality on Japanese singing is the project's main
technical risk. The gate (median unit-onset error ≤ 150 ms, p90 ≤ 400 ms on
the ja set) decides whether ADR-003's in-house aligner ships as default or a
fallback backend gets scheduled. Copyright constraint: reference songs are
**never committed** (DESIGN §11).

## Scope

- `scripts/eval_alignment.py`
- `docs/VALIDATION.md` (annotation + evaluation playbook)
- `docs/research/alignment-eval-TEMPLATE.md` (report template)

## Detailed Requirements

1. Reference format: a *reference timing file* is a normal `timing.json`
   whose unit boundaries were human-corrected (produced by running
   `align` then fixing in an editor with audio scrubbing; playbook explains
   using Audacity label export → provided converter helper inside the
   script: `--from-audacity labels.txt lyrics.txt`).
2. `eval_alignment.py` CLI (argparse, runnable via `uv run python`):
   - `evaluate REF.json HYP.json [--json]`: match lines by id; per timed
     unit compute onset error `|hyp.start − ref.start|` and offset error;
     report per-song and aggregate: median, mean, p90, p99, % units
     > 400 ms, % untimed. Text table + optional JSON.
   - `run --project DIR --ref REF.json`: re-runs the align stage with
     current code (imports `run_align_stage`), then evaluates — one-command
     iteration loop for tuning.
   - Exit code 1 when the ja gate thresholds are exceeded (for scripted
     tracking), 0 otherwise; thresholds as flags with DESIGN defaults.
3. `docs/VALIDATION.md` playbook:
   - Test-set definition: minimum 3 Japanese songs (ballad / up-tempo /
     rap-adjacent dense lyrics) + 1 English song, 1 full song each,
     maintainer-owned audio, stored outside the repo (path convention
     `~/openkaraoke-eval/<slug>/`).
   - Annotation procedure (expected effort ≈ 20 min/song): correct only
     line starts + every 5th unit + audibly wrong units; document the
     partial-annotation convention (`unannotated units excluded from
     metrics` — the script honors a `"ref": true` marker per unit; units
     without the marker are excluded).
   - Go/no-go recording procedure: copy the template to
     `docs/research/alignment-eval-<yyyy-mm-dd>.md`, fill metrics tables,
     tuning attempts (chunk size, model revision), decision + rationale.
4. Template contains: environment table (device, versions), per-song metric
   tables, aggregate gate table with PASS/FAIL, decision section, follow-up
   issue checklist (pre-filled with the U1 fallback options from
   ISSUE_PLAN).
5. Harness has unit tests with synthetic ref/hyp pairs (metric math, marker
   exclusion, gate exit codes). No committed audio.

## Acceptance Criteria

- [ ] `evaluate` produces correct metrics on synthetic pairs (hand-computed
      expectations).
- [ ] Audacity-label conversion round-trips a crafted label file.
- [ ] Playbook executable by a person who has only read DESIGN §2/§5.3
      (review sign-off by the maintainer counts).
- [ ] A dated eval report exists with a recorded GO or NO-GO decision and
      is referenced from ISSUE_PLAN validation strategy item 3.
- [ ] NO-GO path: follow-up issue(s) filed per U1 before closing this issue.

## Validation

```bash
uv run pytest tests/eval -q
uv run python scripts/eval_alignment.py evaluate tests/assets/eval/ref.json tests/assets/eval/hyp.json
# manual: full playbook run on the 4-song local set; report committed
```

## Dependencies

- 18 (separation for real songs), 20 (aligner under test)

## Non-goals

- Automatic dataset download; committing any copyrighted material;
  CI execution of real-song evals.

## Design References

- DESIGN §11 (alignment quality row), §12 U1/U7, §9 (numbers written back)
