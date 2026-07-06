# ADR-005: Per-song workspace directory + manifest as the pipeline contract

- **Status:** Accepted (2026-07-06)

## Context

Automatic lyric timing will be imperfect; the product's usability hinges on a
cheap, deterministic **edit → re-render loop**. Stages have heavy, cacheable
outputs (stems ≈ minutes of compute). Users need to inspect intermediates,
and implementation agents need crisp file-level contracts between issues.

## Decision

- Every song is processed inside a **self-contained workspace directory**
  with fixed, ASCII-only file names (DESIGN §4.1); the input media is copied
  in at `new` time.
- `project.json` is the machine-owned **manifest**: per-stage status, params,
  tool versions, and an `inputs_hash` over each stage's declared inputs.
  Staleness is *computed at load* by re-hashing; hand-editing `timing.json`
  or `style.toml` therefore transparently invalidates downstream stages —
  this is the edit-loop mechanism, not a special case.
- `timing.json` is simultaneously a machine artifact and the primary
  **user-editable document** (with `timing lint` as its validator and
  enhanced-LRC import/export as an alternative editing path).
- All writes are atomic (temp + `os.replace`) and confined to the workspace,
  the model cache, or an explicit `--output` path.

## Alternatives considered

- **Single-shot temp-dir pipeline (no persistent workspace)** — rejected:
  kills the edit loop and re-render cheapness.
- **SQLite state / hidden dot-dirs** — rejected: opaque to users and to
  low-capability implementation agents; files-with-schemas are the contract.
- **Make/DVC-style external DAG tools** — rejected: dependency weight;
  our DAG is a fixed 5-stage line.

## Consequences

- Hashing discipline is mandatory in every stage implementation (shared
  helpers in `workspace/hashing.py`).
- Workspaces are large (copied input + stems); documented, with `quick
  --no-keep` for throwaway use.
- Concurrent runs on the same workspace are unsupported in v1 (single-user
  CLI); a simple lockfile guard prevents corruption (issue 06).
