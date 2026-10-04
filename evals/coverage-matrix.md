# Ink API Skills — Coverage Matrix

Maps every key Ink API element to the skill reference document(s) that cover it.
Use this to verify no critical API surface is missing from the skills.

## Legend
- ✅ = Covered with code examples
- 📝 = Mentioned/referenced (no full code example)
- — = Not applicable to this skill

---

## Modules

| Maven Artifact | ink-app-builder | ink-custom-brush-builder |
|---|---|---|
| `ink-authoring` | ✅ setup.md | — |
| `ink-authoring-android` | ✅ setup.md | — |
| `ink-authoring-compose` | ✅ drawing-surface.md | — |
| `ink-brush-compose` | ✅ stock-brushes.md | ✅ brush-hierarchy.md |
| `ink-brush` | ✅ stock-brushes.md | ✅ brush-hierarchy.md |
| `ink-geometry-compose` | ✅ setup.md, eraser.md | — |
| `ink-geometry` | ✅ eraser.md | — |
| `ink-rendering` | ✅ stroke-rendering.md | — |
| `ink-strokes` | ✅ drawing-surface.md | 📝 examples.md |
| `ink-storage` | ✅ persistence.md | ✅ serialization.md |
| `ink-nativeloader` | ✅ setup.md | 📝 SKILL.md |
| `input-motionprediction` | ✅ setup.md, advanced.md | — |

---

## Key Classes — ink-app-builder

| Class / API | Reference Doc | Status |
|---|---|---|
| `InProgressStrokes` (composable) | drawing-surface.md | ✅ |
| `CanvasStrokeRenderer` | stroke-rendering.md | ✅ |
| `Brush` | stock-brushes.md | ✅ |
| `Brush.createWithComposeColor()` | stock-brushes.md | ✅ |
| `Brush.copy()` | stock-brushes.md | ✅ |
| `Brush.copyWithComposeColor()` | stock-brushes.md | ✅ |
| `StockBrushes.pressurePen()` | stock-brushes.md | ✅ |
| `StockBrushes.marker()` | stock-brushes.md | ✅ |
| `StockBrushes.highlighter()` | stock-brushes.md | ✅ |
| `StockBrushes.dashedLine()` | stock-brushes.md | ✅ |
| `StockBrushes.emojiHighlighter()` | stock-brushes.md | ✅ |
| `Stroke` (immutable) | drawing-surface.md | ✅ |
| `MutableVec` | eraser.md | ✅ |
| `MutableSegment` | eraser.md | ✅ |
| `MutableParallelogram` | eraser.md | ✅ |
| `AffineTransform` | eraser.md | ✅ |
| `Intersection.intersects()` | eraser.md | ✅ |
| `BlendMode.Multiply` | stroke-rendering.md | ✅ |
| `TextureBitmapStore` | advanced.md | ✅ |
| `Matrix()` (stroke transform) | stroke-rendering.md | ✅ |

---

## Key Classes — ink-custom-brush-builder

| Class / API | Reference Doc | Status |
|---|---|---|
| `BrushFamily` | brush-hierarchy.md | ✅ |
| `BrushCoat` | brush-hierarchy.md | ✅ |
| `BrushTip` | brush-tip.md | ✅ |
| `BrushPaint` | brush-paint.md | ✅ |
| `BrushBehavior` | brush-behaviors.md | ✅ |
| `SourceNode` | brush-behaviors.md | ✅ |
| `ConstantNode` | brush-behaviors.md | ✅ |
| `NoiseNode` | brush-behaviors.md | ✅ |
| `ToolTypeFilterNode` | brush-behaviors.md | ✅ |
| `DampingNode` | brush-behaviors.md | ✅ |
| `ResponseNode` | brush-behaviors.md | ✅ |
| `IntegralNode` | brush-behaviors.md | ✅ |
| `BinaryOpNode` | brush-behaviors.md | ✅ |
| `InterpolationNode` | brush-behaviors.md | ✅ |
| `TargetNode` | brush-behaviors.md | ✅ |
| `PolarTargetNode` | brush-behaviors.md | ✅ |
| `SourceNode.Source.*` (all) | brush-behaviors.md | ✅ |
| `TargetNode.Target.*` (all) | brush-behaviors.md | ✅ |
| `OutOfRange.*` | brush-behaviors.md | ✅ |
| `ProgressDomain.*` | brush-behaviors.md | ✅ |
| `SelfOverlap` | brush-paint.md | ✅ |
| `BrushPaint.TextureLayer` (`TilingTexture` / `StampingTexture`) | textures.md | ✅ |
| `BrushPaint.ColorFunction.OpacityMultiplier` | brush-paint.md | ✅ |
| `BrushPaint.ColorFunction.ReplaceColor` | brush-paint.md | ✅ |
| `TextureBitmapStore` | textures.md | ✅ |
| `BrushFamily.encode()` | serialization.md | ✅ |
| `BrushFamily.decode()` | serialization.md | ✅ |
| `AndroidBrushFamilySerialization` | serialization.md | ✅ |
| `Version.DEVELOPMENT` | serialization.md | ✅ |
| `BrushFamily.InputModel.DEFAULT_INPUT_MODEL` / `SlidingWindowModel` | input-model.md | ✅ |
| `BrushFamily.InputModel.PASSTHROUGH_MODEL` | input-model.md | ✅ |
| `@ExperimentalInkCustomBrushApi` (`1.0.0` stable serialization) | serialization.md | 📝 |

---

## Behavioral Patterns

| Pattern | ink-app-builder | ink-custom-brush-builder |
|---|---|---|
| Wet/dry ink separation | ✅ drawing-surface.md | — |
| Eraser via geometry intersection | ✅ eraser.md | — |
| Undo/redo history stack | ✅ undo-redo.md | — |
| Stroke persistence (Room) | ✅ persistence.md | — |
| MVVM architecture | ✅ architecture.md | — |
| Brush switching | ✅ stock-brushes.md | — |
| Bitmap export | ✅ advanced.md | — |
| Motion prediction | ✅ advanced.md | — |
| Drag & drop coexistence | ✅ advanced.md | — |
| Pressure → size behavior | — | ✅ brush-behaviors.md |
| Tilt → width behavior | — | ✅ brush-behaviors.md |
| Speed → opacity behavior | — | ✅ brush-behaviors.md |
| Noise jitter | — | ✅ brush-behaviors.md |
| Multi-coat brushes | — | ✅ brush-hierarchy.md |
| Texture brushes | — | ✅ textures.md |
| Brush serialization | 📝 persistence.md | ✅ serialization.md |
| Input model configuration | — | ✅ input-model.md |

---

## Gaps Identified

> None critical. All key Ink API classes and patterns are covered by at least
> one skill reference document.

### Minor gaps (acceptable):
- `InProgressStrokesView` (View-based) — intentionally excluded (Compose-only)
- `ViewStrokeRenderer` (View-based) — intentionally excluded
- `InProgressStrokesManager` (low-level) — too advanced for these skills
- `StrokeInputBatch` (manual input building) — niche use case
- OpenGL/Vulkan rendering — not publicly documented yet
