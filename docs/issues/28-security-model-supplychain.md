# Title

Security hardening: download verification, offline guarantee, SECURITY.md

## Summary

Prove the network-side security properties (DESIGN §8 T4/T5): end-to-end
model download verification tests, the offline zero-socket guarantee across
the whole CLI, plus authoring `SECURITY.md` (threat model summary, reporting
policy, pickle-risk disclosure).

## Context

Complements issue 27 on the supply-chain side of the threat table; ADR-006
is the normative policy. This is also where the project's public security
documentation is written for OSS release.

## Scope

- `tests/security/test_supplychain.py`, `test_offline.py`
- `SECURITY.md` (repo root)
- `NOTICE` file (attributions: UVR/audio-separator, whisperX technique,
  model licenses)

## Detailed Requirements

1. **T4 tests** (against issue 17's fixture server):
   - Tampered payload (correct size, wrong bytes) ⇒ `ModelError`, file kept
     under a quarantine name `<file>.quarantine`, clear message (adjust 17
     implementation if it deviates — spec here is normative: quarantine
     rename, no deletion).
   - Redirect to a different host is refused for registry entries whose
     source pins a host (download layer must set
     `allow_redirects` policy accordingly — verify with the fixture
     server issuing 302 to a second local server).
   - `models verify` detects post-download bit flips (re-used from 17,
     asserted here as a security property with `# SEC:T4` tags).
2. **T5 offline guarantee** (`test_offline.py`): socket-blocking fixture
   (monkeypatched `socket.socket` raising) around **every** CLI command
   with `--offline` on a fully-cached fake workspace: `new, generate,
   separate, align, subtitle, render, timing *, inspect, models list,
   models verify, doctor` — all must complete or fail with exit ≤ 8
   without attempting a socket. (`models download` asserted to refuse
   before socket creation.) Add `# SEC:T5` tags.
3. **SECURITY.md** contents (English):
   - Supported versions table (0.x latest only).
   - Private vulnerability reporting: GitHub Security Advisories
     (`Report a vulnerability`), 90-day coordinated disclosure default.
   - Threat-model summary linking DESIGN §8; explicit statements:
     media parsing delegated to system ffmpeg (keep it updated; not
     sandboxed by us), separation checkpoints are pickle-format —
     mitigations (pinning + sha256) and residual risk, no telemetry /
     network policy, `HF_TOKEN` handling.
   - Scope exclusions: outputs' copyright status is the user's
     responsibility (link README legal notice).
4. **NOTICE**: attribution lines required by ADR-002 (UVR + audio-separator
   credit), whisperX BSD-2 technique attribution, model source list with
   licenses as recorded in the registry — generated content must match
   `registry.py` (unit test compares).
5. Any 17-behavior adjustments land here with their own tests (quarantine
   naming, redirect policy) — keep diffs minimal and within
   `models_cache/`.

## Acceptance Criteria

- [ ] Tamper/redirect/bit-flip matrix green; quarantine behavior exact.
- [ ] Offline sweep covers every registered CLI command (parametrized over
      the click group's command list so new commands are auto-included —
      failing loudly if a future command forgets offline handling).
- [ ] SECURITY.md + NOTICE reviewed against DESIGN §8 (checklist in PR).
- [ ] NOTICE↔registry consistency test green.

## Validation

```bash
uv run pytest tests/security/test_supplychain.py tests/security/test_offline.py -q
```

## Dependencies

- 17 (store under test); most CLI commands landed (22–26) for the sweep

## Non-goals

- GitHub org/repo settings, release signing (32); converting models to
  safetensors (U4 follow-up).

## Design References

- DESIGN §8 T4/T5, §8.3; ADR-006; ADR-002 (attribution duty)
