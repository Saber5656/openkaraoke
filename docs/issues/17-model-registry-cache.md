# Title

Model registry, cache store, download + hash verify

## Summary

Implement `openkaraoke.models_cache`: the pinned model registry (separation
model + per-language alignment models), the cache directory layout, download
with sha256 verification, `verify`/`list` logic, and offline-mode behavior.
This module is the **only** network-capable code in the product (ADR-006).

## Context

DESIGN §8 T4/T5 and ADR-006 define the supply-chain rules. Downstream
backends (18/19) request models exclusively through this module.

## Scope

- `src/openkaraoke/models_cache/registry.py`, `store.py`
- Unit tests with a local HTTP fixture server (pytest `http.server` thread)

## Detailed Requirements

1. `registry.py` — dataclass entries, all constants in code:

   | key | kind | source | file | sha256 |
   |---|---|---|---|---|
   | `separation/bs_roformer_ep317` | audio-separator model | audio-separator download mechanism | `model_bs_roformer_ep_317_sdr_12.9755.ckpt` | `TBD-at-implementation` |
   | `align/ja` | HF transformers | repo `jonatasgrosman/wav2vec2-large-xlsr-53-japanese` | pinned `revision` (commit sha, resolved at implementation) | per-file via HF etag + recorded snapshot hash |
   | `align/en` | torchaudio bundle | `WAV2VEC2_ASR_BASE_960H` | torchaudio-managed | recorded url sha256 |

   Implementation task includes **resolving and hard-coding** the actual
   sha256/revision values (download once, hash, commit the constants; record
   the values in the PR description as evidence).
2. `store.py`:
   - Layout: `<cache_root>/models/<key>/…`; HF snapshots under
     `<cache_root>/hf/` via `HF_HOME` env (set process-locally during
     download only).
   - `ensure(key, *, offline: bool, console) -> Path`:
     present+verified ⇒ path; present+hash-mismatch ⇒ `ModelError`
     (remedy: `models verify` / re-download instructions; never auto-delete);
     absent+offline ⇒ `ModelError` exit-6 with `models download` hint;
     absent+online ⇒ download with a visible notice line
     (`downloading <key> (<size>)…`), verify sha256, atomic move into place.
   - Downloads: `huggingface_hub.snapshot_download(repo, revision=…)` for HF;
     for the audio-separator model, delegate to the library's own download
     by pointing its `model_file_dir` at our layout, then hash-verify the
     resulting file ourselves.
   - `verify_all() -> list[VerifyResult]`; `list_models() -> table rows`
     (key, kind, cached?, size, hash-ok?).
   - Socket hygiene: module exposes `assert_offline_safe()` used by tests —
     under `--offline`, `ensure` must return/raise **before** any network
     import/socket call; tested with a socket-blocking fixture
     (monkeypatch `socket.socket` to raise).
3. Registry keys for alignment map from language via
   `alignment_model_key(language) -> str` (`ja`→`align/ja`, `en`→`align/en`,
   else `ModelError` "language not supported in v1").
4. No arbitrary URLs: public API accepts registry keys only. A local
   override path (`--model-path`, wired in 23/26) bypasses download but
   still logs a trust warning line.
5. mypy strict for registry; store may relax for hub types.

## Acceptance Criteria

- [ ] Fixture-server download verifies hash, retries once on truncation
      (HTTP 500 / short read), and lands atomically (no partial file visible
      under the final name at any point — poll from a thread in the test).
- [ ] Hash mismatch refuses load and does not delete the file.
- [ ] Offline behavior per matrix (present/absent × offline) exactly as
      req 2; socket-block test proves zero network under offline.
- [ ] `verify_all` flags a bit-flipped cached file.
- [ ] TODO-marker test: registry contains no `TBD-at-implementation` strings
      at issue completion (constants resolved).

## Validation

```bash
uv run pytest tests/models_cache -q
```

## Dependencies

- 03 (cache_root), 05

## Non-goals

- Actual model inference (18/19); CLI surfaces (26); safetensors migration
  for the separation ckpt (tracked as U4).

## Design References

- DESIGN §8 T4/T5, §12 U4; ADR-006
