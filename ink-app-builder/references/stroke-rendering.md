# Stroke Rendering

Reference for rendering completed (`dry`) strokes onto a Compose Canvas using `CanvasStrokeRenderer`.

## CanvasStrokeRenderer

`CanvasStrokeRenderer` is the primary API for drawing finalized `Stroke` objects onto an Android `Canvas`. It is provided by the `ink-rendering` module.

### Factory Method

```kotlin
import androidx.ink.rendering.android.canvas.CanvasStrokeRenderer

// Basic renderer (no textures; uses default TextureBitmapStore { null })
val renderer = CanvasStrokeRenderer.create()

// Renderer with texture support
val renderer = CanvasStrokeRenderer.create(textureStore = myTextureStore)
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `textureStore` | `TextureBitmapStore` | `TextureBitmapStore { null }` | Provides bitmaps for textured brushes. Required for emoji/custom texture brushes (non-nullable; do not pass `null`). |

> **Note on Offscreen / Software `Canvas` Rendering**: Starting in `1.0.0` (Stable), `CanvasStrokeRenderer.create(textureStore)` automatically detects software canvases (`!canvas.isHardwareAccelerated`, such as `Canvas(bitmap)`) and falls back to path rendering (`Canvas.drawPath`). The 2-argument `CanvasStrokeRenderer.create(forcePathRendering = true, textureStore)` overload seen in early `1.0.0-alpha` samples is `@RestrictTo(LIBRARY_GROUP)` (and `@InkInternalOnlyApi` in `1.1.0-alpha09+`) and must **not** be called in app code.

### Re-creating with Texture Cache

When textures are loaded dynamically, re-create the renderer when the texture store's generation changes:

```kotlin
import androidx.compose.runtime.getValue
import androidx.compose.runtime.remember
import androidx.ink.rendering.android.canvas.CanvasStrokeRenderer
import androidx.lifecycle.compose.collectAsStateWithLifecycle

val textureStore = LocalTextureStore.current
val cacheGen by textureStore.generation.collectAsStateWithLifecycle()
val canvasStrokeRenderer = remember(textureStore, cacheGen) {
    CanvasStrokeRenderer.create(textureStore)
}
```

## Drawing Strokes in Compose Canvas

Use the Compose `Canvas` composable and access the native Android `Canvas` for rendering:

```kotlin
import android.graphics.Matrix
import androidx.compose.foundation.Canvas
import androidx.compose.ui.graphics.nativeCanvas
import androidx.core.graphics.withSave

Canvas(modifier = Modifier.fillMaxSize()) {
    val nativeCanvas = drawContext.canvas.nativeCanvas
    strokes.forEach { stroke ->
        nativeCanvas.withSave {
            canvasStrokeRenderer.draw(
                canvas = this,
                stroke = stroke,
                strokeToScreenTransform = Matrix()
            )
        }
    }
}
```

### `draw()` Method Signature

```kotlin
canvasStrokeRenderer.draw(
    canvas: Canvas,           // Android native Canvas
    stroke: Stroke,           // The stroke to render
    strokeToScreenTransform: Matrix  // Transform matrix (identity for 1:1 mapping)
)
```

The `strokeToScreenTransform` matrix maps stroke coordinates to screen coordinates. Use `Matrix()` (identity) when strokes are stored in screen coordinates. For pan/zoom scenarios, pass the active world-to-screen transformation matrix.

## Blend Modes for Brush Types

Different brush families require different blend modes. Use `BlendMode.Multiply` for highlighter brushes to create a translucent highlight effect over existing content.

### Per-Stroke Blend Mode with `withSaveLayer`

```kotlin
import android.graphics.Matrix
import androidx.compose.foundation.Canvas
import androidx.compose.ui.geometry.toRect
import androidx.compose.ui.graphics.BlendMode
import androidx.compose.ui.graphics.nativeCanvas
import androidx.compose.ui.graphics.withSaveLayer
import androidx.core.graphics.withSave
import androidx.ink.brush.StockBrushes

Canvas(modifier = Modifier.fillMaxSize()) {
    val nativeCanvas = drawContext.canvas.nativeCanvas

    strokes.forEach { stroke ->
        // Determine blend mode based on brush family
        val blendMode = if (stroke.brush.family == StockBrushes.highlighter()) {
            BlendMode.Multiply
        } else {
            BlendMode.SrcOver
        }

        // Use withSaveLayer to apply blend mode per-stroke
        drawContext.canvas.withSaveLayer(
            drawContext.size.toRect(),
            androidx.compose.ui.graphics.Paint().apply {
                this.blendMode = blendMode
            }
        ) {
            nativeCanvas.withSave {
                canvasStrokeRenderer.draw(
                    canvas = this,
                    stroke = stroke,
                    strokeToScreenTransform = Matrix()
                )
            }
        }
    }
}
```

### Why `withSaveLayer`?

- `withSaveLayer` creates an offscreen buffer for each stroke.
- The blend mode is applied when the buffer is composited back onto the main canvas.
- Without it, the blend mode would only affect how each pixel of the stroke blends with what's already on the canvas, leading to incorrect rendering for semi-transparent brushes.

## Rendering to an Offscreen Bitmap (Export)

For exporting drawings as images, render strokes to an offscreen `Bitmap`:

```kotlin
import android.graphics.Bitmap
import android.graphics.Canvas
import androidx.core.graphics.createBitmap
import androidx.core.graphics.withSave
import androidx.ink.brush.TextureBitmapStore
import androidx.ink.rendering.android.canvas.CanvasStrokeRenderer
import androidx.ink.strokes.Stroke

suspend fun createExportBitmap(
    width: Int,
    height: Int,
    strokes: List<Stroke>,
    textureStore: TextureBitmapStore
): Bitmap {
    val bitmap = createBitmap(width, height)
    val canvas = Canvas(bitmap)

    // Draw background
    canvas.drawColor(android.graphics.Color.WHITE)

    // Since 1.0.0 stable, CanvasStrokeRenderer.create() automatically falls back
    // to path rendering when drawing onto a software Canvas(bitmap).
    val renderer = CanvasStrokeRenderer.create(textureStore = textureStore)

    strokes.forEach { stroke ->
        canvas.withSave {
            renderer.draw(
                canvas = this,
                stroke = stroke,
                strokeToScreenTransform = android.graphics.Matrix()
            )
        }
    }

    return bitmap
}
```

> **Path Rendering Caveat on Software `Canvas(bitmap)`**: When `CanvasStrokeRenderer` falls back to path rendering (`CanvasPathRenderer`) on a software `Canvas(bitmap)`, strokes using mesh-only features — such as `SelfOverlap.ACCUMULATE` or `STAMPING` textures (`BrushPaint.StampingTexture`, like `StockBrushes.emojiHighlighter()`'s mini-emoji trail) — cannot be drawn by `CanvasPathRenderer` and are skipped (and per-vertex opacity/color `BrushBehavior`s are not rendered). Standard `StockBrushes` (`pressurePen()`, `marker()`, `highlighter()`, `dashedLine()`, which use `SelfOverlap.ANY` or `SelfOverlap.DISCARD`) render on software `Canvas(bitmap)` without issue.

## Common Pitfalls

- **Calling `forcePathRendering = true`**: The 2-arg `CanvasStrokeRenderer.create(forcePathRendering, textureStore)` overload is `@RestrictTo(LIBRARY_GROUP)` (and `@InkInternalOnlyApi` in `1.1.0-alpha09+`). Always use the public 1-arg `CanvasStrokeRenderer.create(textureStore)`, which automatically falls back to path rendering on software `Bitmap` canvases.
- **Not wrapping `draw()` in `withSave`**: Always wrap individual `draw()` calls in `canvas.withSave { }` to prevent transform/clip leakage between strokes.
- **Using Compose `Canvas` APIs directly**: The Ink renderer requires a native Android `Canvas`. Access it via `drawContext.canvas.nativeCanvas`. Do not attempt to use Compose `DrawScope` methods to render strokes.
- **Stale renderer after texture load**: If you create the renderer once and then load new textures later, those textures won't render correctly. Re-create the renderer or use the `remember(cacheGen)` pattern.
- **Incorrect blend mode for highlighter**: Using `BlendMode.SrcOver` (the default) with a highlighter makes it look like a flat semi-transparent stroke. Use `BlendMode.Multiply` for the characteristic highlighting effect.
