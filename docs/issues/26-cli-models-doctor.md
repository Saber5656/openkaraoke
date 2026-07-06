# Title

CLI `models` and `doctor`

## Summary

Implement `openkaraoke models list|download|verify` over issue 17's store,
and `openkaraoke doctor` — the environment diagnostic covering ffmpeg/libass,
fonts, devices, model cache, config, and disk space, with `--json` output
and per-OS remediation guidance.

## Context

DESIGN §6.1 fixes the doctor check table; ADR-006 fixes the models UX.
Doctor is the designated first-run experience (§2.1).

## Scope

- `src/openkaraoke/cli/models.py`, `doctor.py`
- `src/openkaraoke/doctor/checks.py` (check implementations)
- CLI tests with mocked environments

## Detailed Requirements

1. `models list [--json]`: table from `list_models()` (key, kind, cached,
   size, hash-ok). `models download [KEY…]`: no args ⇒ separation default +
   alignment models for `ja` and `en`; honors `--offline` by refusing
   immediately (exit 6). `models verify [--json]`: re-hash all cached
   entries; exit 6 if any mismatch, listing files (never auto-delete —
   ADR-006).
2. `doctor [--json]` runs the DESIGN §6.1 checks, each returning
   `CheckResult{id, status: pass|warn|fail, detail, remedy | None}`:
   - `ffmpeg`, `ffprobe`: found + version ≥ 6 (07 helpers).
   - `libass`: `has_subtitles_filter`.
   - `python`: interpreter + package version.
   - `ml-extras`: import probe (warn-only when missing).
   - `device`: report available accelerators (torch probe guarded; also
     onnxruntime providers when importable).
   - `cjk-font`: platform strategy — Linux: `fc-list :lang=ja family`
     non-empty (subprocess via the 07 runner; `fc-list` missing ⇒ warn);
     macOS: any of `/System/Library/Fonts/ヒラギノ角ゴシック W3.ttc`,
     `Hiragino Sans` via `system_profiler`-free static path list, or Noto
     paths under `/Library/Fonts`|`~/Library/Fonts`; plus: if
     `style.font.family` default font unresolvable ⇒ warn with Noto install
     remedy. (Exact path list in code; keep it a data constant for U6.)
   - `models`: registry entries cached + verified (warn when absent).
   - `disk`: ≥ 2 GiB free at cache root (shutil.disk_usage).
   - `config`: `load_app_config` succeeds (report path used).
3. Output: aligned table (id, status glyph ✓/⚠/✖, detail); failures append
   remedy lines. `--json`: list of CheckResult. Exit 0 when no `fail`
   (warns allowed), 5 otherwise (DESIGN §6 doctor row).
4. Tests: each check unit-tested with monkeypatched probes (missing ffmpeg,
   old ffmpeg, no libass, no fonts, no ml extras, low disk); exit-code
   matrix; `--json` schema; `models` flows against the issue-17 fixture
   server (download/list/verify + offline refusal).

## Acceptance Criteria

- [ ] Doctor table matches DESIGN §6.1 row-for-row (ids stable).
- [ ] Every fail path carries an actionable remedy string.
- [ ] `models verify` catches a corrupted cache file (integration with 17
      fixtures).
- [ ] `--json` outputs validate against a checked-in JSON Schema file
      (`docs/schemas/doctor.schema.json` — add it here).
- [ ] mypy strict clean (doctor module).

## Validation

```bash
uv run pytest tests/cli/test_models_doctor.py -q
uv run openkaraoke doctor        # manual smoke on dev machine
```

## Dependencies

- 07, 17

## Non-goals

- Auto-installing anything; font downloading (v2 idea); GPU benchmarking.

## Design References

- DESIGN §6.1, §2.1, §12 U6; ADR-006
