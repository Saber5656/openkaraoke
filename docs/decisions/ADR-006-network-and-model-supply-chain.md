# ADR-006: Offline-first network policy and pinned model supply chain

- **Status:** Accepted (2026-07-06)

## Context

The project will be public OSS. Trust requires that a media-processing tool
does not silently touch the network, and ML weights are a real supply-chain
attack surface (RoFormer checkpoints are pickle-based `.ckpt`; HF repos are
mutable unless revision-pinned). The product principle is "fully local".

## Decision

1. **Runtime network policy:** the only code paths allowed to open sockets
   are model downloads (`models download`, or first-use auto-download with a
   visible notice). `--offline` guarantees zero network use and is tested via
   a socket-blocking test fixture. No telemetry, no update checks, ever.
2. **Pinned model registry** (`models_cache/registry.py`): every model is
   identified by exact source (HF repo id + filename + revision, or
   audio-separator model filename), expected size, and **sha256**. Downloads
   are verified after fetch and re-verifiable via `models verify`; mismatch ⇒
   refuse to load (exit 6).
3. **Format preference:** HF transformer models loaded with
   `use_safetensors=True` and pinned `revision`. For pickle-based separation
   checkpoints, hash pinning is the compensating control; pursuing
   safetensors alternatives is tracked (unknown U4).
4. **No arbitrary model URLs** via CLI in v1. A local file override
   (`--model-path`) is allowed with an explicit trust warning (documented).
5. Secrets: none stored. `HF_TOKEN` env var is honored read-only for gated
   models, never logged or persisted.

## Alternatives considered

- Trust-on-first-use without hashes — rejected: silent poisoning risk.
- Vendoring weights in releases — rejected: size, licensing, update cadence.

## Consequences

- First run requires an explicit ~2 GiB download step; UX cost accepted and
  smoothed by `models download` + `doctor` guidance.
- Model upgrades are deliberate registry PRs (hash bump), reviewable and
  bisectable.
- Implementation must funnel all downloads through one module (issue 17);
  security tests enforce the socket policy (issue 28).
