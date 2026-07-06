# Title

Packaging: accelerator extras matrix, uv tool / pipx install

## Summary

Finalize distributable packaging: the `cpu`/`cuda` accelerator extras with
uv index configuration per the Astral PyTorch guide, macOS default-wheel
(MPS) path, verified `uv tool install` / `pipx install` flows from a local
build and TestPyPI, dependency bound tightening, and the documented install
matrix.

## Context

ADR-001 and DESIGN §3.4/§10. Research doc §4: accelerator builds live on
dedicated PyTorch indexes; audio-separator brings its own onnxruntime
extras — the interplay is known-unknown U3 and must be resolved here.

## Scope

- `pyproject.toml` (extras + `[tool.uv]` indexes), `uv.lock`
- `docs/INSTALL.md`
- Local verification scripts under `scripts/check_install.sh`

## Detailed Requirements

1. Extras design (target; adjust only with a recorded reason in the PR):
   - `openkaraoke[ml]` — default ML stack: torch/torchaudio from PyPI
     (macOS/arm64 gets MPS; Linux gets CUDA-bundled default wheels),
     `audio-separator` CPU onnxruntime.
   - `openkaraoke[ml-cpu]` — explicit CPU-only: torch/torchaudio pinned to
     the `+cpu` index via uv `explicit = true` index config;
     onnxruntime CPU.
   - `openkaraoke[ml-cuda]` — CUDA: torch cu-suffixed index +
     `audio-separator[gpu]` (onnxruntime-gpu). Document the CUDA minor
     chosen and why (latest supported by torch at implementation time).
   - Conflict rule: extras are mutually exclusive; add a runtime doctor
     warning when both onnxruntime and onnxruntime-gpu are importable.
2. Resolve **U3**: install each extra into a fresh venv on
   (a) macOS arm64, (b) Linux x86_64 CPU, (c) Linux + NVIDIA (if available;
   else document as untested-blocked and open a follow-up) — record exact
   `uv sync`/`uv tool install` transcripts under the PR; failures get
   follow-up issues.
3. Tighten dependency bounds from the issue-01 initial values to the
   actually-tested versions (torch `>=X.Y,<X.(Y+2)` pattern); re-lock.
4. `uv tool install --from . openkaraoke[ml]` and
   `pipx install .[ml]` both yield a working `openkaraoke doctor` on a
   clean machine (use a container for Linux; document macOS manual run).
5. TestPyPI dry run: build (`uv build`), upload to TestPyPI (manual, by the
   maintainer — agent prepares `scripts/` + docs; no credentials in repo),
   install from TestPyPI into a fresh venv, `doctor` + fake-ML `quick`
   smoke. Record evidence in the PR.
6. `docs/INSTALL.md`: matrix table (OS × accelerator × command), ffmpeg
   prerequisites per OS, font guidance, cache locations, offline install
   notes (`models download` on a networked machine + copying the cache).
7. Wheel hygiene: `uv build` wheel contains no tests/fixtures; data files
   (style template, doctor schema) included — asserted by a script that
   inspects the wheel file list (`scripts/check_wheel_contents.py`, run in
   CI from this issue on).

## Acceptance Criteria

- [ ] All three extras resolve + import correctly per platform matrix
      (transcripts attached); U3 resolved or split into follow-ups.
- [ ] Clean-machine `uv tool install` → `doctor` pass documented for both
      OSes.
- [ ] Wheel content check green in CI.
- [ ] INSTALL.md matrix complete; README points to it.
- [ ] `uv.lock` reflects final bounds; CI still green.

## Validation

```bash
uv build && uv run python scripts/check_wheel_contents.py dist/*.whl
bash scripts/check_install.sh cpu     # per-platform, documented
```

## Dependencies

- 01; 18/19 (real ML deps in place)

## Non-goals

- PyPI production publishing + release automation (32); Windows support
  (v2); conda packaging.

## Design References

- DESIGN §3.4, §10, §12 U3; ADR-001; research §4
