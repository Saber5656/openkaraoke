# Title

Style configuration (style.toml schema + template)

## Summary

Implement the `StyleConfig` pydantic schema for `style.toml` (DESIGN §4.5),
its loader with range validation and helpful errors, and the commented
template file written into new workspaces.

## Context

Style is the user's knob for the visual output; it feeds the ASS builder (12)
and the render command builder (15). It is security-relevant (T7: path
fields, numeric ranges).

## Scope

- `src/openkaraoke/config/schema.py` (extend with `StyleConfig` tree)
- `src/openkaraoke/config/style_template.toml` (packaged data file)
- Loader function + unit tests

## Detailed Requirements

1. Model tree mirroring DESIGN §4.5 exactly: `StyleConfig{video: VideoStyle,
   background: BackgroundStyle, font: FontStyle, colors: ColorStyle,
   layout: LayoutStyle, timing_display: TimingDisplayStyle, audio: AudioStyle,
   schema_version: Literal[1]}` — every field with the documented default
   and `Field(ge=…, le=…)` ranges; `extra="forbid"` everywhere.
2. Color fields validated by regex `^#[0-9A-Fa-f]{6}([0-9A-Fa-f]{2})?$`.
3. `background.mode` enum `auto|solid|image|source_video`;
   `image_path`: empty string or a path — loader resolves it project-relative
   and requires existence + file type in `{png,jpg,jpeg,webp}` **only when
   mode == "image"** (checked by `resolve_background(style, workspace,
   input_media_type)` helper which also resolves `auto`).
4. `preset` enum: `ultrafast|superfast|veryfast|faster|fast|medium|slow|slower|veryslow`.
5. `load_style(path) -> StyleConfig`: tomllib parse; on validation error,
   `ConfigError` listing each failure as `table.key: message (got: value)`.
6. `style_hash(style) -> str` = sha256 of canonical JSON dump (used by the
   manifest for subtitle/render staleness).
7. Template file: every key present with default value and a one-line
   comment (English), matching the schema byte-for-byte on load
   (`load_style(template) == StyleConfig()` test).
8. `write_style_template(dest: Path)` copies the packaged template
   (importlib.resources), refusing overwrite unless `force`.

## Acceptance Criteria

- [ ] Template loads to exactly the default model.
- [ ] Out-of-range (`crf = 99`), bad color (`#12345`), unknown key, and
      missing-image-path cases produce exit-3 errors naming the key path.
- [ ] `auto` resolves to `source_video` iff input is video, else `solid`.
- [ ] `style_hash` stable across dict ordering / file formatting.
- [ ] mypy strict; coverage ≥ 95%.

## Validation

```bash
uv run pytest tests/config/test_style.py -q
```

## Dependencies

- 03

## Non-goals

- ASS mapping (12), ffmpeg mapping (15), fonts detection (26).

## Design References

- DESIGN §4.5, §8 T7, §5.4/§5.5 (consumers)
