# Brush Paint Configuration

## Overview

`BrushPaint` controls the visual rendering of a brush stroke — blending, opacity, color modification, and textures. Each `BrushCoat` accepts either a single `paint: BrushPaint` or a prioritized list `paintPreferences: List<BrushPaint>` (the renderer selects and draws the *first* `BrushPaint` in `paintPreferences` that is compatible with the device and renderer).

## SelfOverlap

`SelfOverlap` (default: `SelfOverlap.ANY`) controls how parts of the **same stroke** that intersect itself are treated during rendering:

| Value | Behavior | Use Case & Renderer Compatibility |
|-------|----------|-----------------------------------|
| `SelfOverlap.ANY` **(default)** | Uses whichever mode is most efficient and feature-complete for the brush and device (`ACCUMULATE` via `CanvasMeshRenderer` on Android U+ / API 34+, or `DISCARD` via `CanvasPathRenderer` on Android T- / software `Canvas`). | General-purpose default for most brushes (pens, markers, watercolor, and any coat with opacity/color `BrushBehavior`s). |
| `SelfOverlap.ACCUMULATE` | Overlapping regions within the stroke accumulate opacity (double opacity where a translucent stroke crosses itself). Requires `CanvasMeshRenderer` (Android 14 / API 34+ hardware `Canvas`); **cannot** be drawn by `CanvasPathRenderer` (if no fallback `BrushPaint` with `ANY` or `DISCARD` is provided in `BrushCoat.paintPreferences`, the coat is skipped on API ≤ 33 and software `Canvas(bitmap)`). | Physical marker, pencil shading, or watercolor effects where self-overlap must always build up density on API 34+. |
| `SelfOverlap.DISCARD` | Discards overlapping content within the stroke so the entire stroke renders as a flat, uniform-opacity outline (`CanvasPathRenderer`). **Incompatible** with `BrushPaint.StampingTexture` and `BrushBehavior`s targeting opacity (`OPACITY_MULTIPLIER`) or color (`HUE_OFFSET_IN_RADIANS`, `CHROMA_MULTIPLIER`, `LIGHTNESS_OFFSET`) on the same `BrushCoat` (those effects will not be rendered). | Translucent solid-color or `TilingTexture` strokes where self-intersection should not darken (e.g., `StockBrushes.highlighter(selfOverlap = SelfOverlap.DISCARD)`, PDF annotations). |

> **Clarification:** "self overlap" refers to overlap within a single stroke (e.g., drawing a loop). Overlap between different strokes is handled separately by the compositor.

## `BrushPaint.ColorFunction`

`BrushPaint.ColorFunction` (in `1.1.0-alpha03+`; top-level `androidx.ink.brush.ColorFunction` in `@RestrictTo` `1.1.0-alpha02`) is an abstract class that modifies the stroke color. A paint can have multiple color functions applied in list order.

### `OpacityMultiplier`

Multiplies the overall opacity of the stroke.

```kotlin
import androidx.ink.brush.BrushPaint

val opacityFunction = BrushPaint.ColorFunction.OpacityMultiplier(0.7f)  // 70% opaque
```

- Range: 0.0 (fully transparent) to 1.0 (fully opaque)
- This multiplies with the brush's base color alpha

### `ReplaceColor`

Ignores the brush's dynamic input color and replaces it with a fixed color for a specific coat/paint (for example, rendering a dark drop-shadow coat underneath a colored main coat). Use `withColorLong(...)` or `withColorIntArgb(...)`:

```kotlin
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.toArgb
import androidx.ink.brush.BrushPaint

// Using @ColorLong (preserves Compose Color space, e.g. sRGB / Display P3):
val fixedShadowColor = BrushPaint.ColorFunction.ReplaceColor.withColorLong(Color.Black.value.toLong())

// Or using @ColorInt ARGB:
val fixedShadowColorArgb = BrushPaint.ColorFunction.ReplaceColor.withColorIntArgb(Color.Black.toArgb())
```

## Code Examples

### Basic Paint Configuration

```kotlin
import androidx.ink.brush.BrushPaint
import androidx.ink.brush.SelfOverlap

val paint = BrushPaint(
    selfOverlap = SelfOverlap.DISCARD,
    colorFunctions = listOf(
        BrushPaint.ColorFunction.OpacityMultiplier(1f),
    ),
)
```

### Semi-Transparent Paint

```kotlin
import androidx.ink.brush.BrushPaint
import androidx.ink.brush.SelfOverlap

val semiTransparentPaint = BrushPaint(
    selfOverlap = SelfOverlap.ANY,
    colorFunctions = listOf(
        BrushPaint.ColorFunction.OpacityMultiplier(0.6f),
    ),
)
```

### Paint with Texture Layer

In `1.1.0-alpha03+`, `BrushPaint.TextureLayer` is an abstract class with concrete subclasses **`BrushPaint.TilingTexture`** (for tiled textures along the stroke) and **`BrushPaint.StampingTexture`** (for particle-stamped textures):

```kotlin
import androidx.ink.brush.BrushPaint
import androidx.ink.brush.SelfOverlap

val texturedPaint = BrushPaint(
    selfOverlap = SelfOverlap.ANY,
    colorFunctions = listOf(
        BrushPaint.ColorFunction.OpacityMultiplier(0.8f),
    ),
    textureLayers = listOf(
        BrushPaint.TilingTexture(
            clientTextureId = "pencil-grain",
            sizeX = 1.0f,
            sizeY = 1.0f,
            sizeUnit = BrushPaint.TextureLayer.SizeUnit.BRUSH_SIZE,
            blendMode = BrushPaint.TextureLayer.BlendMode.MODULATE,
        ),
    ),
)
```

### Using Paint in a BrushCoat

```kotlin
import androidx.ink.brush.BrushCoat
import androidx.ink.brush.BrushTip

val coat = BrushCoat(
    tip = BrushTip(
        scaleX = 1f,
        scaleY = 1f,
        cornerRounding = 1f,
    ),
    paintPreferences = listOf(paint), // or simply: paint = paint
)
```

## Common Pitfalls

- **Combining `SelfOverlap.DISCARD` with opacity/color `BrushBehavior`s or `StampingTexture`** — `SelfOverlap.DISCARD` forces the path renderer (`CanvasPathRenderer`), whereas per-vertex opacity (`OPACITY_MULTIPLIER`), color (`HUE_OFFSET_IN_RADIANS`, `CHROMA_MULTIPLIER`, `LIGHTNESS_OFFSET`) behaviors, and `BrushPaint.StampingTexture` require the mesh renderer (`CanvasMeshRenderer`). If specified on the same `BrushCoat`, those behavior or stamping effects will not be rendered. Use `SelfOverlap.ANY` or `SelfOverlap.ACCUMULATE` instead.
- **Using `SelfOverlap.ACCUMULATE` or `StampingTexture` without a fallback `BrushPaint` when supporting API ≤ 33 or bitmap export** — `CanvasMeshRenderer` requires Android 14+ (API 34+) and a hardware-accelerated `Canvas`. On Android 13 and below (API 26–33) or when rendering to a software `Canvas(bitmap)`, `CanvasStrokeRenderer` uses `CanvasPathRenderer`, which rejects any `BrushPaint` with `SelfOverlap.ACCUMULATE` or `BrushPaint.StampingTexture` (`canDraw` returns `false`). Either use `SelfOverlap.ANY` (which automatically falls back to path rendering when no `StampingTexture` is present) or supply a second, path-compatible fallback `BrushPaint` in `BrushCoat(tip = ..., paintPreferences = listOf(meshPaint, fallbackPathPaint))` so the coat is not silently skipped.
- **Using `SelfOverlap.DISCARD` for watercolor or shading** — Watercolor and shading effects need visible overlap accumulation (`SelfOverlap.ANY` or `SelfOverlap.ACCUMULATE`). Reserve `SelfOverlap.DISCARD` for uniform-opacity translucent strokes like highlighters (`StockBrushes.highlighter(selfOverlap = SelfOverlap.DISCARD)` — note that `StockBrushes.highlighter()` with no arguments defaults to `SelfOverlap.ANY`).
- **Setting `OpacityMultiplier` to 0** — This makes the stroke completely invisible.
- **Confusing `BrushCoat.paintPreferences` with `BrushFamily.coats`** — `paintPreferences: List<BrushPaint>` is a **prioritized fallback list** of alternative `BrushPaint` configurations for a single coat; the renderer selects only the *first* compatible `BrushPaint` in the list. If you want multiple rendering layers to composite together (e.g., a shadow layer plus a main stroke layer), use multiple `BrushCoat` objects in `BrushFamily(coats = listOf(coat0, coat1))`.
- **Calling `ReplaceColor.withComposeColor(Color.Black)`** — `ReplaceColor.withComposeColor` is an `@InkInternalOnlyApi` / `@RestrictTo(LIBRARY_GROUP)` method that accepts Ink's internal `androidx.ink.brush.color.Color` type, not `androidx.compose.ui.graphics.Color`. Use `BrushPaint.ColorFunction.ReplaceColor.withColorLong(composeColor.value.toLong())` or `withColorIntArgb(composeColor.toArgb())`.
- **Empty `paintPreferences` list** — A `BrushCoat` requires a non-empty `paintPreferences` list (or use the single-paint `BrushCoat(tip, paint)` constructor).
