# Eraser

Reference for implementing geometry-based stroke erasing using the Ink geometry API.

## Eraser Strategy: Intersection-Based Whole-Stroke Removal

The Ink API provides geometric primitives for detecting intersections between the eraser path and existing strokes. The eraser creates a `MutableParallelogram` from each segment of the drag gesture and tests it against each stroke's shape.

When a stroke's shape intersects the eraser parallelogram, the entire stroke is removed.

## Key Types

| Type | Module | Purpose |
|---|---|---|
| `MutableSegment` | `ink-geometry` | A line segment between two points |
| `MutableParallelogram` | `ink-geometry` | A padded parallelogram around a segment (the eraser's hit area) |
| `Intersection.intersects()` | `ink-geometry` | Tests whether a stroke's shape intersects a parallelogram |
| `AffineTransform` | `ink-geometry` | Transform applied during intersection testing |
| `MutableVec` | `ink-geometry` | A mutable 2D vector/point |

## Implementation

### ViewModel State

```kotlin
import androidx.ink.geometry.MutableVec

class DrawingViewModel : ViewModel() {
    // Track previous touch point for segment construction
    private var previousPoint: MutableVec? = null

    // Eraser width (padding around the drag segment)
    private val eraserPadding = 50f

    // Eraser mode toggle
    private val _isEraserMode = MutableStateFlow(false)
    val isEraserMode: StateFlow<Boolean> = _isEraserMode.asStateFlow()
}
```

### Eraser Lifecycle

```kotlin
fun startErase() {
    previousPoint = null
}

fun endErase() {
    previousPoint = null
    viewModelScope.launch { saveStrokes() }
}
```

- `startErase()` is called when the eraser drag begins (`onDragStart`).
- `endErase()` is called when the drag ends (`onDragEnd`). It resets state and persists the updated stroke list.

### Core Erase Logic

```kotlin
import androidx.ink.geometry.AffineTransform
import androidx.ink.geometry.Intersection.intersects
import androidx.ink.geometry.MutableParallelogram
import androidx.ink.geometry.MutableSegment
import androidx.ink.geometry.MutableVec
import androidx.ink.strokes.Stroke

fun erase(x: Float, y: Float) {
    val strokesBefore = currentStrokes
    val strokesAfter = eraseIntersectingStrokes(x, y, strokesBefore)

    if (strokesAfter.size != strokesBefore.size) {
        updateStrokes(strokesAfter)
    }
}

private fun eraseIntersectingStrokes(
    currentX: Float,
    currentY: Float,
    currentStrokes: List<Stroke>,
): List<Stroke> {
    val prev = previousPoint
    previousPoint = MutableVec(currentX, currentY)

    // Need at least two points to form a segment
    if (prev == null) return currentStrokes

    // Create a segment from previous point to current point
    val segment = MutableSegment(prev, MutableVec(currentX, currentY))

    // Expand segment into a parallelogram with padding
    val parallelogram = MutableParallelogram()
        .populateFromSegmentAndPadding(segment, eraserPadding)

    // Find all strokes that intersect the eraser area
    val strokesToRemove = currentStrokes.filter { stroke ->
        stroke.shape.intersects(parallelogram, AffineTransform.IDENTITY)
    }

    return if (strokesToRemove.isNotEmpty()) {
        currentStrokes - strokesToRemove.toSet()
    } else {
        currentStrokes
    }
}
```

### How `populateFromSegmentAndPadding` Works

```
        ┌──────────────────────────────┐
        │         padding              │
        │   ┌──────────────────────┐   │
        │   │  segment (prev→curr) │   │  ← parallelogram
        │   └──────────────────────┘   │
        │         padding              │
        └──────────────────────────────┘
```

The `eraserPadding` expands the segment into a parallelogram on all sides. Larger values make the eraser more forgiving (easier to erase strokes), smaller values require more precision.

## Compose Touch Handling for Eraser

Use `detectDragGestures` on a `Box` overlay when in eraser mode:

```kotlin
import androidx.compose.foundation.gestures.detectDragGestures
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.ui.Modifier
import androidx.compose.ui.input.pointer.pointerInput

if (isEraserMode) {
    Box(
        modifier = Modifier
            .fillMaxSize()
            .pointerInput(Unit) {
                detectDragGestures(
                    onDragStart = { onEraseStart() },
                    onDragEnd = { onEraseEnd() },
                    onDragCancel = { onEraseEnd() }
                ) { change, _ ->
                    onErase(change.position.x, change.position.y)
                    change.consume()
                }
            }
    )
}
```

### Integration with DrawingSurface

The eraser overlay replaces `InProgressStrokes` when active:

```kotlin
import androidx.compose.foundation.Canvas
import androidx.compose.foundation.gestures.detectDragGestures
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.ui.Modifier
import androidx.compose.ui.input.pointer.pointerInput
import androidx.ink.authoring.compose.InProgressStrokes

Box(modifier = modifier) {
    // Dry ink canvas (always rendered)
    Canvas(modifier = Modifier.fillMaxSize()) { /* render strokes */ }

    if (isEraserMode) {
        // Eraser touch handler
        Box(modifier = Modifier.fillMaxSize().pointerInput(Unit) {
            detectDragGestures(
                onDragStart = { viewModel.startErase() },
                onDragEnd = { viewModel.endErase() },
                onDragCancel = { viewModel.endErase() }
            ) { change, _ ->
                viewModel.erase(change.position.x, change.position.y)
                change.consume()
            }
        })
    } else {
        // Wet ink input (InProgressStrokes)
        InProgressStrokes(/* ... */)
    }
}
```

## Eraser Mode Toggle

```kotlin
fun setEraserMode(enabled: Boolean) {
    _isEraserMode.update { enabled }
}
```

When switching to eraser mode, the brush selection UI should visually indicate the active tool. When switching back to a brush, call `setEraserMode(false)`.

## Tuning Eraser Padding

| Padding Value | Behavior |
|---|---|
| `10f–30f` | Precise eraser — must touch the stroke closely |
| `40f–60f` | Balanced — good default for finger/stylus |
| `80f+` | Large eraser — erases strokes in a wide area |

## Compose & Android Graphics Geometry Conversions (`ink-geometry-compose` / `ink-geometry`)

The `ink-geometry-compose` (`androidx.ink.geometry.compose.*`) and `ink-geometry` (`androidx.ink.geometry.*`) modules provide extension functions to convert between Compose/Android graphics types and Ink geometry types — useful for converting `PointerInputChange.position` (`Offset`), transforming eraser hit tests under pan/zoom (`Matrix` → `AffineTransform`), and computing stroke bounding boxes (`Box` → `Rect` / `RectF`):

```kotlin
import android.graphics.Matrix as AndroidMatrix
import android.graphics.RectF
import androidx.compose.ui.geometry.Offset
import androidx.compose.ui.geometry.Rect
import androidx.compose.ui.graphics.Matrix as ComposeMatrix
import androidx.ink.geometry.AffineTransform
import androidx.ink.geometry.ImmutableAffineTransform
import androidx.ink.geometry.Intersection.intersects
import androidx.ink.geometry.MutableAffineTransform
import androidx.ink.geometry.MutableParallelogram
import androidx.ink.geometry.MutableVec
import androidx.ink.geometry.compose.from
import androidx.ink.geometry.compose.populateFrom
import androidx.ink.geometry.compose.toOffset
import androidx.ink.geometry.compose.toRect
import androidx.ink.geometry.from
import androidx.ink.geometry.populateFrom
import androidx.ink.geometry.toRectF
import androidx.ink.strokes.Stroke

// 1. Offset ↔ Vec (ink-geometry-compose)
fun populatePointFromOffset(reusableVec: MutableVec, offset: Offset): Offset {
    reusableVec.populateFrom(offset) // In-place without allocation
    return reusableVec.toOffset()
}

// 2. Erasing with Pan / Zoom Transform (Matrix → AffineTransform)
fun eraseWithPanZoom(
    strokes: List<Stroke>,
    screenEraserBox: MutableParallelogram,
    worldToScreenAndroidMatrix: AndroidMatrix,
    worldToScreenComposeMatrix: ComposeMatrix? = null,
): List<Stroke> {
    // Convert Android Matrix (or Compose Matrix) to AffineTransform
    val meshToEraserTransform: AffineTransform =
        if (worldToScreenComposeMatrix != null) {
            ImmutableAffineTransform.from(worldToScreenComposeMatrix) ?: AffineTransform.IDENTITY
        } else {
            MutableAffineTransform().populateFrom(worldToScreenAndroidMatrix)
        }

    return strokes.filterNot { stroke ->
        stroke.shape.intersects(screenEraserBox, meshToEraserTransform)
    }
}

// 3. Stroke Bounding Box → Compose Rect / Android RectF
fun getStrokeBounds(stroke: Stroke): Pair<Rect?, RectF?> {
    val boundingBox = stroke.shape.computeBoundingBox() ?: return null to null
    return boundingBox.toRect() to boundingBox.toRectF()
}
```

## Common Pitfalls

- **Forgetting to track `previousPoint`**: The eraser requires *two* consecutive points to form a segment. If `previousPoint` is not tracked, no intersection testing occurs.
- **Not resetting `previousPoint` in `startErase()`/`endErase()`**: Failing to reset causes the eraser to create a segment from the end of the last erase gesture to the start of the new one, potentially erasing unintended strokes.
- **Erasing on every touch point without change detection**: Always check `if (strokesAfter.size != strokesBefore.size)` before updating state. Unnecessary updates trigger recomposition.
- **Not calling `change.consume()`**: If the pointer change is not consumed, touch events may propagate to siblings and cause unintended behavior.
- **Using `AffineTransform.IDENTITY` incorrectly with pan/zoom**: In `stroke.shape.intersects(parallelogram, meshToParallelogram)`, the second parameter maps from the `PartitionedMesh`'s coordinate space (stroke/world space) to the `Parallelogram`'s coordinate space. Either transform pointer coordinates (`change.position`) into world/stroke space first (using `pointerEventToWorldTransform`) and pass `AffineTransform.IDENTITY`, or keep `parallelogram` in screen space and pass `meshToParallelogram = MutableAffineTransform().populateFrom(worldToScreenMatrix)` (or `ImmutableAffineTransform.from(worldToScreenMatrix)`).
- **Not integrating with undo/redo**: Erasing should go through `updateStrokes()` to participate in the undo/redo history. See `undo-redo.md`.
