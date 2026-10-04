# Ink App Builder — Eval Results

**Date**: 2026-06-10  
**Skill**: `ink-app-builder`  
**Total Evals**: 10  

---

## Summary

| Eval | Title | Score | Notes |
|------|-------|-------|-------|
| 1 | Setup Dependencies | PASS | All artifacts present with version catalog pattern |
| 2 | Basic Drawing Surface | PASS | Full wet/dry ink architecture with Box layering |
| 3 | Brush Creation | PASS | All required patterns present |
| 4 | Eraser Implementation | PASS | Full geometry-based intersection eraser |
| 5 | Undo/Redo | PASS | Snapshot-based history stack |
| 6 | Stroke Persistence | PASS | ink-storage + Room + ByteArray pattern |
| 7 | Stroke Rendering | PASS | CanvasStrokeRenderer with withSave |
| 8 | Brush Switching | PASS | copy() pattern with alpha adjustment |
| 9 | Highlighter with Blend Mode | PASS | BlendMode.Multiply + withSaveLayer + alpha |
| 10 | Bitmap Export | PASS | createBitmap + Canvas(bitmap) + CanvasStrokeRenderer + compress |

**Overall: 10/10 PASS, 0 PARTIAL, 0 FAIL**

---

## Eval 1: Setup Dependencies

**Prompt**: "Add the Android Ink API dependencies to my Compose project"

**Reference docs consulted**: `setup.md`

### Generated Response

Here's how to add the Android Ink API dependencies to your Compose project:

**1. Add the version catalog entries** (`gradle/libs.versions.toml`):

```toml
[versions]
ink = "<latest_version>"
inputMotionPrediction = "<latest_version>"

[libraries]
# Core Ink modules
androidx-ink-authoring       = { module = "androidx.ink:ink-authoring",    version.ref = "ink" }
androidx-ink-brush           = { module = "androidx.ink:ink-brush",        version.ref = "ink" }
androidx-ink-geometry        = { module = "androidx.ink:ink-geometry",     version.ref = "ink" }
androidx-ink-nativeloader    = { module = "androidx.ink:ink-nativeloader", version.ref = "ink" }
androidx-ink-rendering       = { module = "androidx.ink:ink-rendering",    version.ref = "ink" }
androidx-ink-strokes         = { module = "androidx.ink:ink-strokes",      version.ref = "ink" }
androidx-ink-storage         = { module = "androidx.ink:ink-storage",      version.ref = "ink" }

# Compose integration modules
androidx-ink-authoring-android = { group = "androidx.ink", name = "ink-authoring-android", version.ref = "ink" }
androidx-ink-authoring-compose = { group = "androidx.ink", name = "ink-authoring-compose", version.ref = "ink" }
androidx-ink-brush-compose     = { group = "androidx.ink", name = "ink-brush-compose",     version.ref = "ink" }
androidx-ink-geometry-compose  = { group = "androidx.ink", name = "ink-geometry-compose",   version.ref = "ink" }

# Motion prediction (optional but recommended for stylus input)
androidx-input-motionprediction = { module = "androidx.input:input-motionprediction", version.ref = "inputMotionPrediction" }
```

**2. Add the dependencies** (`app/build.gradle.kts`):

```kotlin
dependencies {
    // Ink core
    implementation(libs.androidx.ink.authoring)
    implementation(libs.androidx.ink.brush)
    implementation(libs.androidx.ink.geometry)
    implementation(libs.androidx.ink.nativeloader)
    implementation(libs.androidx.ink.rendering)
    implementation(libs.androidx.ink.strokes)
    implementation(libs.androidx.ink.storage)

    // Ink Compose integration
    implementation(libs.androidx.ink.authoring.android)
    implementation(libs.androidx.ink.authoring.compose)
    implementation(libs.androidx.ink.brush.compose)
    implementation(libs.androidx.ink.geometry.compose)

    // Motion prediction (optional)
    implementation(libs.androidx.input.motionprediction)
}
```

**3. Ensure your `app/build.gradle.kts` has the correct min SDK and Compose enabled**:

```kotlin
android {
    defaultConfig {
        minSdk = 26
    }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    buildFeatures {
        compose = true
    }
}
```

> **Important**: All `ink-*` artifacts must use the same version. The version catalog with a single `ink` version ref prevents version drift. Also, don't forget `ink-nativeloader` (without it, the rendering engine fails at runtime with `UnsatisfiedLinkError`) and `ink-authoring-android` (required at runtime by `ink-authoring-compose`).

### Self-Evaluation

**Must Include checklist**:
- ✅ `ink-authoring` and `ink-authoring-android` — present in version catalog and build.gradle
- ✅ `ink-authoring-compose` (not `ink-authoring` alone) — present in version catalog and build.gradle
- ✅ `ink-brush` and `ink-brush-compose` — present
- ✅ `ink-geometry` and `ink-geometry-compose` — present
- ✅ `ink-rendering` — present
- ✅ `ink-strokes` — present
- ✅ `ink-storage` — present
- ✅ `ink-nativeloader` — present
- ✅ Version catalog pattern or single version variable — uses `libs.versions.toml` with `version.ref = "ink"`

**Must NOT Include checklist**:
- ✅ ❌ Hardcoded version numbers like `1.0.0-alpha01` — correctly absent (uses `<latest_version>`)
- ✅ ❌ `InProgressStrokesView` (View-based surface) — correctly absent

**Score**: **PASS**

---

## Eval 2: Basic Drawing Surface

**Prompt**: "Create a Compose screen where users can draw with their finger or stylus"

**Reference docs consulted**: `drawing-surface.md`, `stroke-rendering.md`, `stock-brushes.md`

### Generated Response

```kotlin
import android.graphics.Matrix
import androidx.compose.foundation.Canvas
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.nativeCanvas
import androidx.core.graphics.withSave
import androidx.ink.authoring.compose.InProgressStrokes
import androidx.ink.brush.Brush
import androidx.ink.brush.StockBrushes
import androidx.ink.brush.compose.createWithComposeColor
import androidx.ink.rendering.android.canvas.CanvasStrokeRenderer
import androidx.ink.strokes.Stroke

@Composable
fun DrawingScreen() {
    // Create the stroke renderer
    val canvasStrokeRenderer = remember { CanvasStrokeRenderer.create() }

    // Track completed strokes
    var strokes by remember { mutableStateOf(listOf<Stroke>()) }

    // Create a default brush
    val currentBrush = remember {
        Brush.createWithComposeColor(
            family = StockBrushes.pressurePen(),
            color = Color.Black,
            size = 5f,
            epsilon = 0.1f
        )
    }

    Box(modifier = Modifier.fillMaxSize()) {
        // Layer 1: Dry ink — render completed strokes
        Canvas(modifier = Modifier.fillMaxSize()) {
            val canvas = drawContext.canvas.nativeCanvas
            strokes.forEach { stroke ->
                canvas.withSave {
                    canvasStrokeRenderer.draw(
                        canvas = this,
                        stroke = stroke,
                        strokeToScreenTransform = Matrix()
                    )
                }
            }
        }

        // Layer 2: Wet ink — in-progress strokes with live input
        InProgressStrokes(
            defaultBrush = currentBrush,
            nextBrush = { currentBrush },
            onStrokesFinished = { finishedStrokes ->
                strokes = strokes + finishedStrokes
            }
        )
    }
}
```

This uses a two-layer approach:
1. **Dry ink** (`Canvas`) — renders completed `Stroke` objects via `CanvasStrokeRenderer.draw()`
2. **Wet ink** (`InProgressStrokes`) — captures touch/stylus input and renders strokes in real-time with ultra-low latency using front-buffer rendering

The `onStrokesFinished` callback delivers completed strokes when the user lifts their finger/stylus. They're appended to the strokes list and rendered on the Canvas.

### Self-Evaluation

**Must Include checklist**:
- ✅ `InProgressStrokes` composable — present
- ✅ `CanvasStrokeRenderer.create()` — present
- ✅ `Canvas` composable for dry ink rendering — present
- ✅ `canvasStrokeRenderer.draw(canvas, stroke, Matrix())` — present
- ✅ `Brush.createWithComposeColor(family, color, size, epsilon)` — present
- ✅ `onStrokesFinished` callback to capture completed strokes — present
- ✅ `Box` layering: Canvas underneath, InProgressStrokes on top — present

**Must NOT Include checklist**:
- ✅ ❌ `InProgressStrokesView` — correctly absent
- ✅ ❌ `View.OnTouchListener` — correctly absent
- ✅ ❌ `MotionEvent` handling — correctly absent
- ✅ ❌ `Canvas.drawPath()` or manual point iteration — correctly absent

**Score**: **PASS**

---

## Eval 3: Brush Creation

**Prompt**: "Create a Brush using a stock pen brush family"

**Reference docs consulted**: `stock-brushes.md`

### Generated Response

```kotlin
import androidx.ink.brush.Brush
import androidx.ink.brush.StockBrushes
import androidx.ink.brush.compose.createWithComposeColor
import androidx.compose.ui.graphics.Color

val brush = Brush.createWithComposeColor(
    family = StockBrushes.pressurePen(),
    color = Color.Black,
    size = 5f,
    epsilon = 0.1f
)
```

**Parameters explained**:
- `family` — `StockBrushes.pressurePen()` returns a `BrushFamily` for a pressure-sensitive pen with variable width
- `color` — A Compose `Color` from `androidx.compose.ui.graphics.Color`
- `size` — Stroke width in pixels
- `epsilon` — Geometric approximation tolerance. Smaller values (e.g., `0.01f`) give higher fidelity curves but use more memory. `0.1f` is a good default for general drawing.

> **Note**: `Brush.createWithComposeColor()` is an extension function from the `ink-brush-compose` module. Make sure that dependency is included in your project. Brushes are immutable — to change properties, use `copy()` or `copyWithComposeColor()`.

### Self-Evaluation

**Must Include checklist**:
- ✅ `StockBrushes.pressurePen()` — present
- ✅ `Brush.createWithComposeColor(family, color, size, epsilon)` — present
- ✅ `Color` from `androidx.compose.ui.graphics` — present (`import androidx.compose.ui.graphics.Color`)
- ✅ `epsilon` parameter with explanation — present with detailed explanation

**Must NOT Include checklist**:
- ✅ ❌ `Brush()` direct constructor — correctly absent
- ✅ ❌ `Color.argb()` (Android framework color) — correctly absent
- ✅ ❌ Missing `epsilon` — `epsilon` is present

**Score**: **PASS**

---

## Eval 4: Eraser Implementation

**Prompt**: "Implement an eraser tool that removes strokes the user draws over"

**Reference docs consulted**: `eraser.md`

### Generated Response

The Ink API provides a geometry-based eraser that detects intersections between the eraser path and existing strokes, then removes entire strokes that intersect.

**ViewModel eraser state and logic**:

```kotlin
import androidx.ink.geometry.AffineTransform
import androidx.ink.geometry.Intersection.intersects
import androidx.ink.geometry.MutableParallelogram
import androidx.ink.geometry.MutableSegment
import androidx.ink.geometry.MutableVec
import androidx.ink.strokes.Stroke
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update
import kotlinx.coroutines.launch

class DrawingViewModel : ViewModel() {
    // Track previous touch point for segment construction
    private var previousPoint: MutableVec? = null
    private val eraserPadding = 50f

    private val _isEraserMode = MutableStateFlow(false)
    val isEraserMode: StateFlow<Boolean> = _isEraserMode.asStateFlow()

    fun setEraserMode(enabled: Boolean) {
        _isEraserMode.update { enabled }
    }

    fun startErase() {
        previousPoint = null
    }

    fun endErase() {
        previousPoint = null
        viewModelScope.launch { saveStrokes() }
    }

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
}
```

**Compose eraser touch handler** — use `detectDragGestures` on a `Box` overlay when in eraser mode:

```kotlin
import androidx.compose.foundation.Canvas
import androidx.compose.foundation.gestures.detectDragGestures
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.ui.Modifier
import androidx.compose.ui.input.pointer.pointerInput
import androidx.ink.authoring.compose.InProgressStrokes

// In your DrawingSurface composable:
Box(modifier = modifier) {
    // Dry ink canvas (always rendered)
    Canvas(modifier = Modifier.fillMaxSize()) { /* render strokes */ }

    if (isEraserMode) {
        // Eraser touch handler
        Box(modifier = Modifier
            .fillMaxSize()
            .pointerInput(Unit) {
                detectDragGestures(
                    onDragStart = { viewModel.startErase() },
                    onDragEnd = { viewModel.endErase() },
                    onDragCancel = { viewModel.endErase() }
                ) { change, _ ->
                    viewModel.erase(change.position.x, change.position.y)
                    change.consume()
                }
            }
        )
    } else {
        // Wet ink input (InProgressStrokes)
        InProgressStrokes(/* ... */)
    }
}
```

### Self-Evaluation

**Must Include checklist**:
- ✅ `MutableSegment` for eraser path — present
- ✅ `MutableParallelogram` with `populateFromSegmentAndPadding` — present
- ✅ `stroke.shape.intersects(parallelogram, AffineTransform.IDENTITY)` — present
- ✅ Previous point tracking — present (`previousPoint: MutableVec?`)
- ✅ `detectDragGestures` for eraser input — present
- ✅ Filtering strokes list to remove intersecting strokes — present (`currentStrokes - strokesToRemove.toSet()`)

**Must NOT Include checklist**:
- ✅ ❌ Drawing white strokes over existing ones — correctly absent
- ✅ ❌ Bitmap masking — correctly absent
- ✅ ❌ Per-pixel comparison — correctly absent
- ✅ ❌ `Canvas.clipPath()` — correctly absent

**Score**: **PASS**

---

## Eval 5: Undo/Redo

**Prompt**: "Add undo and redo to my drawing app"

**Reference docs consulted**: `undo-redo.md`

### Generated Response

The Ink API undo/redo uses a snapshot-based history stack where each entry is a complete list of strokes at that point in time.

```kotlin
import androidx.ink.strokes.Stroke
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update
import kotlinx.coroutines.launch

class DrawingViewModel : ViewModel() {
    // Full history of stroke states
    private val history = mutableListOf<List<Stroke>>()

    // Current position in history (-1 = no history)
    private var historyIndex = -1

    // Observable undo/redo availability
    private val _canUndo = MutableStateFlow(false)
    val canUndo: StateFlow<Boolean> = _canUndo.asStateFlow()

    private val _canRedo = MutableStateFlow(false)
    val canRedo: StateFlow<Boolean> = _canRedo.asStateFlow()

    init {
        viewModelScope.launch {
            val initialStrokes = repository.loadStrokes(documentId)
            _uiState.update { it.copy(strokes = initialStrokes) }

            // Seed history with the initial state
            history.clear()
            history.add(initialStrokes)
            historyIndex = 0
            updateUndoRedoState()
        }
    }

    // Record a new state whenever strokes change
    private fun updateStrokes(newStrokes: List<Stroke>) {
        // Truncate any "future" history beyond current index
        if (historyIndex < history.size - 1) {
            history.subList(historyIndex + 1, history.size).clear()
        }

        // Append the new state and advance the index
        history.add(newStrokes)
        historyIndex++

        // Update UI state
        _uiState.update { it.copy(strokes = newStrokes) }
        updateUndoRedoState()
    }

    private fun updateUndoRedoState() {
        _canUndo.value = historyIndex > 0
        _canRedo.value = historyIndex < history.size - 1
    }

    fun undo() {
        if (canUndo.value) {
            historyIndex--
            _uiState.update { it.copy(strokes = history[historyIndex]) }
            updateUndoRedoState()
            viewModelScope.launch { saveStrokes() }
        }
    }

    fun redo() {
        if (canRedo.value) {
            historyIndex++
            _uiState.update { it.copy(strokes = history[historyIndex]) }
            updateUndoRedoState()
            viewModelScope.launch { saveStrokes() }
        }
    }

    // Called when user finishes drawing new strokes
    fun onStrokesFinished(finishedStrokes: List<Stroke>) {
        val current = history.getOrElse(historyIndex) { emptyList() }
        val newStrokes = current + finishedStrokes
        updateStrokes(newStrokes)
        viewModelScope.launch { saveStrokes() }
    }
}
```

And in Compose UI:

```kotlin
import androidx.compose.material3.Icon
import androidx.compose.material3.IconButton
import androidx.compose.runtime.getValue
import androidx.compose.ui.res.painterResource
import androidx.lifecycle.compose.collectAsStateWithLifecycle

val canUndo by viewModel.canUndo.collectAsStateWithLifecycle()
val canRedo by viewModel.canRedo.collectAsStateWithLifecycle()

IconButton(onClick = { viewModel.undo() }, enabled = canUndo) {
    Icon(painter = painterResource(R.drawable.undo), contentDescription = "Undo")
}

IconButton(onClick = { viewModel.redo() }, enabled = canRedo) {
    Icon(painter = painterResource(R.drawable.redo), contentDescription = "Redo")
}
```

Key behaviors:
- When the user makes a new edit after undoing, all "future" entries are discarded (standard linear undo model)
- Strokes are immutable, so history entries share `Stroke` instances — only the list allocation is duplicated
- Both `undo()` and `redo()` persist the current state

### Self-Evaluation

**Must Include checklist**:
- ✅ `mutableListOf<List<Stroke>>()` history stack — present
- ✅ `historyIndex` integer tracker — present
- ✅ Truncating future states on new stroke — present (`history.subList(historyIndex + 1, history.size).clear()`)
- ✅ `canUndo` / `canRedo` StateFlow or State — present as `StateFlow<Boolean>`
- ✅ Updating UI state after undo/redo — present (`_uiState.update`)

**Must NOT Include checklist**:
- ✅ ❌ Command pattern with separate command classes — correctly absent
- ✅ ❌ Storing bitmaps for each state — correctly absent
- ✅ ❌ Cloning strokes (they're immutable) — correctly absent (explicitly noted strokes are shared)

**Score**: **PASS**

---

## Eval 6: Stroke Persistence

**Prompt**: "Save my drawing strokes to a Room database so they persist across app launches"

**Reference docs consulted**: `persistence.md`

### Generated Response

The Ink API provides serialization via the `ink-storage` module. Here's how to persist strokes to Room:

**1. Add the `ink-storage` dependency** (if not already):
```kotlin
implementation(libs.androidx.ink.storage)
```

**2. Define serialization data classes**:

```kotlin
import kotlinx.serialization.Serializable

@Serializable
data class SerializedStroke(
    val inputs: ByteArray,          // Encoded StrokeInputBatch
    val brush: SerializedBrush
)

@Serializable
data class SerializedBrush(
    val size: Float,
    val color: Long,
    val epsilon: Float,
    val stockBrush: SerializedStockBrush,
    val clientBrushFamilyId: String? = null,
)

@Serializable
enum class SerializedStockBrush {
    MarkerLatest, PressurePenLatest, HighlighterLatest, DashedLineLatest
}
```

**3. Encode and decode strokes**:

```kotlin
import androidx.ink.brush.Brush
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.StockBrushes
import androidx.ink.storage.encode
import androidx.ink.storage.decode
import androidx.ink.strokes.Stroke
import androidx.ink.strokes.StrokeInputBatch
import java.io.ByteArrayOutputStream
import java.io.ByteArrayInputStream
import kotlinx.serialization.encodeToString
import kotlinx.serialization.json.Json

private val stockBrushToEnum = mapOf(
    StockBrushes.marker()      to SerializedStockBrush.MarkerLatest,
    StockBrushes.pressurePen() to SerializedStockBrush.PressurePenLatest,
    StockBrushes.highlighter() to SerializedStockBrush.HighlighterLatest,
    StockBrushes.dashedLine()  to SerializedStockBrush.DashedLineLatest,
)

private val enumToStockBrush = stockBrushToEnum.entries.associate { (k, v) -> v to k }

fun serializeBrush(
    brush: Brush,
    customBrushes: Map<String, BrushFamily> = emptyMap(),
): SerializedBrush {
    val customBrushId = customBrushes.entries.firstOrNull { it.value == brush.family }?.key
    return SerializedBrush(
        size = brush.size,
        color = brush.colorLong,
        epsilon = brush.epsilon,
        stockBrush = stockBrushToEnum[brush.family] ?: SerializedStockBrush.MarkerLatest,
        clientBrushFamilyId = customBrushId,
    )
}

fun deserializeBrush(
    serialized: SerializedBrush,
    customBrushes: Map<String, BrushFamily> = emptyMap(),
): Brush {
    val family = serialized.clientBrushFamilyId?.let { customBrushes[it] }
        ?: enumToStockBrush[serialized.stockBrush]
        ?: StockBrushes.marker()
    return Brush.createWithColorLong(
        family = family,
        colorLong = serialized.color,
        size = serialized.size,
        epsilon = serialized.epsilon,
    )
}

fun serializeStroke(
    stroke: Stroke,
    customBrushes: Map<String, BrushFamily> = emptyMap(),
): String {
    val encodedInputs = ByteArrayOutputStream().use { outputStream ->
        stroke.inputs.encode(outputStream)
        outputStream.toByteArray()
    }
    val serializedStroke = SerializedStroke(
        inputs = encodedInputs,
        brush = serializeBrush(stroke.brush, customBrushes)
    )
    return Json.encodeToString(serializedStroke)
}

fun deserializeStroke(
    data: String,
    customBrushes: Map<String, BrushFamily> = emptyMap(),
): Stroke? {
    return try {
        val serialized = Json.decodeFromString<SerializedStroke>(data)
        val inputs = ByteArrayInputStream(serialized.inputs).use { inputStream ->
            StrokeInputBatch.decode(inputStream)
        }
        val brush = deserializeBrush(serialized.brush, customBrushes)
        Stroke(brush = brush, inputs = inputs)
    } catch (e: Exception) {
        null
    }
}
```

**4. Room entity**:

```kotlin
import androidx.room.ColumnInfo
import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity(tableName = "documents")
data class DocumentEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,
    @ColumnInfo(name = "strokes_data")
    val strokesData: List<String> = emptyList(), // Each entry is a serialized Stroke JSON string
)
```

**5. Room TypeConverter**:

```kotlin
import androidx.room.TypeConverter
import kotlinx.serialization.encodeToString
import kotlinx.serialization.json.Json

class Converters {
    private val json = Json { ignoreUnknownKeys = true }

    @TypeConverter
    fun fromStringList(list: List<String>?): String? {
        return list?.let { json.encodeToString(it) }
    }

    @TypeConverter
    fun toStringList(jsonString: String?): List<String> {
        if (jsonString.isNullOrEmpty()) return emptyList()
        return try {
            json.decodeFromString<List<String>>(jsonString)
        } catch (e: Exception) {
            emptyList()
        }
    }
}
```

**6. Repository**:

```kotlin
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext

class StrokeRepositoryImpl(
    private val dao: DocumentDao,
) : StrokeRepository {
    override suspend fun saveStrokes(documentId: Long, strokes: List<Stroke>) {
        withContext(Dispatchers.IO) {
            val serializedList = strokes.map { serializeStroke(it) }
            val doc = dao.getById(documentId)
            doc?.let { dao.update(it.copy(strokesData = serializedList)) }
        }
    }

    override suspend fun loadStrokes(documentId: Long): List<Stroke> =
        withContext(Dispatchers.IO) {
            val doc = dao.getById(documentId) ?: return@withContext emptyList()
            doc.strokesData.mapNotNull { deserializeStroke(it) }
        }
}
```

**7. Load on ViewModel init**:

```kotlin
init {
    viewModelScope.launch {
        val initialStrokes = repository.loadStrokes(documentId)
        _uiState.update { it.copy(strokes = initialStrokes) }

        history.clear()
        history.add(initialStrokes)
        historyIndex = 0
        updateUndoRedoState()
    }
}
```

### Self-Evaluation

**Must Include checklist**:
- ✅ Serialization using `ink-storage` module — present (`encode`/`decode` from `ink-storage`)
- ✅ Room entity storing serialized stroke data — present (`DocumentEntity` with `strokesData: List<String>` containing encoded `ByteArray` stroke inputs)
- ✅ Room TypeConverter for Stroke list serialization — present (`Converters` class with `@TypeConverter`)
- ✅ Loading strokes on ViewModel init — present

**Must NOT Include checklist**:
- ✅ ❌ Bitmap storage — correctly absent
- ✅ ❌ JSON serialization of stroke points — correctly absent (uses `ink-storage` encode/decode for binary serialization of `StrokeInputBatch`)
- ✅ ❌ SharedPreferences — correctly absent

**Score**: **PASS**

---

## Eval 7: Stroke Rendering

**Prompt**: "Render a list of finished strokes on a Compose Canvas"

**Reference docs consulted**: `stroke-rendering.md`

### Generated Response

```kotlin
import android.graphics.Matrix
import androidx.compose.foundation.Canvas
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.runtime.Composable
import androidx.compose.runtime.remember
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.nativeCanvas
import androidx.core.graphics.withSave
import androidx.ink.rendering.android.canvas.CanvasStrokeRenderer
import androidx.ink.strokes.Stroke

@Composable
fun StrokeCanvas(
    strokes: List<Stroke>,
    modifier: Modifier = Modifier,
) {
    val canvasStrokeRenderer = remember { CanvasStrokeRenderer.create() }

    Canvas(modifier = modifier.fillMaxSize()) {
        val canvas = drawContext.canvas.nativeCanvas
        strokes.forEach { stroke ->
            canvas.withSave {
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

Key points:
- `CanvasStrokeRenderer.create()` creates the renderer from the `ink-rendering` module
- Access the native Android `Canvas` via `drawContext.canvas.nativeCanvas`
- Wrap each `draw()` call in `canvas.withSave { }` to prevent transform/clip leakage between strokes
- `Matrix()` (identity) maps stroke coordinates 1:1 to screen coordinates
- Do NOT use Compose `DrawScope` methods to render strokes — the Ink renderer requires a native Android `Canvas`

### Self-Evaluation

**Must Include checklist**:
- ✅ `CanvasStrokeRenderer.create()` — present
- ✅ `Canvas(modifier)` composable — present
- ✅ `drawContext.canvas.nativeCanvas` access — present
- ✅ `canvasStrokeRenderer.draw(canvas, stroke, Matrix())` — present
- ✅ `canvas.withSave { }` block — present (`canvas.withSave { }`)

**Must NOT Include checklist**:
- ✅ ❌ Iterating stroke input points and drawing lines — correctly absent
- ✅ ❌ `Path.moveTo()` / `Path.lineTo()` — correctly absent
- ✅ ❌ `drawPath()` calls — correctly absent

**Score**: **PASS**

---

## Eval 8: Brush Switching

**Prompt**: "Let users switch between pen, marker, and highlighter brushes"

**Reference docs consulted**: `stock-brushes.md`

### Generated Response

```kotlin
import androidx.compose.ui.graphics.Color
import androidx.ink.brush.Brush
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.StockBrushes
import androidx.ink.brush.compose.composeColor
import androidx.ink.brush.compose.copyWithComposeColor
import androidx.ink.brush.compose.createWithComposeColor
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow

private const val HIGHLIGHTER_ALPHA = 0.3f

// Available brush families
val brushFamilies = listOf(
    StockBrushes.pressurePen(),
    StockBrushes.marker(),
    StockBrushes.highlighter(),
)

// In ViewModel:
private val _selectedBrush = MutableStateFlow(
    Brush.createWithComposeColor(
        family = StockBrushes.pressurePen(),
        color = Color.Black,
        size = 5f,
        epsilon = 0.1f
    )
)
val currentBrush: StateFlow<Brush> = _selectedBrush.asStateFlow()

fun changeBrush(brushFamily: BrushFamily) {
    val currentColor = _selectedBrush.value.composeColor

    val colorToApply = when (brushFamily) {
        StockBrushes.highlighter() ->
            currentColor.copy(alpha = HIGHLIGHTER_ALPHA)
        else ->
            currentColor.copy(alpha = 1f)
    }

    _selectedBrush.value = _selectedBrush.value
        .copy(family = brushFamily)
        .copyWithComposeColor(colorToApply)
}
```

In the Compose UI, provide buttons for each brush:

```kotlin
import androidx.compose.foundation.layout.Row
import androidx.compose.material3.Icon
import androidx.compose.material3.IconButton
import androidx.compose.runtime.Composable
import androidx.ink.brush.Brush
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.StockBrushes

@Composable
fun BrushSelector(
    currentBrush: Brush,
    onBrushSelected: (BrushFamily) -> Unit,
) {
    Row {
        IconButton(onClick = { onBrushSelected(StockBrushes.pressurePen()) }) {
            Icon(/* pen icon */)
        }
        IconButton(onClick = { onBrushSelected(StockBrushes.marker()) }) {
            Icon(/* marker icon */)
        }
        IconButton(onClick = { onBrushSelected(StockBrushes.highlighter()) }) {
            Icon(/* highlighter icon */)
        }
    }
}
```

Key points:
- Use `brush.copy(family = newFamily)` to switch brush families while preserving color and size
- Adjust alpha for highlighter brushes (`0.3f`) so they look translucent
- Restore full alpha (`1f`) when switching back to pen/marker

### Self-Evaluation

**Must Include checklist**:
- ✅ `StockBrushes.pressurePen()`, `.marker()`, `.highlighter()` — all three present
- ✅ `brush.copy(family = newFamily)` for switching — present
- ✅ Alpha adjustment for highlighter (e.g., `color.copy(alpha = 0.3f)`) — present
- ✅ State management for current brush — present (`MutableStateFlow<Brush>`)

**Must NOT Include checklist**:
- ✅ ❌ Recreating entire Brush object from scratch each time — correctly absent (uses `copy()`)
- ✅ ❌ Using different Canvas layers for different brushes — correctly absent

**Score**: **PASS**

---

## Eval 9: Highlighter with Blend Mode

**Prompt**: "Make my highlighter brush translucent so it looks like a real highlighter"

**Reference docs consulted**: `stock-brushes.md`, `stroke-rendering.md`

### Generated Response

To create a realistic highlighter effect, combine a translucent color with `BlendMode.Multiply` rendering:

**1. Create the highlighter brush with reduced alpha**:

```kotlin
import androidx.ink.brush.Brush
import androidx.ink.brush.StockBrushes
import androidx.ink.brush.compose.createWithComposeColor
import androidx.compose.ui.graphics.Color

val highlighterBrush = Brush.createWithComposeColor(
    family = StockBrushes.highlighter(),
    color = Color.Yellow.copy(alpha = 0.3f),
    size = 20f,
    epsilon = 0.1f
)
```

**2. Render with per-stroke blend mode using `withSaveLayer`**:

```kotlin
import android.graphics.Matrix
import androidx.compose.foundation.Canvas
import androidx.compose.ui.geometry.toRect
import androidx.compose.ui.graphics.BlendMode
import androidx.compose.ui.graphics.withSaveLayer
import androidx.compose.ui.graphics.nativeCanvas
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

**Why `withSaveLayer`?**
- Creates an offscreen buffer for each stroke
- The blend mode is applied when the buffer is composited back onto the main canvas
- Without it, `BlendMode.Multiply` only affects how each pixel blends with what's already drawn, leading to incorrect rendering for semi-transparent brushes

### Self-Evaluation

**Must Include checklist**:
- ✅ `StockBrushes.highlighter()` — present
- ✅ `BlendMode.Multiply` or alpha-based transparency — present (both `BlendMode.Multiply` AND `alpha = 0.3f`)
- ✅ `drawContext.canvas.withSaveLayer` for per-stroke blend mode — present
- ✅ Color with `alpha = 0.3f` (or similar) — present

**Must NOT Include checklist**:
- ✅ ❌ Global canvas alpha — correctly absent
- ✅ ❌ Semi-transparent pen as a substitute — correctly absent (uses `StockBrushes.highlighter()`)

**Score**: **PASS**

---

## Eval 10: Bitmap Export

**Prompt**: "Export the current drawing as a PNG bitmap file"

**Reference docs consulted**: `advanced.md`, `stroke-rendering.md`

### Generated Response

```kotlin
import android.graphics.Bitmap
import android.graphics.Canvas
import android.graphics.Matrix
import android.net.Uri
import androidx.core.graphics.createBitmap
import androidx.core.graphics.withSave
import androidx.ink.brush.TextureBitmapStore
import androidx.ink.rendering.android.canvas.CanvasStrokeRenderer
import androidx.ink.strokes.Stroke
import java.io.File
import java.io.FileOutputStream

suspend fun createExportBitmap(
    width: Int,
    height: Int,
    strokes: List<Stroke>,
    textureStore: TextureBitmapStore,
): Bitmap {
    val bitmap = createBitmap(width, height)
    val canvas = Canvas(bitmap)

    // Draw white background
    canvas.drawColor(android.graphics.Color.WHITE)

    // Since 1.0.0 stable, CanvasStrokeRenderer.create() automatically falls back
    // to path rendering when drawing onto a software Canvas(bitmap).
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

    return bitmap
}

fun saveBitmapToFile(bitmap: Bitmap, file: File): Uri {
    FileOutputStream(file).use { output ->
        bitmap.compress(Bitmap.CompressFormat.PNG, 100, output)
    }
    return Uri.fromFile(file)
}
```

Usage:
```kotlin
// In ViewModel
fun exportDrawing(width: Int, height: Int) {
    viewModelScope.launch {
        val bitmap = createExportBitmap(width, height, currentStrokes, textureStore)
        val file = File(context.cacheDir, "drawing_export.png")
        saveBitmapToFile(bitmap, file)
        // Share or save the file as needed
    }
}
```

Key points:
- **Automatic software `Canvas` path fallback**: Starting in `1.0.0` (Stable), `CanvasStrokeRenderer.create(textureStore)` automatically detects software `Bitmap` canvases (`!canvas.isHardwareAccelerated`) and falls back to `Canvas.drawPath`. Do not call the 2-arg `forcePathRendering = true` overload, which is `@RestrictTo(LIBRARY_GROUP)`.
- Use `createBitmap(width, height)` from `androidx.core.graphics`
- Wrap the bitmap in an Android `Canvas(bitmap)` for rendering
- Save with `bitmap.compress(Bitmap.CompressFormat.PNG, 100, output)`

### Self-Evaluation

**Must Include checklist**:
- ✅ `createBitmap(width, height)` (or `Bitmap.createBitmap`) — present (`createBitmap(width, height)`)
- ✅ `Canvas(bitmap)` (Android Canvas wrapping bitmap) — present
- ✅ `CanvasStrokeRenderer.create()` (or `CanvasStrokeRenderer.create(textureStore)`) — present
- ✅ Drawing all strokes onto bitmap canvas — present (`strokes.forEach { stroke -> canvas.withSave { renderer.draw(...) } }`)
- ✅ Saving bitmap to file via `compress(PNG, ...)` — present (`bitmap.compress(Bitmap.CompressFormat.PNG, 100, output)`)

**Must NOT Include checklist**:
- ✅ ❌ Screenshot capture (`View.drawToBitmap()`) — correctly absent
- ✅ ❌ PixelCopy API — correctly absent
- ✅ ❌ Rendering to a Compose Canvas and somehow extracting pixels — correctly absent

**Score**: **PASS**

---

## Overall Assessment

### Results Summary

| Metric | Value |
|--------|-------|
| Total Evals | 10 |
| PASS | 10 |
| PARTIAL | 0 |
| FAIL | 0 |
| **Pass Rate** | **100%** |

### Skill Quality Assessment

The `ink-app-builder` skill reference docs are **excellent**. Key strengths:

1. **Complete code samples**: Every reference doc includes full, copy-pasteable Kotlin/Compose code
2. **Consistent API patterns**: The same patterns (`CanvasStrokeRenderer.create()`, `Brush.createWithComposeColor()`, etc.) are used consistently across all docs
3. **Anti-patterns documented**: The SKILL.md explicitly lists things NOT to do, preventing common mistakes
4. **Task routing table**: Makes it easy to find the right reference doc for any given task
5. **Architecture guidance**: The `architecture.md` provides a clear MVVM pattern that ties all pieces together

### No Gaps Found in Skill Docs

All 10 eval prompts are fully answerable from the skill reference docs alone. No external knowledge was needed.
