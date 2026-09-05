# ADR-001: Python ≥ 3.11 with uv-managed packaging

- **Status:** Accepted (2026-07-06)
- **Deciders:** product owner + Fable (design agent)

## Context

openkaraoke's core value depends on ML components (vocal separation, CTC
forced alignment) whose practical open-source implementations
(`audio-separator`, torch/torchaudio, transformers) live in the Python
ecosystem. The tool is a local CLI intended for `uv tool install` / `pipx`
distribution. Accelerator-specific torch builds require index gymnastics that
uv handles first-class (see research doc §4).

## Decision

- Language/runtime: **Python, `requires-python = ">=3.11,<3.14"`**.
- Packaging/deps/venv: **uv** (`pyproject.toml` + committed `uv.lock`), src
  layout (`src/openkaraoke/`), console script `openkaraoke`.
- CLI framework: **click ≥ 8.1** (ubiquitous, boring, well-known to
  implementation agents). Validation: **pydantic v2**. Console: **rich**.
- Lint/format: **ruff** (format + lint). Types: **mypy**, strict on non-ML
  modules, relaxed on ML backend modules.
- ML deps isolated behind extras so core commands work without them (lazy
  imports; see DESIGN §3.3).

## Alternatives considered

- **TypeScript/Node or Rust core shelling out to Python ML tools** — rejected:
  two-runtime packaging pain, no benefit for a batch pipeline.
- **Poetry/pip-tools** — rejected: uv is the owner's standard and has the
  official PyTorch integration guide.
- **Typer** — rejected in favor of plain click (fewer layers of magic).

## Consequences

- Accelerator extras matrix (cpu/cuda) must be designed in the packaging issue
  (issue 30) per the uv PyTorch guide; macOS uses default wheels (MPS).
- Python 3.10 excluded (audio-separator allows it, but 3.11+ gives better
  errors/perf; narrower test matrix).
