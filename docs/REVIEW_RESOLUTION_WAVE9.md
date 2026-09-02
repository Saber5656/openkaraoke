# Wave 9 concrete review-resolution addendum

- Repository: `Saber5656/openkaraoke`
- Pull request: #33
- Current PR head identity: the authoritative value is supplied by the immutable review manifest; this file intentionally omits the mutable commit SHA to avoid a self-referential identity.
- Current PR base pinned for this review: `47b7fb349a801ea8c475e95f48c0cd13be3bf597`
- This document records documentation-level handling for documentation-only PR findings. It is not a claim that implementation, runtime tests, build, CI, security, or release validation is complete.
- The immutable review manifest pins current head, base, and artifact blob identity; any later change invalidates this evidence and requires a fresh review.
- No PR review bot is triggered or rerun.

## Blocking specialist handoffs

| Role | Required evidence |
|---|---|
| QA/full validation | `tech-qa`/`tech-tester` must execute workspace, timing, LRC, ASS, render, CLI, packaging, docs-lint, and repository-full-validation gates for this head/base. |
| Security/privacy | `tech-security`/`tech-devopssec` must accept model-cache, network, subprocess, path, media, deletion, and release boundaries. |

Missing, pending, failed, skipped, cancelled, timed-out, stale, or non-accepting specialist/full-validation evidence blocks thread resolution and merge.

## Thread contracts

### 1. Thread `PRRT_kwDOTNkGPs6Oni-G` — Install the ML extra in the happy-path command

- File: `docs/DESIGN.md`
- Line: 93
- Finding basis: Existing review finding “Install the ML extra in the happy-path command” at `docs/DESIGN.md:93`.

**Normative resolution**: At `docs/DESIGN.md:93`, The documented primary install command SHALL install openkaraoke[ml] before the first separate or align stage.

**Focused verification gate**: Run the documented quick start in a clean environment and assert the ML extra is installed before separate/align.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 2. Thread `PRRT_kwDOTNkGPs6Oni-J` — Include requested stage params in the staleness key

- File: `docs/issues/23-cli-stage-commands.md`
- Line: 24
- Finding basis: Existing review finding “Include requested stage params in the staleness key” at `docs/issues/23-cli-stage-commands.md:24`.

**Normative resolution**: At `docs/issues/23-cli-stage-commands.md:24`, The staleness key SHALL include a canonical JSON serialization of every requested stage parameter, including model, device, offset, and command flags.

**Focused verification gate**: Complete a stage, rerun with changed model/device/offset, and assert the key changes and the stage runs.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 3. Thread `PRRT_kwDOTNkGPs6Oni-M` — Copy --output even when render is cached

- File: `docs/issues/23-cli-stage-commands.md`
- Line: 46
- Finding basis: Existing review finding “Copy --output even when render is cached” at `docs/issues/23-cli-stage-commands.md:46`.

**Normative resolution**: At `docs/issues/23-cli-stage-commands.md:46`, A cached render SHALL copy its existing verified artifact to --output; a missing cached artifact SHALL make the render stage stale.

**Focused verification gate**: Render once, remove only the output copy, rerun with --output, and assert the cached artifact is copied.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 4. Thread `PRRT_kwDOTNkGPs6Oni-P` — Allow documented absolute image backgrounds

- File: `docs/issues/15-render-command-builder.md`
- Line: 31
- Finding basis: Existing review finding “Allow documented absolute image backgrounds” at `docs/issues/15-render-command-builder.md:31`.

**Normative resolution**: At `docs/issues/15-render-command-builder.md:31`, A validated absolute background.image_path SHALL remain an absolute path in a separate ffmpeg input argument and SHALL not be placed in the filtergraph.

**Focused verification gate**: Build a plan with an absolute image background and assert it remains a separate validated ffmpeg input.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 5. Thread `PRRT_kwDOTNkGPs6Oni-V` — Prefix solid-background hex colors for ffmpeg

- File: `docs/issues/15-render-command-builder.md`
- Line: 37
- Finding basis: Existing review finding “Prefix solid-background hex colors for ffmpeg” at `docs/issues/15-render-command-builder.md:37`.

**Normative resolution**: At `docs/issues/15-render-command-builder.md:37`, Solid background colors SHALL be emitted to ffmpeg as 0x<RRGGBB> values.

**Focused verification gate**: Generate the default solid filter and assert it contains 0x101020.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 6. Thread `PRRT_kwDOTNkGPs6Oni-Z` — Keep line bounds synchronized with units

- File: `docs/issues/10-timing-model-lint.md`
- Line: 29
- Finding basis: Existing review finding “Keep line bounds synchronized with units” at `docs/issues/10-timing-model-lint.md:29`.

**Normative resolution**: At `docs/issues/10-timing-model-lint.md:29`, Timing-line bounds SHALL equal the first unit start and last unit end; lint SHALL fail when either stored bound differs from its units.

**Focused verification gate**: Edit unit times without line bounds and assert lint fails; update bounds and assert the control passes.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 7. Thread `PRRT_kwDOTNkGPs6Oni-c` — Run timing lint before building subtitles

- File: `docs/issues/23-cli-stage-commands.md`
- Line: 43
- Finding basis: Existing review finding “Run timing lint before building subtitles” at `docs/issues/23-cli-stage-commands.md:43`.

**Normative resolution**: At `docs/issues/23-cli-stage-commands.md:43`, The stage runner SHALL execute timing lint before building subtitles and SHALL stop with the documented exit-4 diagnostics when lint has errors.

**Focused verification gate**: Provide invalid timing.json and assert timing lint runs before subtitle planning and returns exit 4.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 8. Thread `PRRT_kwDOTNkGPs6Oni-g` — Add the missing export overwrite flag

- File: `docs/issues/25-cli-timing-inspect.md`
- Line: 34
- Finding basis: Existing review finding “Add the missing export overwrite flag” at `docs/issues/25-cli-timing-inspect.md:34`.

**Normative resolution**: At `docs/issues/25-cli-timing-inspect.md:34`, The timing export command SHALL accept --force; any existing output file SHALL block export until that flag is supplied.

**Focused verification gate**: Export LRC to an existing file without --force and require refusal; repeat with --force and require replacement.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 9. Thread `PRRT_kwDOTNkGPs6Oni-k` — Treat missing stage outputs as stale

- File: `docs/issues/06-workspace-manifest.md`
- Line: 60
- Finding basis: Existing review finding “Treat missing stage outputs as stale” at `docs/issues/06-workspace-manifest.md:60`.

**Normative resolution**: At `docs/issues/06-workspace-manifest.md:60`, A declared stage output that is absent or invalid SHALL mark its producing stage stale and SHALL cause that producer to run before downstream stages.

**Focused verification gate**: Delete a declared producer output and assert its producer is stale and runs before downstream stages.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 10. Thread `PRRT_kwDOTNkGPs6Oni-q` — Keep model loads inside the verified cache path

- File: `docs/issues/19-alignment-emissions.md`
- Line: 38
- Finding basis: Existing review finding “Keep model loads inside the verified cache path” at `docs/issues/19-alignment-emissions.md:38`.

**Normative resolution**: At `docs/issues/19-alignment-emissions.md:38`, Model constructors SHALL receive only the verified path returned by models_cache.ensure with local_files_only enabled; model names SHALL not trigger network access.

**Focused verification gate**: Point model loader at a missing cache path with sockets blocked and assert local failure without network.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 11. Thread `PRRT_kwDOTNkGPs6Oni-t` — Do not rewrite absolute input paths as relative

- File: `docs/issues/07-media-probe.md`
- Line: 42
- Finding basis: Existing review finding “Do not rewrite absolute input paths as relative” at `docs/issues/07-media-probe.md:42`.

**Normative resolution**: At `docs/issues/07-media-probe.md:42`, The path helper SHALL preserve absolute input paths unchanged and SHALL add ./ only to relative paths that begin with a dash.

**Focused verification gate**: Pass absolute /tmp/song.mp3 and expanded ~/Music/song.mp3 and assert both remain absolute in argv.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 12. Thread `PRRT_kwDOTNkGPs6Oni-v` — Keep eval markers compatible with the timing schema

- File: `docs/issues/21-alignment-quality-eval.md`
- Line: 52
- Finding basis: Existing review finding “Keep eval markers compatible with the timing schema” at `docs/issues/21-alignment-quality-eval.md:52`.

**Normative resolution**: At `docs/issues/21-alignment-quality-eval.md:52`, Evaluation annotations SHALL be stored in a strict evaluation.json sidecar keyed by timing-unit ID; timing.json SHALL remain free of an undeclared ref field.

**Focused verification gate**: Load timing.json plus evaluation.json and assert annotations come from the sidecar without schema errors.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 13. Thread `PRRT_kwDOTNkGPs6Oni-1` — Use a supported uv tool install syntax

- File: `docs/issues/30-packaging-install-matrix.md`
- Line: 48
- Finding basis: Existing review finding “Use a supported uv tool install syntax” at `docs/issues/30-packaging-install-matrix.md:48`.

**Normative resolution**: At `docs/issues/30-packaging-install-matrix.md:48`, The packaging check SHALL use the supported local command uv tool install '.[ml]' and SHALL test the resulting executable.

**Focused verification gate**: Run packaging on a clean venv and assert uv tool install .[ml] produces a working command.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 14. Thread `PRRT_kwDOTNkGPs6Oni-3` — Clamp countdown windows for large dot counts

- File: `docs/issues/13-ass-display-program.md`
- Line: 41
- Finding basis: Existing review finding “Clamp countdown windows for large dot counts” at `docs/issues/13-ass-display-program.md:41`.

**Normative resolution**: At `docs/issues/13-ass-display-program.md:41`, Countdown intro events SHALL use show = max(0, line.start_ms - countdown_dots * 1000 - 500).

**Focused verification gate**: Set countdown_dots to 10 with line start 6000 ms and assert every show time is non-negative.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 15. Thread `PRRT_kwDOTNkGPs6OnmAN` — Align the network-policy wording across docs.

- File: `docs/decisions/ADR-006-network-and-model-supply-chain.md`
- Line: 17
- Finding basis: Existing review finding “Align the network-policy wording across docs.” at `docs/decisions/ADR-006-network-and-model-supply-chain.md:17`.

**Normative resolution**: At `docs/decisions/ADR-006-network-and-model-supply-chain.md:17`, Runtime documentation SHALL state that network access is limited to explicit model-cache download or first-use operations with a visible notice; media processing and ordinary CLI stages make no direct network request.

**Focused verification gate**: Run ordinary generate with sockets blocked and model-cache download with network; assert only the model operation contacts network and emits a notice.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 16. Thread `PRRT_kwDOTNkGPs6OnmAX` — Fix the source-video path contract.

- File: `docs/DESIGN.md`
- Line: 63
- Finding basis: Existing review finding “Fix the source-video path contract.” at `docs/DESIGN.md:63`.

**Normative resolution**: At `docs/DESIGN.md:63`, Source-video rendering SHALL use the ingested input/original<ext> path for the original supported extension, not a hard-coded .mp4 name.

**Focused verification gate**: Ingest a webm fixture and assert source-video render references input/original.webm.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 17. Thread `PRRT_kwDOTNkGPs6OnmAc` — Move audio/analysis.wav to the align stage.

- File: `docs/DESIGN.md`
- Line: 173
- Finding basis: Existing review finding “Move audio/analysis.wav to the align stage.” at `docs/DESIGN.md:173`.

**Normative resolution**: At `docs/DESIGN.md:173`, audio/analysis.wav SHALL be produced and owned by the align stage; ingest SHALL not list it as an output.

**Focused verification gate**: Run ingest and align and assert analysis.wav is absent after ingest and present after align.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 18. Thread `PRRT_kwDOTNkGPs6OnmAi` — Add 05-error-framework to issue 04's prerequisites.

- File: `docs/ISSUE_PLAN.md`
- Line: 34
- Finding basis: Existing review finding “Add 05-error-framework to issue 04's prerequisites.” at `docs/ISSUE_PLAN.md:34`.

**Normative resolution**: At `docs/ISSUE_PLAN.md:34`, The authoritative dependency graph SHALL include 05-error-framework -> 04-console-logging.

**Focused verification gate**: Generate dependency graph and assert 05 precedes 04.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 19. Thread `PRRT_kwDOTNkGPs6OnmA2` — Align the coverage scope

- File: `docs/issues/02-ci-quality-gates.md`
- Line: 39
- Finding basis: Existing review finding “Align the coverage scope” at `docs/issues/02-ci-quality-gates.md:39`.

**Normative resolution**: At `docs/issues/02-ci-quality-gates.md:39`, Coverage SHALL be configured from one source list containing lyrics, timing, subtitles, render.command, workspace, config, and models_cache.registry; CI SHALL invoke pytest-cov without a conflicting broad scope.

**Focused verification gate**: Run coverage and inspect measured source; assert exactly the seven documented packages and no broad conflicting scope.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 20. Thread `PRRT_kwDOTNkGPs6OnmA_` — Add the FFmpeg smoke test to scope or drop it from acceptance.

- File: `docs/issues/02-ci-quality-gates.md`
- Line: 61
- Finding basis: Existing review finding “Add the FFmpeg smoke test to scope or drop it from acceptance.” at `docs/issues/02-ci-quality-gates.md:61`.

**Normative resolution**: At `docs/issues/02-ci-quality-gates.md:61`, Issue 02 SHALL include tests/test_ci_ffmpeg.py in its scope and SHALL wire its marked smoke test into the CI acceptance gate.

**Focused verification gate**: Check issue-02 scope/CI and assert tests/test_ci_ffmpeg.py is present and its marker executes.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 21. Thread `PRRT_kwDOTNkGPs6OnmBD` — Use one canonical exception name.

- File: `docs/issues/05-error-framework.md`
- Line: 35
- Finding basis: Existing review finding “Use one canonical exception name.” at `docs/issues/05-error-framework.md:35`.

**Normative resolution**: At `docs/issues/05-error-framework.md:35`, The public exception table SHALL use EnvMissingError for exit code 5 in every issue and design reference.

**Focused verification gate**: Raise missing environment and assert EnvMissingError with exit 5.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 22. Thread `PRRT_kwDOTNkGPs6OnmBR` — Add the missing force parameter.

- File: `docs/issues/06-workspace-manifest.md`
- Line: 32
- Finding basis: Existing review finding “Add the missing force parameter.” at `docs/issues/06-workspace-manifest.md:32`.

**Normative resolution**: At `docs/issues/06-workspace-manifest.md:32`, create_workspace SHALL have signature create_workspace(dir: Path, *, force: bool = False) and SHALL refuse a non-empty directory unless force=True.

**Focused verification gate**: Call create_workspace on non-empty directory with force false/true; assert refusal then layout verification.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 23. Thread `PRRT_kwDOTNkGPs6OnmBY` — Make the timeout kill path explicit.

- File: `docs/issues/07-media-probe.md`
- Line: 35
- Finding basis: Existing review finding “Make the timeout kill path explicit.” at `docs/issues/07-media-probe.md:35`.

**Normative resolution**: At `docs/issues/07-media-probe.md:35`, The media probe SHALL create a new process group/session, kill the complete group on timeout, and on Windows SHALL use a new process group plus tree termination.

**Focused verification gate**: Timeout an ffmpeg descendant and assert the process group is gone on POSIX and Windows cleanup is specified/tested.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 24. Thread `PRRT_kwDOTNkGPs6OnmBc` — Tighten the JA segmentation contract.

- File: `docs/issues/09-lyrics-parser-segmentation.md`
- Line: 45
- Finding basis: Existing review finding “Tighten the JA segmentation contract.” at `docs/issues/09-lyrics-parser-segmentation.md:45`.

**Normative resolution**: At `docs/issues/09-lyrics-parser-segmentation.md:45`, Japanese segmentation SHALL operate on Unicode grapheme clusters, preserving combining marks and ZWJ sequences as one unit.

**Focused verification gate**: Segment text with combining marks and ZWJ emoji and assert each grapheme cluster remains one unit.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 25. Thread `PRRT_kwDOTNkGPs6OnmBi` — Define the missing lint outcomes.

- File: `docs/issues/10-timing-model-lint.md`
- Line: 50
- Finding basis: Existing review finding “Define the missing lint outcomes.” at `docs/issues/10-timing-model-lint.md:50`.

**Normative resolution**: At `docs/issues/10-timing-model-lint.md:50`, Timing lint SHALL define unit-invalid as duration below 30 ms, unit-short as duration below 60 ms, and line-overlap-mid as overlap greater than 500 ms and at most 2000 ms.

**Focused verification gate**: Run fixtures for 20 ms, 50 ms, and overlap 600/2500 ms; assert exact diagnostic code and range.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 26. Thread `PRRT_kwDOTNkGPs6OnmBt` — Make from_lrc return one contract.

- File: `docs/issues/11-lrc-import-export.md`
- Line: 45
- Finding basis: Existing review finding “Make from_lrc return one contract.” at `docs/issues/11-lrc-import-export.md:45`.

**Normative resolution**: At `docs/issues/11-lrc-import-export.md:45`, from_lrc SHALL return tuple[TimingDocument, list[Diagnostic]] in its signature, prose, and callers.

**Focused verification gate**: Import enhanced and line-only LRC and assert tuple document plus diagnostics at every call site.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 27. Thread `PRRT_kwDOTNkGPs6OnmB1` — Bound countdown_dots or clamp the intro show time.

- File: `docs/issues/13-ass-display-program.md`
- Line: 44
- Finding basis: Existing review finding “Bound countdown_dots or clamp the intro show time.” at `docs/issues/13-ass-display-program.md:44`.

**Normative resolution**: At `docs/issues/13-ass-display-program.md:44`, Countdown intro show SHALL be clamped to zero after subtracting configured dots and intro gap.

**Focused verification gate**: Set large countdown_dots and assert intro show clamps at zero.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 28. Thread `PRRT_kwDOTNkGPs6OnmB7` — Source-video mode needs a duration guard

- File: `docs/issues/15-render-command-builder.md`
- Line: 48
- Finding basis: Existing review finding “Source-video mode needs a duration guard” at `docs/issues/15-render-command-builder.md:48`.

**Normative resolution**: At `docs/issues/15-render-command-builder.md:48`, Source-video ffmpeg input SHALL use -stream_loop -1 and -t plan.duration_ms, so a short source cannot truncate the planned output.

**Focused verification gate**: Render source video shorter than plan.duration_ms and assert output reaches planned duration.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 29. Thread `PRRT_kwDOTNkGPs6OnmCB` — Add coverage controls before using partial annotations for gating.

- File: `docs/issues/21-alignment-quality-eval.md`
- Line: 52
- Finding basis: Existing review finding “Add coverage controls before using partial annotations for gating.” at `docs/issues/21-alignment-quality-eval.md:52`.

**Normative resolution**: At `docs/issues/21-alignment-quality-eval.md:52`, The alignment evaluation gate SHALL require at least 80% of units annotated per song and SHALL report coverage beside median and p90 metrics.

**Focused verification gate**: Run evaluation with 79% and 80% annotated units; assert only 80% meets gate and coverage is reported.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 30. Thread `PRRT_kwDOTNkGPs6OnmCH` — Tighten the workspace deletion guard.

- File: `docs/issues/24-cli-generate-quick.md`
- Line: 43
- Finding basis: Existing review finding “Tighten the workspace deletion guard.” at `docs/issues/24-cli-generate-quick.md:43`.

**Normative resolution**: At `docs/issues/24-cli-generate-quick.md:43`, --no-keep SHALL delete only after the resolved path equals the canonical workspace root recorded in project.json and the required workspace layout passes schema validation.

**Focused verification gate**: Point --no-keep at a decoy project.json and assert deletion is refused; use canonical workspace as control.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 31. Thread `PRRT_kwDOTNkGPs6OnmCN` — Require explicit overwrite protection for any existing timing file.

- File: `docs/issues/25-cli-timing-inspect.md`
- Line: 41
- Finding basis: Existing review finding “Require explicit overwrite protection for any existing timing file.” at `docs/issues/25-cli-timing-inspect.md:41`.

**Normative resolution**: At `docs/issues/25-cli-timing-inspect.md:41`, Timing import SHALL require --force for every pre-existing timing file, regardless of its metadata.

**Focused verification gate**: Import over any existing timing file without --force and require refusal, then allow with --force.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 32. Thread `PRRT_kwDOTNkGPs6OnmCT` — Fail the CJK font check instead of warning.

- File: `docs/issues/26-cli-models-doctor.md`
- Line: 45
- Finding basis: Existing review finding “Fail the CJK font check instead of warning.” at `docs/issues/26-cli-models-doctor.md:45`.

**Normative resolution**: At `docs/issues/26-cli-models-doctor.md:45`, The doctor command SHALL treat a missing configured or default CJK font as a failing prerequisite and SHALL return a non-zero result.

**Focused verification gate**: Run doctor with the configured/default CJK font absent and assert non-zero; install the font and assert the prerequisite passes.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

## Merge boundary

- `gate-task-evaluator` must re-fetch PR state, current head/base, required-check inventory, review decision, unresolved thread state, policy version, and merge candidate immediately before any merge mutation.
- `github_mergeable` and a successful CodeRabbit status are not merge authorization.
- This task permits at most one PR Bot review. This artifact authorizes no Bot trigger or rerun.
