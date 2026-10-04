# Drawing Surface

Reference for building the Compose-based drawing surface that captures user input and renders strokes.

## Core Concept: Wet Ink vs. Dry Ink

The Ink API separates stroke rendering into two layers:

1. **Wet ink** — Strokes currently being drawn by the user, managed by the `InProgressStrokes` composable. This layer provides low-latency rendering using front-buffer rendering.
2. **Dry ink** — Completed strokes rendered in a Compose `Canvas`. These are the persisted `Stroke` objects.

Both layers are stacked inside a `Box` so that wet ink overlays the dry ink canvas.

## `InProgressStrokes` Composable

The `InProgressStrokes` composable (from `ink-authoring-compose`) handles touch/stylus input capture and real-time stroke rendering automatically.

### Key Parameters

| Parameter | Type | Description |
|---|---|---|
| `defaultBrush` | `Brush?` | The brush applied when a stroke begins (or `null` to disable wet-ink input while keeping internal rendering resources initialized) |
| `nextBrush` | `() -> Brush?` | Lambda called to get the brush for the *next* stroke (defaults to `{ defaultBrush }`; allows switching brushes between strokes) |
| `pointerEventToWorldTransform` | `Matrix` | `androidx.compose.ui.graphics.Matrix` mapping pointer input coordinates to world/stroke space (defaults to `IDENTITY_MATRIX`; note this is Compose's `Matrix`, whereas `CanvasStrokeRenderer.draw` takes `android.graphics.Matrix` or `AffineTransform`) |
| `strokeToWorldTransform` | `Matrix` | `androidx.compose.ui.graphics.Matrix` mapping stroke coordinates to world space (defaults to `IDENTITY_MATRIX`) |
| `maskPath` | `Path?` | Optional `androidx.compose.ui.graphics.Path` masking regions where wet ink should not render (e.g., under floating toolbars; defaults to `null`) |
| `textureBitmapStore` | `TextureBitmapStore` | Store for custom brush textures (e.g., emoji highlighters). Defaults to `TextureBitmapStore { null }` (non-nullable; do not pass `null`). |
| `onStrokesFinished` | `(List<Stroke>) -> Unit` | Callback delivering completed `Stroke` objects when the user lifts their finger/stylus |

### Usage

```kotlin
import android.graphics.Matrix
import androidx.compose.foundation.Canvas
import androidx.compose.foundation.gestures.detectDragGestures
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.getValue
import androidx.compose.runtime.key
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.nativeCanvas
import androidx.compose.ui.input.pointer.pointerInput
import androidx.core.graphics.withSave
import androidx.ink.authoring.compose.InProgressStrokes
import androidx.ink.brush.Brush
import androidx.ink.rendering.android.canvas.CanvasStrokeRenderer
import androidx.ink.strokes.Stroke
import androidx.lifecycle.compose.collectAsStateWithLifecycle

@Composable
fun DrawingSurface(
    strokes: List<Stroke>,
    canvasStrokeRenderer: CanvasStrokeRenderer,
    currentBrush: Brush,
    onGetNextBrush: () -> Brush,
    onStrokesFinished: (List<Stroke>) -> Unit,
    isEraserMode: Boolean,
    modifier: Modifier = Modifier,
    onErase: (Float, Float) -> Unit = { _, _ -> },
    onEraseStart: () -> Unit = {},
    onEraseEnd: () -> Unit = {},
) {
    val textureStore = LocalTextureStore.current
    val cacheGen by textureStore.generation.collectAsStateWithLifecycle()

    // Synchronous handoff buffer: holds newly finished strokes in Compose Snapshot State
    // for the single frame between InProgressStrokes removing the wet stroke and an
    // asynchronous StateFlow (collectAsStateWithLifecycle) delivering the updated `strokes` list.
    var pendingStrokes by remember { mutableStateOf<List<Stroke>>(emptyList()) }

    LaunchedEffect(strokes) {
        if (strokes.isEmpty()) {
            pendingStrokes = emptyList()
        } else if (pendingStrokes.isNotEmpty()) {
            pendingStrokes = pendingStrokes.filter { pending -> strokes.none { it === pending } }
        }
    }

    Box(modifier = modifier) {
        // Layer 1: Dry ink — completed strokes + same-frame pending handoff strokes
        Canvas(modifier = Modifier.fillMaxSize()) {
            val canvas = drawContext.canvas.nativeCanvas
            val strokesToDraw = if (pendingStrokes.isEmpty()) {
                strokes
            } else {
                strokes + pendingStrokes.filter { pending -> strokes.none { it === pending } }
            }
            strokesToDraw.forEach { stroke ->
                canvas.withSave {
                    canvasStrokeRenderer.draw(
                        canvas = this,
                        stroke = stroke,
                        strokeToScreenTransform = Matrix()
                    )
                }
            }
        }

        // Layer 2: Eraser touch handler OR Wet ink (InProgressStrokes)
        if (isEraserMode) {
            Box(
                modifier = Modifier
                    .fillMaxSize()
                    .pointerInput(Unit) {
                        detectDragGestures(
                            onDragStart = { onEraseStart() },
                            onDragEnd = { onEraseEnd() },
                            onDragCancel = { onEraseEnd() },
                            onDrag = { change, _ ->
                                onErase(change.position.x, change.position.y)
                                change.consume()
                            }
                        )
                    }
            )
        } else {
            key(cacheGen) {
                InProgressStrokes(
                    defaultBrush = currentBrush,
                    nextBrush = onGetNextBrush,
                    onStrokesFinished = { newStrokes ->
                        // Mutate Compose Snapshot State synchronously in the same UI thread
                        // run loop BEFORE InProgressStrokes removes the completed wet strokes.
                        pendingStrokes = pendingStrokes + newStrokes
                        onStrokesFinished(newStrokes)
                    },
                    textureBitmapStore = textureStore
                )
            }
        }
    }
}
```

## Box Layering Structure

The drawing surface uses a `Box` with stacked children:

```
Box {
    ├── (Optional) Background image layer
    ├── Canvas — dry ink rendering (completed strokes + same-frame pendingStrokes)
    ├── If eraser mode:
    │   └── Eraser touch handler (detectDragGestures)
    │   Else:
    │   └── InProgressStrokes (wet ink + input capture)
    └── (Optional) Gesture interceptor for drag & drop (pointerInputWithSiblingFallthrough)
}
```

### Why the separation matters

- `InProgressStrokes` uses front-buffer rendering for near-zero latency during active drawing.
- Completed strokes are re-rendered via `CanvasStrokeRenderer` in a standard Compose `Canvas`, which integrates with the Compose rendering pipeline and supports features like blend modes and save layers.

## `onStrokesFinished` & Preventing Wet-to-Dry Handoff Flicker

When the user lifts their finger/stylus, `InProgressStrokes` delivers completed `Stroke` objects through `onStrokesFinished` on the UI thread, and **immediately removes the wet strokes right after `onStrokesFinished` returns**:

```kotlin
// Inside InProgressShapes.kt (androidx.ink:ink-authoring-compose):
override fun onShapesCompleted(shapes: Map<InProgressStrokeId, CompletedShapeT>) {
    onShapesCompleted(shapes.values.toList())
    // Must recompose from callback shapes, cannot wait until a later frame.
    ipsv.removeCompletedShapes(shapes.keys)
}
```

### Why naive `StateFlow` collection flickers

If `onStrokesFinished` only updates a `MutableStateFlow` in your `ViewModel` and `DrawingSurface` observes `uiState.strokes` via `collectAsStateWithLifecycle()`:
1. `InProgressStrokes` removes the wet stroke immediately in frame $N$.
2. `collectAsStateWithLifecycle()` collects the `StateFlow` emission via a coroutine (`produceState`), which does **not** synchronously update Compose Snapshot State in the same UI thread run loop.
3. The dry `Canvas` does not receive the new stroke until frame $N+1$, causing a visible **1-frame flicker** where the stroke disappears on stylus/finger lift.

### Recommended Solution: Self-Contained `pendingStrokes` Buffer in `DrawingSurface`

Encapsulating a `var pendingStrokes by remember { mutableStateOf<List<Stroke>>(emptyList()) }` buffer inside `DrawingSurface` (as shown in the `DrawingSurface` implementation above) solves this cleanly:
- **Same-frame invalidation**: `pendingStrokes = pendingStrokes + newStrokes` mutates Compose Snapshot State synchronously inside `onStrokesFinished` before `removeCompletedShapes()` runs, invalidating only the `Canvas` draw scope in frame $N$.
- **Zero double-draw on translucent brushes**: Filtering by reference identity (`strokes.none { it === pending }`) ensures that as soon as `strokes` updates in frame $N+1$ (or even in frame $N$ if the caller uses synchronous state), each stroke is drawn **at most once** — preventing translucent highlighter strokes (`BlendMode.Multiply`) from flashing twice as dark.
- **Zero Undo/Redo/Erase lag & zero boilerplate**: On normal frames (Undo, Redo, Erase, idle), `pendingStrokes.isEmpty()` is `true`, so `Canvas` draws `strokes` directly without copying lists or waiting for a `LaunchedEffect` sync. Callers can pass `uiState.strokes` from `collectAsStateWithLifecycle()` directly.

### Alternative Solution: Local `mutableStateListOf` in the Screen (Cahier Pattern)

If you are working in an existing codebase where `DrawingSurface` renders `strokes` directly without an internal `pendingStrokes` buffer (such as the Cahier sample's `DrawingCanvas.kt`), the parent screen can maintain a `remember { mutableStateListOf<Stroke>() }`, call `localStrokes.addAll(newStrokes)` synchronously in `onStrokesFinished`, and sync external changes (`undo`, `redo`, `erase`) via `LaunchedEffect(uiState.strokes)`:

```kotlin
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
val localStrokes = remember { mutableStateListOf<Stroke>() }

LaunchedEffect(uiState.strokes) {
    if (localStrokes != uiState.strokes) {
        localStrokes.clear()
        localStrokes.addAll(uiState.strokes)
    }
}

DrawingSurface(
    strokes = localStrokes,
    onStrokesFinished = { newStrokes ->
        localStrokes.addAll(newStrokes) // Synchronous Compose snapshot update
        viewModel.onStrokesFinished(newStrokes)
    },
    // ...
)
```

> **Tip**: For new code, the self-contained `pendingStrokes` buffer inside `DrawingSurface` is preferred because it avoids full-list copies (`clear()` + `addAll()`), avoids a 1-frame `LaunchedEffect` delay on Undo/Redo/Erase, and requires zero boilerplate in parent screens.

### ViewModel Handler

In the `ViewModel`, update the history/state and offload disk persistence to a coroutine:

```kotlin
import androidx.ink.strokes.Stroke
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.launch

fun onStrokesFinished(finishedStrokes: List<Stroke>) {
    // Append to the existing stroke list
    val updatedStrokes = currentStrokes + finishedStrokes
    // Update state and persist asynchronously
    updateStrokes(updatedStrokes)
    viewModelScope.launch { saveStrokes() }
}
```

> **Important**: `onStrokesFinished` is called on the UI thread. Keep the handler lightweight — offload persistence to a coroutine.

## TextureBitmapStore for Custom Brushes

When using textured brushes (e.g., emoji highlighters), provide a `TextureBitmapStore` implementation:

```kotlin
import android.content.Context
import android.graphics.Bitmap
import androidx.ink.brush.TextureBitmapStore
import dagger.hilt.android.qualifiers.ApplicationContext
import javax.inject.Inject
import javax.inject.Singleton
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update

@Singleton
class AppTextureBitmapStore @Inject constructor(
    @ApplicationContext context: Context
) : TextureBitmapStore {

    private val loadedBitmaps = mutableMapOf<String, Bitmap>()

    private val _generation = MutableStateFlow(0)
    val generation: StateFlow<Int> = _generation.asStateFlow()

    override operator fun get(clientTextureId: String): Bitmap? {
        return loadedBitmaps[normalizeId(clientTextureId)]
    }

    fun loadTexture(textureId: String, bitmap: Bitmap) {
        loadedBitmaps[normalizeId(textureId)] = bitmap
        _generation.update { it + 1 }
    }

    private fun normalizeId(id: String): String =
        id.removePrefix("ink://ink").removePrefix("/texture:")
}
```

Provide it via `CompositionLocal` for easy access:

```kotlin
import androidx.compose.runtime.staticCompositionLocalOf

val LocalTextureStore = staticCompositionLocalOf<AppTextureBitmapStore> {
    error("No TextureStore provided")
}
```

## Invalidation with Texture Cache Generation

When textures are loaded dynamically, both `CanvasStrokeRenderer` (which caches `Paint` shaders per `BrushPaint` on first draw) and the `InProgressStrokes` composable must be recreated to pick up new textures. Use the `TextureBitmapStore` generation counter for both:

```kotlin
import androidx.compose.runtime.getValue
import androidx.compose.runtime.key
import androidx.compose.runtime.remember
import androidx.ink.authoring.compose.InProgressStrokes
import androidx.ink.rendering.android.canvas.CanvasStrokeRenderer
import androidx.lifecycle.compose.collectAsStateWithLifecycle

val cacheGen by textureStore.generation.collectAsStateWithLifecycle()
val canvasStrokeRenderer = remember(textureStore, cacheGen) {
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

- **Eraser and `InProgressStrokes` coexisting**: When eraser mode is active, you must remove `InProgressStrokes` from the composition (or guard it with a conditional). Having both active simultaneously causes input conflicts.
- **Forgetting `textureBitmapStore`**: Textured brushes (emoji highlighters, custom textures) will render as flat fills without a `TextureBitmapStore`.
- **Not invalidating `CanvasStrokeRenderer` and `InProgressStrokes` on texture generation changes**: If textures are loaded after initial composition, neither `InProgressStrokes` nor `CanvasStrokeRenderer` (which caches `BrushPaint` shaders) will pick them up until recreated. Use `remember(textureStore, cacheGen) { CanvasStrokeRenderer.create(textureStore) }` and `key(cacheGen) { InProgressStrokes(...) }`.
- **Wet-to-dry 1-frame flicker on stroke completion**: `InProgressStrokes` removes completed wet strokes immediately after `onStrokesFinished` returns in the same UI thread run loop. Relying solely on an asynchronous `StateFlow` (`collectAsStateWithLifecycle()`) without a synchronous Compose Snapshot State handoff (`pendingStrokes` inside `DrawingSurface` or a local `mutableStateListOf<Stroke>()`) causes a 1-frame gap where the stroke disappears.
- **Blocking in `onStrokesFinished`**: This callback runs on the main thread. Perform persistence in a `viewModelScope.launch {}`.
