# Advanced Topics

Reference for advanced Ink API patterns including motion prediction, bitmap export, drag & drop coexistence, and custom textures.

## Motion Prediction

### Dependency

```toml
# gradle/libs.versions.toml
[libraries]
androidx-input-motionprediction = { module = "androidx.input:input-motionprediction", version.ref = "inputMotionPrediction" }
```

```kotlin
// app/build.gradle.kts
implementation(libs.androidx.input.motionprediction)
```

### What It Does

The `androidx.input.motionprediction` library predicts future touch/stylus points based on the current input trajectory. This reduces perceived latency by rendering predicted points ahead of the actual input.

The `InProgressStrokes` composable integrates with motion prediction automatically when the dependency is available on the classpath. No additional code is required — simply including the dependency enables the feature.

### Impact

| Without Prediction | With Prediction |
|---|---|
| Stroke trails behind the stylus | Stroke appears to track the stylus in real-time |
| ~20-50ms perceived latency | Near-zero perceived latency |
| Acceptable for finger input | Critical for stylus input |

> **Recommendation**: Always include the motion prediction dependency for production apps, especially those targeting stylus input.

## Bitmap Export

Export the current drawing canvas as a `Bitmap` for sharing, saving to gallery, or generating thumbnails.

### Full Implementation

```kotlin
import android.graphics.Bitmap
import android.graphics.Canvas
import android.graphics.Matrix
import androidx.core.graphics.createBitmap
import androidx.core.graphics.withSave
import androidx.ink.brush.TextureBitmapStore
import androidx.ink.rendering.android.canvas.CanvasStrokeRenderer
import androidx.ink.strokes.Stroke

suspend fun createExportBitmap(
    width: Int,
    height: Int,
    strokes: List<Stroke>,
    textureStore: TextureBitmapStore,
    backgroundColor: Int = android.graphics.Color.WHITE,
    backgroundBitmap: Bitmap? = null,
): Bitmap {
    val exportBitmap = createBitmap(width, height)
    val canvas = Canvas(exportBitmap)

    // 1. Draw solid background color
    canvas.drawColor(backgroundColor)

    // 2. Draw background image (if any), scaled to fill
    backgroundBitmap?.let { bmp ->
        val scaleX = width.toFloat() / bmp.width.toFloat()
        val scaleY = height.toFloat() / bmp.height.toFloat()
        val scale = maxOf(scaleX, scaleY)

        val dx = (width - bmp.width * scale) / 2f
        val dy = (height - bmp.height * scale) / 2f

        val matrix = Matrix().apply {
            setScale(scale, scale)
            postTranslate(dx, dy)
        }
        canvas.drawBitmap(bmp, matrix, null)
    }

    // 3. Render strokes (since 1.0.0 stable, CanvasStrokeRenderer.create() automatically
    // falls back to path rendering when drawing onto a software Canvas(bitmap))
    val renderer = CanvasStrokeRenderer.create(textureStore = textureStore)
    strokes.forEach { stroke ->
        canvas.withSave {
            renderer.draw(
                canvas = this,
                stroke = stroke,
                strokeToScreenTransform = Matrix()
            )
        }
    }

    return exportBitmap
}
```

### Saving to a File

```kotlin
import android.graphics.Bitmap
import android.net.Uri
import java.io.File
import java.io.FileOutputStream

fun saveBitmapToFile(bitmap: Bitmap, file: File): Uri {
    FileOutputStream(file).use { output ->
        bitmap.compress(Bitmap.CompressFormat.PNG, 100, output)
    }
    return Uri.fromFile(file)
}
```

### Key Points

- **Automatic path rendering fallback**: Starting in `1.0.0` (Stable), `CanvasStrokeRenderer.create(textureStore)` automatically detects software `Bitmap` canvases (`!canvas.isHardwareAccelerated`) and falls back to `Canvas.drawPath`. Do **not** call the 2-arg `CanvasStrokeRenderer.create(forcePathRendering = true, textureStore)` overload — it is `@RestrictTo(LIBRARY_GROUP)` (and `@InkInternalOnlyApi` in `1.1.0-alpha09+`).
- Use the canvas dimensions from the current view size for 1:1 export, or specify custom dimensions for thumbnails.
- Background images should be scaled to fill (`maxOf(scaleX, scaleY)`) and centered.

## Drag & Drop Coexistence

When a drawing surface needs to coexist with drag & drop (e.g., dragging images onto the canvas), standard `pointerInput` modifiers consume events and prevent siblings from receiving them.

### The Problem

By default, `pointerInput` modifiers do not share pointer events with sibling composables. If you have both a drag-and-drop handler and the `InProgressStrokes` composable, one will block the other.

### Solution: `pointerInputWithSiblingFallthrough`

Create a custom `Modifier` that delegates to the standard pointer input implementation but returns `true` from `sharePointerInputWithSiblings()`:

```kotlin
import androidx.compose.ui.Modifier
import androidx.compose.ui.input.pointer.PointerEvent
import androidx.compose.ui.input.pointer.PointerEventPass
import androidx.compose.ui.input.pointer.PointerInputEventHandler
import androidx.compose.ui.input.pointer.SuspendingPointerInputModifierNode
import androidx.compose.ui.node.DelegatingNode
import androidx.compose.ui.node.ModifierNodeElement
import androidx.compose.ui.node.PointerInputModifierNode
import androidx.compose.ui.platform.InspectorInfo
import androidx.compose.ui.unit.IntSize

internal fun Modifier.pointerInputWithSiblingFallthrough(
    pointerInputEventHandler: PointerInputEventHandler
) = this then PointerInputSiblingFallthroughElement(pointerInputEventHandler)

private class PointerInputSiblingFallthroughModifierNode(
    pointerInputEventHandler: PointerInputEventHandler
) : PointerInputModifierNode, DelegatingNode() {

    var pointerInputEventHandler: PointerInputEventHandler
        get() = delegateNode.pointerInputEventHandler
        set(value) { delegateNode.pointerInputEventHandler = value }

    val delegateNode = delegate(
        SuspendingPointerInputModifierNode(pointerInputEventHandler)
    )

    override fun onPointerEvent(
        pointerEvent: PointerEvent,
        pass: PointerEventPass,
        bounds: IntSize
    ) {
        delegateNode.onPointerEvent(pointerEvent, pass, bounds)
    }

    override fun onCancelPointerInput() {
        delegateNode.onCancelPointerInput()
    }

    // This is the key override — allows siblings to also receive pointer events
    override fun sharePointerInputWithSiblings() = true
}

private data class PointerInputSiblingFallthroughElement(
    val pointerInputEventHandler: PointerInputEventHandler
) : ModifierNodeElement<PointerInputSiblingFallthroughModifierNode>() {

    override fun create() =
        PointerInputSiblingFallthroughModifierNode(pointerInputEventHandler)

    override fun update(node: PointerInputSiblingFallthroughModifierNode) {
        node.pointerInputEventHandler = pointerInputEventHandler
    }

    override fun InspectorInfo.inspectableProperties() {
        name = "pointerInputWithSiblingFallthrough"
        properties["pointerInputEventHandler"] = pointerInputEventHandler
    }
}
```

### Usage in Drawing Surface

Use `pointerInputWithSiblingFallthrough` for the drag/long-press handler, so that regular touch input still reaches `InProgressStrokes`:

```kotlin
import androidx.compose.foundation.Canvas
import androidx.compose.foundation.gestures.detectDragGesturesAfterLongPress
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.ui.Modifier
import androidx.ink.authoring.compose.InProgressStrokes

Box(modifier = modifier) {
    // Dry ink canvas
    Canvas(modifier = Modifier.fillMaxSize()) { /* ... */ }

    // Wet ink (InProgressStrokes) — also receives pointer events shared by the sibling above
    InProgressStrokes(
        defaultBrush = currentBrush,
        nextBrush = onGetNextBrush,
        onStrokesFinished = onStrokesFinished,
        textureBitmapStore = textureStore
    )

    // Long-press drag handler (for drag & drop)
    // Placed AFTER InProgressStrokes so it is hit-tested first and shares pointer input with siblings
    Box(
        modifier = Modifier
            .fillMaxSize()
            .pointerInputWithSiblingFallthrough {
                detectDragGesturesAfterLongPress(
                    onDragStart = { onStartDrag() },
                    onDrag = { change, _ -> change.consume() },
                    onDragEnd = { /* ... */ }
                )
            }
    )
}
```

### Layering Order

The `pointerInputWithSiblingFallthrough` `Box` must be placed *after* `InProgressStrokes` in the `Box` children. Compose hit-tests `Box` children in reverse declaration order (last child to first) and stops at the first hit child unless `sharePointerInputWithSiblings()` returns `true`. Placing the `pointerInputWithSiblingFallthrough` `Box` last ensures it is hit-tested first and allows hit-testing to continue down to `InProgressStrokes`.

## TextureBitmapStore for Custom Textures

The `TextureBitmapStore` interface provides bitmaps to the Ink rendering pipeline for textured brushes (such as `StockBrushes.emojiHighlighter()` and custom textured brushes). It is a stable public API in `androidx.ink.brush`.

### Interface

```kotlin
fun interface TextureBitmapStore {
    operator fun get(clientTextureId: String): Bitmap?
}
```

### Implementation Pattern

```kotlin
import android.content.Context
import android.graphics.Bitmap
import android.graphics.BitmapFactory
import androidx.ink.brush.TextureBitmapStore
import dagger.hilt.android.qualifiers.ApplicationContext
import javax.inject.Inject
import javax.inject.Singleton
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update

@Singleton
class AppTextureBitmapStore @Inject constructor(
    @ApplicationContext context: Context
) : TextureBitmapStore {

    private val resources = context.resources

    // Map texture IDs to drawable resources
    private val textureResources: Map<String, Int> = mapOf(
        "emoji-heart" to R.drawable.emoji_heart,
        "emoji-star"  to R.drawable.emoji_star,
        "emoji-poop"  to R.drawable.emoji_poop,
        // Add custom textures here
    )

    private val loadedBitmaps = mutableMapOf<String, Bitmap>()

    // Generation counter for cache invalidation
    private val _generation = MutableStateFlow(0)
    val generation = _generation.asStateFlow()

    override operator fun get(clientTextureId: String): Bitmap? {
        val id = normalizeId(clientTextureId)
        return loadedBitmaps.getOrPut(id) {
            textureResources[id]?.let { resId ->
                BitmapFactory.decodeResource(resources, resId)
            } ?: return null
        }
    }

    fun loadTexture(textureId: String, bitmap: Bitmap) {
        val id = normalizeId(textureId)
        loadedBitmaps[id] = bitmap
        _generation.update { it + 1 }  // Triggers recomposition
    }

    private fun normalizeId(id: String): String =
        id.removePrefix("ink://ink").removePrefix("/texture:")
}
```

### Providing via CompositionLocal

```kotlin
import androidx.compose.runtime.CompositionLocalProvider
import androidx.compose.runtime.staticCompositionLocalOf

val LocalTextureStore = staticCompositionLocalOf<AppTextureBitmapStore> {
    error("No TextureStore provided")
}

// In your Activity or top-level Composable:
CompositionLocalProvider(LocalTextureStore provides textureStore) {
    DrawingScreen()
}
```

### Generation-Based Invalidation

When textures are loaded dynamically (e.g., custom brush textures from a database), increment the generation counter. Composables observing `generation` will recompose, recreating the `CanvasStrokeRenderer` and `InProgressStrokes`:

```kotlin
import androidx.compose.runtime.getValue
import androidx.compose.runtime.key
import androidx.compose.runtime.remember
import androidx.ink.authoring.compose.InProgressStrokes
import androidx.ink.rendering.android.canvas.CanvasStrokeRenderer
import androidx.lifecycle.compose.collectAsStateWithLifecycle

val cacheGen by textureStore.generation.collectAsStateWithLifecycle()

val renderer = remember(textureStore, cacheGen) {
    CanvasStrokeRenderer.create(textureStore)
}

key(cacheGen) {
    InProgressStrokes(
        defaultBrush = currentBrush,
        nextBrush = onGetNextBrush,
        onStrokesFinished = onStrokesFinished,
        textureBitmapStore = textureStore
    )
}
```

## Common Pitfalls

- **Calling `forcePathRendering = true` in export**: The 2-arg `CanvasStrokeRenderer.create(forcePathRendering, textureStore)` overload is `@RestrictTo(LIBRARY_GROUP)`. Use `CanvasStrokeRenderer.create(textureStore)`, which automatically falls back to path rendering on software `Bitmap` canvases.
- **Not normalizing texture IDs**: The Ink API may prefix texture IDs with `ink://ink/texture:`. Your `TextureBitmapStore` should strip these prefixes for consistent lookup.
- **Drag & drop blocking InProgressStrokes**: Without `pointerInputWithSiblingFallthrough`, the drag handler consumes all touch events, preventing any drawing.
- **Order of composable children in Box**: In Compose, the last child in a `Box` is hit-tested first and stops sibling hit-testing unless `sharePointerInputWithSiblings()` is `true`. Place the `pointerInputWithSiblingFallthrough` gesture interceptor *after* `InProgressStrokes` so both siblings receive pointer events.
- **TextureBitmapStore not being a Singleton**: If multiple instances exist, textures loaded in one won't be available in another. Use `@Singleton` scope.
- **Forgetting to increment generation on texture load**: Without the generation counter bump, the renderer and `InProgressStrokes` won't pick up newly loaded textures until the next recomposition from an unrelated state change.
