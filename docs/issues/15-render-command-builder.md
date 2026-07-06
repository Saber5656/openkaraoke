# Title

Render command builder (backgrounds, filtergraph)

## Summary

Implement `openkaraoke.render.command`: pure construction of the ffmpeg argv
for the render stage for all three background modes (solid / image /
source_video), including scaling/darken/blur filters, subtitle burn-in, audio
gain, and encoding parameters — no process execution.

## Context

DESIGN §5.5 fixes the command shapes and the security rule: argv lists only,
`cwd = workspace`, fixed relative paths inside the filtergraph (no
user-controlled strings in filters — T2).

## Scope

- `src/openkaraoke/render/command.py`
- Unit tests (string/argv assertions only)

## Detailed Requirements

1. `build_render_argv(plan: RenderPlan) -> list[str]` where
   `RenderPlan{mode, style: StyleConfig, duration_ms: int,
   instrumental: relpath, ass: relpath = "subtitles/karaoke.ass",
   background: relpath | color, input_media: relpath | None,
   out_tmp: relpath, ffmpeg: Path}` — all paths **workspace-relative**
   (validated: `not p.is_absolute() and ".." not in p.parts`, else
   `WorkspaceError`).
2. Common tail: `-c:v libx264 -preset <preset> -crf <crf> -pix_fmt yuv420p
   -c:a aac -b:a 192k -ar 44100 -movflags +faststart -progress pipe:1
   -nostats -y <out_tmp>`; `-r <fps>` on the video input side per mode.
   Audio filter `-filter:a volume=<gain>dB` only when `gain_db != 0`
   (formatted `%.1f`).
3. Mode `solid`: inputs `-f lavfi -i color=c=<RRGGBB hex without #>:s=<WxH>:r=<fps>:d=<dur_s>`
   then `-i <instrumental>`; `-shortest`; vf chain:
   `subtitles=<ass>` + `,format=yuv420p`.
4. Mode `image`: `-loop 1 -framerate <fps> -i <background>` + instrumental;
   vf chain: `scale=<W>:<H>:force_original_aspect_ratio=decrease,
   pad=<W>:<H>:(ow-iw)/2:(oh-ih)/2:color=black` + optional
   `,eq=brightness=-<darken*0.5>` (darken>0) + optional
   `,boxblur=<round(blur*20)>` (blur>0) + `,subtitles=<ass>,format=yuv420p`;
   `-shortest`, `-t <dur_s>` both included (image loops infinitely).
5. Mode `source_video`: `-i <input_media>` + instrumental;
   `-map 0:v:0 -map 1:a:0`; same scale/pad/darken/blur/subtitles chain;
   fps: `-r <min(style.fps, 60)>`; `-shortest`.
6. Filtergraph strings must contain only builder-controlled tokens: the ass
   path and background path are the fixed workspace constants; assert via
   an allowlist regex `^[A-Za-z0-9/_=:.,()\[\]\-]+$` over the final `-vf`
   value (defense in depth) — raise `WorkspaceError` on violation.
7. `dur_s` formatted `%.3f` from `duration_ms`.
8. Table-driven tests: one exact-argv golden per mode with default style;
   gain on/off; darken/blur permutations; absolute-path and `..` rejection;
   allowlist rejection (inject `;` via a crafted relpath).

## Acceptance Criteria

- [ ] Exact argv goldens for the three modes committed and passing.
- [ ] All path/allowlist violations raise before any argv is returned.
- [ ] No `subprocess` import in the module (grep test from 07 still passes).
- [ ] mypy strict; coverage ≥ 95%.

## Validation

```bash
uv run pytest tests/render/test_command.py -q
```

## Dependencies

- 07 (ffmpeg path type/constants), 14 (StyleConfig)

## Non-goals

- Execution/progress (16); background image validation (14 helper);
  loudness normalization (v2).

## Design References

- DESIGN §5.5 (table + encoding), §8 T2
- ADR-004
