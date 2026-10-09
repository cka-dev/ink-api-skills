# Stock Brushes

Reference for the built-in brush families provided by `StockBrushes` and how to create/modify `Brush` instances.

## Available Stock Brush Families

`StockBrushes` (from `ink-brush`) provides factory methods that return `BrushFamily` instances:

| Factory Method | Description | Typical Use |
|---|---|---|
| `StockBrushes.pressurePen()` | Pressure-sensitive pen with variable width | Primary writing/drawing tool |
| `StockBrushes.marker()` | Uniform-width marker | Bold drawing, flat strokes |
| `StockBrushes.highlighter(selfOverlap = SelfOverlap.ANY)` | Semi-transparent highlighter (`selfOverlap` defaults to `SelfOverlap.ANY`; pass `SelfOverlap.DISCARD` to suppress self-overlap darkening on API 34+) | Highlighting over existing content |
| `StockBrushes.dashedLine()` | Dashed/dotted line pattern | Borders, guides, annotations |
| `StockBrushes.emojiHighlighter(clientTextureId, showMiniEmojiTrail = false, selfOverlap = SelfOverlap.ANY)` | Textured highlighter with emoji particles (`showMiniEmojiTrail = true` requires Android 14 / API 34+ mesh rendering) | Decorative highlighting |

### Emoji Highlighter Variants

The emoji highlighter accepts a `clientTextureId` string, an optional `showMiniEmojiTrail` boolean (default `false`), and an optional `selfOverlap: SelfOverlap = SelfOverlap.ANY`:

> **Android U (API 34+) Requirement for `showMiniEmojiTrail`**: Per `StockBrushes.emojiHighlighter` KDoc, the miniature emoji trail coat uses particle stamping (`CanvasMeshRenderer`), which only renders properly starting with **Android 14 (`Build.VERSION_CODES.UPSIDE_DOWN_CAKE`, API 34+)**. On Android 13 and below (API 26–33), always set `showMiniEmojiTrail = false` (or `Build.VERSION.SDK_INT >= Build.VERSION_CODES.UPSIDE_DOWN_CAKE`).

```kotlin
import android.os.Build
import androidx.ink.brush.StockBrushes

val supportsMiniEmojiTrail = Build.VERSION.SDK_INT >= Build.VERSION_CODES.UPSIDE_DOWN_CAKE

// Heart emoji highlighter with mini trail (enabled on API 34+)
val heartHighlighter = StockBrushes.emojiHighlighter(
    clientTextureId = "emoji-heart",
    showMiniEmojiTrail = supportsMiniEmojiTrail
)

// Star emoji highlighter
val starHighlighter = StockBrushes.emojiHighlighter(
    clientTextureId = "emoji-star",
    showMiniEmojiTrail = supportsMiniEmojiTrail
)

// Poop emoji highlighter
val poopHighlighter = StockBrushes.emojiHighlighter(
    clientTextureId = "emoji-poop",
    showMiniEmojiTrail = supportsMiniEmojiTrail
)
```

> **Note**: Emoji highlighters require a `TextureBitmapStore` that maps the emoji ID to a `Bitmap`. See `drawing-surface.md` for the `TextureBitmapStore` pattern.

## Creating a Brush

A `Brush` combines a `BrushFamily` with color, size, and epsilon. Use the Compose extension function for Compose color interop:

```kotlin
import androidx.ink.brush.Brush
import androidx.ink.brush.StockBrushes
import androidx.ink.brush.compose.createWithComposeColor
import androidx.compose.ui.graphics.Color

val brush = Brush.createWithComposeColor(
    family = StockBrushes.pressurePen(),
    color = Color.Gray,
    size = 5f,
    epsilon = 0.1f
)
```

### Parameters

| Parameter | Type | Description |
|---|---|---|
| `family` | `BrushFamily` | The brush family defining stroke shape and behavior |
| `color` | `Color` | Compose color for the stroke |
| `size` | `Float` | Overall stroke width in stroke-space units. Must be finite, strictly positive (`size > 0f`), and **greater than or equal to `epsilon` (`size >= epsilon`)** |
| `epsilon` | `Float` | Geometric precision (minimum distance between two points to be considered distinct). Must be finite and strictly positive (`0f < epsilon <= size`). Typical range: `0.01f` to `0.1f` |

### The `epsilon` Parameter

`epsilon` controls the geometric approximation tolerance of rendered strokes:

- **Lower values** (e.g., `0.01f`): Higher fidelity curves, more memory/CPU usage. Use for fine drawing tools.
- **Higher values** (e.g., `0.1f`): Coarser curves, lower resource usage. Suitable for general drawing.
- **Standard default**: Use `0.1f` for general drawing, or `0.01f` for precision tools.
- **Constraint (`size >= epsilon`)**: `Brush` enforces that `size >= epsilon` (as well as `size > 0f` and `epsilon > 0f`). Passing a `size` less than `epsilon` (for example, when zooming or scaling a brush down without scaling `epsilon`) throws `IllegalArgumentException`.

## Modifying Brushes

Brushes are immutable. Use `copy()` and Compose extension methods to derive new brushes:

### Changing the Brush Family

```kotlin
import androidx.ink.brush.StockBrushes

// Switch from pressure pen to marker, preserving color and size
val markerBrush = currentBrush.copy(family = StockBrushes.marker())
```

### Changing the Color

```kotlin
import androidx.ink.brush.compose.copyWithComposeColor
import androidx.compose.ui.graphics.Color

val redBrush = currentBrush.copyWithComposeColor(Color.Red)
```

### Changing the Size

```kotlin
val thickBrush = currentBrush.copy(size = 20f)
```

### Changing Family and Size Together

```kotlin
val newBrush = currentBrush.copy(
    family = StockBrushes.marker(),
    size = 15f
)
```

### Reading the Current Compose Color

```kotlin
import androidx.ink.brush.compose.composeColor

val currentColor: Color = currentBrush.composeColor
```

## Alpha Handling for Highlighter Brushes

Use reduced alpha on highlighter-type brushes to create a semi-transparent effect:

```kotlin
import android.os.Build
import androidx.compose.ui.graphics.Color
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.StockBrushes
import androidx.ink.brush.compose.composeColor
import androidx.ink.brush.compose.copyWithComposeColor

private const val HIGHLIGHTER_ALPHA = 0.3f
private val supportsMiniEmojiTrail = Build.VERSION.SDK_INT >= Build.VERSION_CODES.UPSIDE_DOWN_CAKE

fun changeBrush(brushFamily: BrushFamily) {
    val currentColor = currentBrush.composeColor
    val colorToApply = when (brushFamily) {
        StockBrushes.highlighter() ->
            currentColor.copy(alpha = HIGHLIGHTER_ALPHA)
        StockBrushes.emojiHighlighter("emoji-heart", showMiniEmojiTrail = supportsMiniEmojiTrail) ->
            Color(0xFFFF45CA).copy(alpha = HIGHLIGHTER_ALPHA)
        StockBrushes.emojiHighlighter("emoji-star", showMiniEmojiTrail = supportsMiniEmojiTrail) ->
            Color(0xFFFFE100).copy(alpha = HIGHLIGHTER_ALPHA)
        else ->
            currentColor.copy(alpha = 1f)
    }
    selectedBrush = currentBrush
        .copy(family = brushFamily)
        .copyWithComposeColor(colorToApply)
}
```

### Pattern: Brush Switching with Alpha Preservation

When switching between brush families, adjust alpha based on whether the target is a highlighter:

```kotlin
fun changeBrushColor(newColor: Color) {
    val isHighlighter = currentBrush.family == StockBrushes.highlighter()
        || currentBrush.family == StockBrushes.emojiHighlighter("emoji-heart", showMiniEmojiTrail = supportsMiniEmojiTrail)
        // ... other highlighter checks

    val adjustedColor = if (isHighlighter) {
        newColor.copy(alpha = HIGHLIGHTER_ALPHA)
    } else {
        newColor.copy(alpha = 1f)
    }

    selectedBrush = currentBrush.copyWithComposeColor(adjustedColor)
}
```

## BrushFamily Equality Comparison (`==`)

`BrushFamily` implements structural `equals()` and `hashCode()` (and non-emoji `StockBrushes` also return cached singleton instances). Always compare `BrushFamily` instances using Kotlin's structural equality operator (`==`), not reference equality (`===`), since `StockBrushes.emojiHighlighter(...)` constructs a new `BrushFamily` instance on each call:

```kotlin
if (stroke.brush.family == StockBrushes.highlighter()) {
    // Apply highlighter-specific rendering
}
```

## Updating an Existing Stroke's Brush (`Stroke.copy`)

To recolor or change the brush of an already-drawn `Stroke`, use `stroke.copy(brush = newBrush)`:

```kotlin
import androidx.compose.ui.graphics.Color
import androidx.ink.brush.compose.copyWithComposeColor
import androidx.ink.strokes.Stroke

fun recolorStroke(stroke: Stroke, newColor: Color): Stroke {
    val updatedBrush = stroke.brush.copyWithComposeColor(newColor)
    return stroke.copy(brush = updatedBrush)
}
```

> **Performance Note**: `Stroke.copy(brush = ...)` automatically checks whether the new `Brush` requires a different `PartitionedMesh`. When only the brush color changes (same `size`, `epsilon`, `inputModel`, and compatible `BrushCoat`s/`BrushTip`s), `Stroke.copy` reuses the existing `PartitionedMesh` instance (`stroke.shape`) without regenerating geometry, preserving rendering cache hits.

## Common Pitfalls

- **Forgetting alpha on highlighter brushes**: Highlighter brushes without reduced alpha (e.g., `0.3f`) look like opaque strokes, not highlights.
- **Mutating brushes**: `Brush` is immutable. `copy()` and `copyWithComposeColor()` return new instances; they do not modify the original.
- **Using `createWithComposeColor` without `ink-brush-compose`**: This extension function requires the `ink-brush-compose` dependency. Without it, you must use `Brush.createWithColorLong()` and convert colors manually.
- **Violating `size >= epsilon`**: `Brush` requires `0f < epsilon <= size`. Setting `size < epsilon` (e.g., `currentBrush.copy(size = 0.05f)` when `epsilon = 0.1f`) throws `IllegalArgumentException` at runtime.
- **Epsilon below `0.01f`**: Setting `epsilon` below `0.01f` (e.g., `0.001f`) dramatically increases memory consumption and stroke data size. Keep `epsilon >= 0.01f`.
- **Emoji highlighter without texture store**: Emoji brush families require a `TextureBitmapStore` with the corresponding bitmap loaded. Without it, the brush renders as a plain fill.
