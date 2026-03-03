# LaTeX Math Rendering Plan

Tracking issue: #30

## Context

GitHub supports LaTeX math in markdown: inline `$...$` and display `$$...$$` / ` ```math ` blocks. Comrak 0.31 (already a dependency) has built-in math support via `NodeValue::Math(NodeMath)` — just needs two options enabled. No custom delimiter parsing required.

## Rendering pipeline

All pure Rust, same shape as the existing mermaid pipeline:

```
LaTeX source
  → mitex (LaTeX → Typst math syntax)
  → typst + typst-svg (Typst → SVG)
  → resvg (SVG → raster image)  [already a dependency]
  → terminal image protocol (Kitty/Sixel/iTerm2)
```

Fallback for terminals without image support: framed text block with Unicode approximation via `unicodeit`.

## Comrak math support

Comrak's `NodeMath` struct:

```rust
pub struct NodeMath {
    pub dollar_math: bool,    // true for $ or $$, false for code math
    pub display_math: bool,   // true for $$ or ```math, false for inline $
    pub literal: String,      // raw LaTeX source
}
```

| Syntax | Comrak node | `display_math` |
|--------|-------------|----------------|
| `$x^2$` | `NodeValue::Math` (inline child of Paragraph) | `false` |
| `$$E = mc^2$$` | `NodeValue::Math` (inline child of Paragraph) | `true` |
| `` $`x^2`$ `` | `NodeValue::Math` (code math) | `false` |
| ` ```math ` | `NodeValue::CodeBlock` with `info == "math"` | n/a |

Enable with: `options.extension.math_dollars = true` and `options.extension.math_code = true`.

## Implementation steps

### 1. Enable comrak math parsing

**`src/document/parser.rs`** — `create_options()` (~line 194): add `math_dollars = true` and `math_code = true`.

### 2. Add dependencies

**`Cargo.toml`**:
- `unicodeit = "0.2"` — LaTeX → Unicode text (for inline math and fallback)
- `mitex = "0.2"` — LaTeX → Typst math conversion
- `typst = "0.14"` — Typst compiler library
- `typst-svg = "0.14"` — Typst → SVG export

### 3. Create `src/math.rs` module

New file parallel to `src/mermaid.rs`:

```rust
/// Convert LaTeX math to Unicode text approximation (for inline and fallback).
pub fn latex_to_unicode(latex: &str) -> String { ... }

/// Render LaTeX math to SVG string.
/// LaTeX → mitex → Typst → typst-svg → SVG
pub fn render_to_svg(latex: &str) -> Result<String> { ... }

/// Render LaTeX math to raster image.
/// Calls render_to_svg() then rasterize_svg() (reuse from mermaid.rs).
pub fn render_to_image(latex: &str, target_width_px: u32) -> Result<DynamicImage> { ... }
```

Wrap rendering in `catch_unwind` for safety (same pattern as `src/mermaid.rs`).

Add `pub mod math;` to `src/lib.rs`.

### 4. Add `math` flag to `InlineStyle`

**`src/document/types.rs`** (~line 443): add `pub math: bool`. Already derives `Default` so it's `false` automatically.

### 5. Add `LineType::Math`

**`src/document/types.rs`** — Add `Math` variant to `LineType` enum so the renderer can style math blocks distinctly from code blocks.

### 6. Add math tracking to parser and model

Following the mermaid pattern:

**`src/document/types.rs`** — Add `math_sources: HashMap<String, String>` to `ParsedDocument`.

**`src/document/parser.rs`** — Add to `ParseContext`:
- `math_sources: HashMap<String, String>` — maps `math://N` → LaTeX source
- `math_as_images: bool` — whether to emit image placeholders
- `failed_math_srcs: &'h HashSet<String>` — sources that failed rendering

**`src/app/model.rs`** — Add to `Model`:
- `failed_math_srcs: HashSet<String>`

**`src/document/mod.rs`** — Add `math_sources` getter to `Document` (like `mermaid_sources()`).

Reuse `should_render_mermaid_as_images()` or generalize it (same condition: real image protocol, not halfblocks).

### 7. Handle inline math (`$...$`) in span collection

**`src/document/parser.rs`** — `collect_inline_spans_recursive()` (~line 1257): add arm for `NodeValue::Math` with `display_math == false`. Set `math: true` on style, convert literal via `latex_to_unicode()`, push as `InlineSpan`. Inline math is always text — can't be an inline image.

Also add arm in `extract_text_recursive()` (~line 1346) so heading text extraction, search, and TOC work with math nodes.

### 8. Handle display math (`$$...$$`) — image or text block

Display math nodes are emitted by comrak as inline children of `Paragraph`. Detect in the `Paragraph` handler (~line 266):

- Add helper `fn extract_display_math_literal(node) -> Option<String>` — returns `Some(literal)` if paragraph contains only a `Math` node with `display_math == true`
- **When `math_as_images` is true** (terminal supports images):
  - Store source in `math_sources` with `math://N` key
  - Check `failed_math_srcs` — if source failed, fall through to text block
  - Emit `ImageRef` placeholder lines (same pattern as mermaid at parser.rs:350-385)
- **When `math_as_images` is false** (fallback):
  - Render as framed text block with `unicodeit` conversion
  - Frame pattern: `┌ math ─┐` / `│ E = mc² │` / `└─────┘`
  - Uses `LineType::Math`
- When display math is mixed with other text, fall through to normal paragraph rendering with inline math styling

### 9. Handle ` ```math ` code blocks

**`src/document/parser.rs`** — In the `CodeBlock` handler (~line 335), add branch for `language == Some("math")` (after mermaid, before csv). Same logic as step 8: image placeholder when supported, framed text block otherwise.

### 10. Integrate with `load_nearby_images()`

**`src/app/model.rs`** — In `load_nearby_images()` (~line 342), add branch for `src.starts_with("math://")`:

```rust
else if src.starts_with("math://") {
    let math_text = self.document.math_sources().get(&src).cloned();
    if let Some(math_text) = math_text {
        match crate::math::render_to_image(&math_text, target_width_px) {
            Ok(img) => {
                self.original_images.insert(src.clone(), img.clone());
                Some(img)
            }
            Err(e) => {
                if self.failed_math_srcs.insert(math_text) {
                    math_failed = true;
                }
                None
            }
        }
    }
}
```

Track `math_failed` and trigger `reflow_layout()` on failure (same as mermaid).

### 11. Add math styling

**`src/ui/style.rs`** — `style_for_inline()` (~line 102): add `if inline.math` branch — green foreground + italic.

Add `LineType::Math` handling in line-type base styles — green + italic.

### 12. Thread math flags through parse APIs

Update all `Document::parse_with_*` methods and `reparse_document()` to pass `math_as_images` and `failed_math_srcs` through to the parser (same pattern as `mermaid_as_images` / `failed_mermaid_srcs`).

### 13. Refactor `rasterize_svg` to shared location

Currently `rasterize_svg()` lives in `src/mermaid.rs`. Move it to a shared location (e.g. `src/math.rs` can call `crate::mermaid::rasterize_svg()`, or extract it to a common module) so both mermaid and math can use it.

## Files to modify

| File | Change |
|------|--------|
| `Cargo.toml` | Add `unicodeit`, `mitex`, `typst`, `typst-svg` |
| `src/lib.rs` | Add `pub mod math;` |
| `src/math.rs` | **New** — `latex_to_unicode()`, `render_to_svg()`, `render_to_image()` |
| `src/document/types.rs` | Add `math: bool` to `InlineStyle`, `LineType::Math`, `math_sources` to `ParsedDocument` |
| `src/document/mod.rs` | Add `math_sources` field + getter to `Document` |
| `src/document/parser.rs` | Enable math options, handle `NodeValue::Math` inline + display, handle ` ```math `, add `math_sources`/`math_as_images`/`failed_math_srcs` to `ParseContext`, thread through parse APIs |
| `src/ui/style.rs` | Style `math` spans and `LineType::Math` lines |
| `src/app/model.rs` | Add `failed_math_srcs`, handle `math://` in `load_nearby_images()`, thread math flags through `reparse_document()` |

## Tests

**Parser tests** (`src/document/parser.rs`):

| Test | Input | Assertion |
|------|-------|-----------|
| `test_inline_math_produces_styled_span` | `$x^2$` in paragraph | Span has `math: true` |
| `test_inline_math_unicode_conversion` | `$\alpha$` | Span text contains `α` |
| `test_display_math_renders_as_image_when_flag_set` | `$$E = mc^2$$` with `math_as_images=true` | `ImageRef` with `math://` src |
| `test_display_math_renders_as_text_block_without_flag` | `$$E = mc^2$$` with `math_as_images=false` | `LineType::Math` framed block |
| `test_display_math_mixed_stays_inline` | `text $$x$$ more` | `LineType::Paragraph` with math-styled span |
| `test_math_code_fence_as_image` | ` ```math ` with flag | `ImageRef` with `math://` src |
| `test_math_code_fence_as_text` | ` ```math ` without flag | `LineType::Math` framed block |
| `test_math_code_inline` | `` $`x^2`$ `` | Math-styled inline span |
| `test_math_in_heading_text_extraction` | `# Title $x$` | TOC text includes `x` |
| `test_math_source_stored` | `$$E = mc^2$$` | `math_sources` has entry |
| `test_math_falls_back_when_in_failed_set` | source in `failed_math_srcs` | No `ImageRef`, framed text block |

**Math module tests** (`src/math.rs`):

| Test | Input | Expected |
|------|-------|----------|
| `test_latex_to_unicode_greek` | `\alpha` | `α` |
| `test_latex_to_unicode_superscript` | `x^2` | contains superscript |
| `test_latex_to_unicode_passthrough` | unknown command | unchanged |
| `test_render_to_svg_simple` | `E = mc^2` | valid SVG string |
| `test_render_to_image_simple` | `x^2 + y^2` | `DynamicImage` with nonzero dimensions |
| `test_render_to_svg_invalid_does_not_panic` | garbage input | `Err`, no panic |

**Model tests** (`src/app/tests.rs`):

| Test | Input | Assertion |
|------|-------|----------|
| `test_reflow_after_picker_converts_math_blocks_to_images` | Model with `$$` doc + Kitty picker | `reflow_layout()` produces `ImageRef` |

## Verification

```bash
cargo test                         # All tests pass
cargo clippy -- -D warnings        # No warnings
./scripts/check.sh                 # Full check suite
```

Manual: `cargo run -- test_math.md` with inline `$\alpha + \beta$` and display `$$E = mc^2$$`, verify image rendering on a Kitty/iTerm2 terminal and text fallback on others.
