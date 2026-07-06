# Title

Workspace layout, manifest, staleness state machine

## Summary

Implement `openkaraoke.workspace`: creation/validation of the per-song
workspace directory, the `project.json` manifest models, sha256 hashing
helpers, computed staleness, atomic writes, path-confinement guard, and a
single-process lockfile.

## Context

This is the pipeline contract at the heart of ADR-005. Stage code (issues
18–24) only interacts with workspaces through this module.

## Scope

- `src/openkaraoke/workspace/layout.py`, `manifest.py`, `hashing.py`
- Unit tests

## Detailed Requirements

1. `layout.py`:
   - Constants for every workspace-relative path in DESIGN §4.1
     (e.g. `SOURCE_WAV = Path("audio/source.wav")`).
   - `slugify(name: str) -> str`: lowercase, NFKD → ASCII-fold, non
     `[a-z0-9-]` → `-`, collapse repeats, trim `-`, clamp to 64 chars;
     result must match `^[a-z0-9][a-z0-9-]{0,63}$` else `InputError` with
     remedy "rename the project directory".
   - `create_workspace(dir: Path) -> None`: create the fixed subdirectories;
     refuse non-empty dir unless `force=True` (then only verify layout).
   - `confine(workspace: Path, candidate: Path) -> Path`: resolve both
     (`Path.resolve(strict=False)`) and require the candidate to be inside
     the workspace root; raise `WorkspaceError` otherwise. **Every write in
     later issues goes through `confine` (or the cache equivalent in 17).**
   - `check_no_symlink_dirs(workspace)`: any of the fixed subdirs being a
     symlink ⇒ `WorkspaceError` (DESIGN §8 T6).
   - `acquire_lock(workspace)` context manager: `O_CREAT|O_EXCL` lockfile
     `.lock` containing pid; stale lock (pid dead) is replaced with a
     warning; busy lock ⇒ `WorkspaceError` "another openkaraoke process…".
2. `hashing.py`: `sha256_file(path) -> str` (1 MiB chunks, lowercase hex);
   `inputs_hash(files: dict[str, str], params: dict) -> str` implementing
   DESIGN §4.6 exactly (`json.dumps(..., sort_keys=True, separators=(",", ":"))`,
   wrapper object `{"files": …, "params": …, "schema": 1}`).
3. `manifest.py` (pydantic, `extra="forbid"`):
   - Models: `Manifest`, `InputInfo`, `StageRecord` per DESIGN §4.2 with
     `status: Literal["pending","done","failed"]`.
   - `Manifest.new(...)` factory used by `new` (issue 22).
   - `load_manifest(workspace) -> Manifest`: missing file / bad JSON /
     wrong `schema_version` / validation error ⇒ `WorkspaceError` with the
     DESIGN §7 remedy text.
   - `save_manifest(workspace, m)`: atomic (same-dir `os.replace`),
     `indent=2`, `ensure_ascii=False`, trailing newline.
   - `STAGE_ORDER = ["separate", "align", "subtitle", "render"]`.
   - `stage_inputs(workspace, m, stage) -> dict[str, str]`: returns the
     stage's declared input files (DESIGN §3.2 table) with their current
     hashes; missing input file ⇒ empty-string hash (yields stale).
   - `is_stale(workspace, m, stage) -> bool`: status != done, or recomputed
     `inputs_hash` differs from `StageRecord.inputs_hash`.
   - `stages_to_run(workspace, m, until: str | None, force: bool) -> list[str]`:
     pipeline order; a stale/forced stage implies all later stages.
4. All functions typed, no ML imports, no network.

## Acceptance Criteria

- [ ] Round-trip: `Manifest.new` → save → load equals (model equality).
- [ ] Hand-editing a declared input (e.g. rewrite `lyrics/timing.json`)
      flips `is_stale("subtitle")` to True without any manifest change.
- [ ] `stages_to_run` returns `["align","subtitle","render"]` when only
      align is stale; `--force` returns all four; `until="align"` truncates.
- [ ] `confine` rejects `../escape`, absolute outside paths, and
      symlink-out subdirs (tests create a symlinked `audio/`).
- [ ] Lock prevents a second concurrent open; stale lock is recovered.
- [ ] Atomicity: killing mid-save (simulated by patching `os.replace` to
      raise) never leaves a corrupt `project.json`.
- [ ] mypy strict; module coverage ≥ 90%.

## Validation

```bash
uv run pytest tests/workspace -q
```

## Dependencies

- 03 (types/paths), 05 (errors)

## Non-goals

- Actually running stages (23/24); ingest population of the manifest (22).

## Design References

- DESIGN §4.1, §4.2, §4.6, §3.2, §8 T6
- ADR-005
