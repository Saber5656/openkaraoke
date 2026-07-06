# Title

User documentation, README, legal notices

## Summary

Write the public-facing documentation: full README (bilingual EN primary +
short JA section), usage guide with the three canonical flows, troubleshooting,
and the legal notice about processing copyrighted content — aligned with the
product boundaries (no fetching features, user responsibility).

## Context

DESIGN §1/§2 define the product story; §8 T10 requires the legal posture to
be explicit. This issue is docs-only but has acceptance criteria tied to
accuracy against the shipped CLI.

## Scope

- `README.md` (rewrite; keeps the original one-liner 任意の曲からカラオケ動画を生成する)
- `docs/USAGE.md`, `docs/TROUBLESHOOTING.md`
- Legal notice section (README) + cross-links from SECURITY.md/NOTICE (28)

## Detailed Requirements

1. `README.md` structure: badges (CI) → one-liner (JA + EN) → 30-second
   demo block (commands of §2.2 + a representative screenshot placeholder
   `docs/images/demo.png` generated from a synthetic render — commit a real
   frame grab from the fixture pipeline, not copyrighted content) →
   features list (v1 truthful only) → install (link INSTALL.md; ffmpeg +
   font prerequisites inline) → quick start → timing-correction loop
   (§2.3 verbatim commands) → project layout summary (§4.1 abridged) →
   **Legal notice** → attribution (UVR/audio-separator, whisperX technique,
   model list link to NOTICE) → license (MIT) → 日本語セクション: 概要 +
   クイックスタートのみ (JA translation of quick start).
2. Legal notice (exact points, EN + JA):
   - The tool processes files you provide; it downloads nothing but
     AI models.
   - Outputs typically contain copyrighted musical works and lyrics; you
     are responsible for having the rights for your use (private use,
     licensed covers, own works); link to SECURITY.md scope note.
   - The project distributes no copyrighted media and will not accept
     contributions adding content downloading/scraping (contribution
     policy line, also added to CONTRIBUTING.md → issue 32 file if not
     yet present: create `CONTRIBUTING.md` stub here with build/test
     basics + this policy).
3. `docs/USAGE.md`: every command of DESIGN §6 with one worked example,
   flags table, exit-code table (§6.2 verbatim), the staleness model
   explained for users (when things re-run), style.toml reference
   (§4.5 field-by-field with the template as source of truth), LRC
   workflow, offline workflow.
4. `docs/TROUBLESHOOTING.md`: symptom → cause → fix table covering every
   DESIGN §7 row plus: garbled/tofu glyphs (font), "subtitles filter not
   found", MPS/CUDA OOM, model download blocked by proxy, alignment
   quality bad (→ edit loop + eval playbook link), Apple quarantine of
   downloaded models (xattr note).
5. Accuracy gate: a docs test (`tests/docs/test_readme_commands.py`)
   extracts fenced `bash` blocks marked `<!-- test -->` from README/USAGE
   and runs them with `OPENKARAOKE_FAKE_ML=1` against fixtures (at least
   quick start and edit loop) — docs cannot drift from the CLI.
6. All EN; JA section reviewed by the maintainer (native check) before
   merge.

## Acceptance Criteria

- [ ] README quick start runs verbatim on a clean checkout (docs test
      green in CI).
- [ ] Legal notice present in EN + JA with all three points.
- [ ] USAGE covers 100% of commands/flags (checklist against `--help`
      output committed in PR description).
- [ ] TROUBLESHOOTING covers every §7 row (cross-reference table included
      in the doc).
- [ ] Demo image is fixture-derived (provenance noted in the image alt).

## Validation

```bash
uv run pytest tests/docs -q
```

## Dependencies

- 24, 26 (CLI complete enough to document truthfully)

## Non-goals

- Website/GitHub Pages; tutorial videos; v2 feature docs.

## Design References

- DESIGN §1, §2, §6, §7, §8 T10
