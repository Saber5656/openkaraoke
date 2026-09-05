# Title

CLI `new`: project creation and ingest

## Summary

Implement `openkaraoke new` per DESIGN §5.1: workspace creation, media
probe + validation, input copy, `source.wav` extraction, lyrics validation +
copy, style template, and manifest initialization.

## Context

First real user-facing command; wires together issues 06–09 and 14. Ingest
is deliberately synchronous and re-runnable (`--force`).

## Scope

- `src/openkaraoke/cli/new.py`
- CLI tests (CliRunner + `@ffmpeg` fixtures)

## Detailed Requirements

1. Signature: `openkaraoke new DIR --input FILE --lyrics FILE
   [--language ja|en] [--audio-stream N] [--force]`.
2. Order of operations (fail fast, no partial workspace on validation
   errors — validate *before* creating anything):
   a. Load app config; language default `cfg.default_language`.
   b. Probe input (07), `validate_input`, classify.
   c. Parse lyrics (09) with the chosen language (validation only).
   d. `slugify(DIR.name)` (06).
   e. Create workspace (06; `--force` allows existing per its semantics),
      acquire lock.
   f. Copy input → `input/original.<ext-lowercased>` (extension from the
      original filename, restricted to the DESIGN §10 whitelist; content
      hash recorded). Copy is streamed with progress for files > 50 MiB.
   g. Extract `audio/source.wav` (08).
   h. Copy lyrics → `lyrics/lyrics.txt` (byte-verbatim).
   i. `write_style_template` (14) — skipped with a notice if present and
      `--force`.
   j. Build + save manifest: input info, language, all stages `pending`.
3. Console output (default verbosity): one `•` line per step, final block
   `project ready: <dir>` + next-step hint (`openkaraoke generate <dir>`).
4. Failure behavior: any error after step e removes the partially created
   workspace **only if** it was created by this invocation and `--force`
   was not used (track a `created_here` flag); with `--force`, leave files
   and report.
5. Errors use the issue-05 taxonomy: probe/validation ⇒ 4; ffmpeg missing ⇒
   5; workspace conflicts ⇒ 8.
6. Tests: happy audio path (tone-5s.wav + 3-line ja lyrics fixture);
   happy video path (colorbar-2s.mp4 → media_type video, style auto ⇒
   source_video note); duration-cap rejection (config override to 0 min in
   a temp config file); bad lyrics rejection leaves no directory;
   `--force` re-ingest updates hashes and resets stages to pending +
   marks all stale; `--audio-stream` out of range ⇒ exit 4.

## Acceptance Criteria

- [ ] Full workspace tree matches DESIGN §4.1 after happy path; manifest
      validates; `inspect`-level load succeeds (via `load_manifest`).
- [ ] Partial-failure cleanup and `--force` semantics per req 4 proven by
      tests.
- [ ] Exit codes per req 5 asserted.
- [ ] mypy strict clean.

## Validation

```bash
uv run pytest tests/cli/test_new.py -q
```

## Dependencies

- 06, 07, 08, 09, 14

## Non-goals

- Running any pipeline stage (23/24); `quick` (24).

## Design References

- DESIGN §5.1, §6 (new row), §4.1/§4.2
