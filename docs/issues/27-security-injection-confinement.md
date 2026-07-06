# Title

Security hardening: text injection, path confinement, subprocess policy tests

## Summary

Consolidate and prove the input-side security properties of DESIGN §8
(T1, T2, T3, T6): a dedicated adversarial test suite plus any gap-fixes it
uncovers in the already-landed modules (06, 07, 09, 12, 15).

## Context

Individual issues implement local checks; this issue attacks them as an
adversary and pins the properties with regression tests so future changes
cannot silently weaken them. Findings that require design changes are filed
as new issues, not fixed ad hoc.

## Scope

- `tests/security/` suite (new)
- Small fixes in existing modules where tests expose gaps (same-PR allowed
  when local and behavior-preserving)

## Detailed Requirements

1. **T3 — lyrics → ASS injection corpus** (`tests/security/test_ass_injection.py`):
   table of hostile lyric lines, each asserted to appear only as escaped
   literal text in the built ASS and to render (ffmpeg burn over 2 s black,
   `@ffmpeg`) with exit 0:
   - `{\an8}TOP` , `{\pos(0,0)}x`, `{\k100}`, `}{`, `\N\n\h`, `\clip(...)`,
     10 KiB single line (must be rejected earlier by the 200-unit cap —
     assert InputError instead), zero-width joiners, RTL override chars
     (U+202E — must be stripped as control-adjacent: extend escaping to
     strip Cf-category bidi controls `U+202A..U+202E`, `U+2066..U+2069`;
     implement in 12's `escape_display` if missing).
2. **T6 — confinement fuzz** (`test_confinement.py`): generated candidates
   (`../x`, `..%2Fx`, absolute, `~`, symlink dirs, symlink workspace root,
   NUL byte, 4096-char path) against `confine`/workspace loaders; plus
   `render --output` and `timing export-lrc -o` allowed-escape paths
   documented and asserted as the *only* outward writes (grep+runtime
   audit: monkeypatch `open`/`os.replace` recorders around a full fake-ML
   `quick` run and diff the write set against the workspace allowlist).
3. **T2 — subprocess policy** (`test_subprocess_policy.py`):
   - Static: AST scan over `src/` asserting `subprocess` usage only in
     `media/ffmpeg.py`, no `shell=True` anywhere, no `os.system`.
   - Dynamic: filenames `-i.wav`, `--; rm -rf.wav`, `$(x).wav`, newline in
     name — create real files with these names in tmp dirs, run probe/
     extract, assert they are treated as data (ffmpeg receives them as
     single argv items — capture via a recording wrapper around `run`).
4. **T1 — probe robustness**: corpus of malformed containers (truncated
   mp4, wav with absurd header sizes, zip renamed .mp3 — generated in
   fixtures script) ⇒ every case ends in `InputError`/`StageError`, never
   a traceback (exit 10) or hang (10 s watchdog).
5. Each property gets a one-line comment tag `# SEC:T<N>` so coverage of
   security properties is greppable; a meta-test asserts all of
   T1/T2/T3/T6 tags exist.
6. Document any accepted residual risks found during the work by appending
   to DESIGN §8.2 (PR updates the table's mitigation column with test
   references).

## Acceptance Criteria

- [ ] Full suite green on Linux + macOS CI, including `@ffmpeg` legs.
- [ ] Bidi/Cf stripping implemented and golden-tested (if it was missing).
- [ ] Runtime write-audit shows zero out-of-allowlist writes in the E2E
      fake run.
- [ ] AST policy test guards `shell=True` regressions.
- [ ] DESIGN §8.2 table updated with test references (docs change in the
      same PR).

## Validation

```bash
uv run pytest tests/security -q
```

## Dependencies

- 06, 07, 12 (attack surfaces exist); 22/23 recommended landed for the
  write-audit (else audit the stage functions directly)

## Non-goals

- Supply-chain/network properties (28); fuzzing ffmpeg itself; sandboxing
  subprocesses (documented out of scope in DESIGN §8 T1).

## Design References

- DESIGN §8.2 T1/T2/T3/T6, §5.4 escaping, §4.1 fixed names
