# Ink App Builder — Eval Prompts

Test prompts to validate the `ink-app-builder` skill produces correct, Compose-only
Ink API code. Each prompt simulates a real developer request.

---

## Eval 1: Setup Dependencies

**Prompt**: "Add the Android Ink API dependencies to my Compose project"

**Must Include**:
- `ink-authoring` and `ink-authoring-android`
- `ink-authoring-compose` (not `ink-authoring` alone)
- `ink-brush` and `ink-brush-compose`
- `ink-geometry` and `ink-geometry-compose`
- `ink-rendering`
- `ink-strokes`
- `ink-storage`
- `ink-nativeloader`
- Version catalog pattern or single version variable

**Must NOT Include**:
- ❌ Hardcoded version numbers like `1.0.0-alpha01`
- ❌ `InProgressStrokesView` (View-based surface — note that `ink-authoring-android` IS required as a runtime dependency)

**Expected Pattern**:
```kotlin
val ink_version = "<latest_version>"
implementation("androidx.ink:ink-authoring-compose:$ink_version")
implementation("androidx.ink:ink-brush-compose:$ink_version")
// ... etc
```

---

## Eval 2: Basic Drawing Surface

**Prompt**: "Create a Compose screen where users can draw with their finger or stylus"

**Must Include**:
- `InProgressStrokes` composable
- `CanvasStrokeRenderer.create()`
- `Canvas` composable for dry ink rendering
- `canvasStrokeRenderer.draw(canvas, stroke, Matrix())`
- `Brush.createWithComposeColor(family, color, size, epsilon)`
- `onStrokesFinished` callback to capture completed strokes
- `Box` layering: Canvas underneath, InProgressStrokes on top

**Must NOT Include**:
- ❌ `InProgressStrokesView`
- ❌ `View.OnTouchListener`
- ❌ `MotionEvent` handling
- ❌ `Canvas.drawPath()` or manual point iteration

**Expected Architecture**:
```
Box {
    Canvas { /* render dry strokes */ }
    InProgressStrokes(defaultBrush, onStrokesFinished, nextBrush)
}
```

---

## Eval 3: Brush Creation

**Prompt**: "Create a Brush using a stock pen brush family"

**Must Include**:
- `StockBrushes.pressurePen()`
- `Brush.createWithComposeColor(family, color, size, epsilon)`
- `Color` from `androidx.compose.ui.graphics`
- `epsilon` parameter with explanation

**Must NOT Include**:
- ❌ `Brush()` direct constructor
- ❌ `Color.argb()` (Android framework color)
- ❌ Missing `epsilon`

---

## Eval 4: Eraser Implementation

**Prompt**: "Implement an eraser tool that removes strokes the user draws over"

**Must Include**:
- `MutableSegment` for eraser path
- `MutableParallelogram` with `populateFromSegmentAndPadding`
- `stroke.shape.intersects(parallelogram, AffineTransform.IDENTITY)`
- Previous point tracking
- `detectDragGestures` for eraser input
- Filtering strokes list to remove intersecting strokes

**Must NOT Include**:
- ❌ Drawing white strokes over existing ones
- ❌ Bitmap masking
- ❌ Per-pixel comparison
- ❌ `Canvas.clipPath()`

---

## Eval 5: Undo/Redo

**Prompt**: "Add undo and redo to my drawing app"

**Must Include**:
- `mutableListOf<List<Stroke>>()` history stack
- `historyIndex` integer tracker
- Truncating future states on new stroke
- `canUndo` / `canRedo` StateFlow or State
- Updating UI state after undo/redo

**Must NOT Include**:
- ❌ Command pattern with separate command classes
- ❌ Storing bitmaps for each state
- ❌ Cloning strokes (they're immutable)

---

## Eval 6: Stroke Persistence

**Prompt**: "Save my drawing strokes to a Room database so they persist across app launches"

**Must Include**:
- Serialization using `ink-storage` module
- Room entity storing serialized stroke data (as `List<String>`, `ByteArray`, or encoded `String`)
- Room TypeConverter for Stroke list serialization
- Loading strokes on ViewModel init

**Must NOT Include**:
- ❌ Bitmap storage
- ❌ JSON serialization of stroke points
- ❌ SharedPreferences

---

## Eval 7: Stroke Rendering

**Prompt**: "Render a list of finished strokes on a Compose Canvas"

**Must Include**:
- `CanvasStrokeRenderer.create()`
- `Canvas(modifier)` composable
- `drawContext.canvas.nativeCanvas` access
- `canvasStrokeRenderer.draw(canvas, stroke, Matrix())`
- `canvas.withSave { }` block

**Must NOT Include**:
- ❌ Iterating stroke input points and drawing lines
- ❌ `Path.moveTo()` / `Path.lineTo()`
- ❌ `drawPath()` calls

---

## Eval 8: Brush Switching

**Prompt**: "Let users switch between pen, marker, and highlighter brushes"

**Must Include**:
- `StockBrushes.pressurePen()`, `.marker()`, `.highlighter()`
- `brush.copy(family = newFamily)` for switching
- Alpha adjustment for highlighter (e.g., `color.copy(alpha = 0.3f)`)
- State management for current brush

**Must NOT Include**:
- ❌ Recreating entire Brush object from scratch each time
- ❌ Using different Canvas layers for different brushes

---

## Eval 9: Highlighter with Blend Mode

**Prompt**: "Make my highlighter brush translucent so it looks like a real highlighter"

**Must Include**:
- `StockBrushes.highlighter()`
- `BlendMode.Multiply` or alpha-based transparency
- `drawContext.canvas.withSaveLayer` for per-stroke blend mode
- Color with `alpha = 0.3f` (or similar)

**Must NOT Include**:
- ❌ Global canvas alpha
- ❌ Semi-transparent pen as a substitute

---

## Eval 10: Bitmap Export

**Prompt**: "Export the current drawing as a PNG bitmap file"

**Must Include**:
- `createBitmap(width, height)` (or `Bitmap.createBitmap`)
- `Canvas(bitmap)` (Android Canvas wrapping bitmap)
- `CanvasStrokeRenderer.create()` (or `CanvasStrokeRenderer.create(textureStore)`)
- Drawing all strokes onto bitmap canvas
- Saving bitmap to file via `compress(PNG, ...)`

**Must NOT Include**:
- ❌ Screenshot capture (`View.drawToBitmap()`)
- ❌ PixelCopy API
- ❌ Rendering to a Compose Canvas and somehow extracting pixels
