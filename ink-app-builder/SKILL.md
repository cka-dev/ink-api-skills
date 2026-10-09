---
name: ink-app-builder
description: >-
  Guides developers in building inking and drawing features in an Android
  project using the Android Jetpack Ink API (androidx.ink) with Jetpack
  Compose. Use this skill when adding freehand drawing, stylus input, digital
  ink, stock brushes, stroke erasing, undo/redo, or stroke persistence to an
  Android app. Delivers complete Compose drawing surfaces (InProgressStrokes),
  dry stroke rendering (CanvasStrokeRenderer), geometry-based erasing, and
  ViewModel/Repository persistence. Compose-only — does NOT cover View-based
  (XML) inking.
---

# Building Inking Apps with the Android Ink API (Compose)

The Android Ink API (`androidx.ink`) is a modular Jetpack library for building
high-performance, low-latency freehand drawing experiences on Android. It uses a
**wet ink / dry ink** two-phase model:

- **Wet ink**: Real-time, ultra-low-latency rendering while the user is drawing
  (handled by `InProgressStrokes`).
- **Dry ink**: Finalized, immutable `Stroke` objects rendered by your app via
  `CanvasStrokeRenderer`.

> **Note on Versions & Experimental APIs**:
> - **Core inking** works out of the box with **`androidx.ink` `1.0.0` (Stable)** and **`1.1.0+`** (`InProgressStrokes`, `CanvasStrokeRenderer`, `StockBrushes`, geometry eraser, and `StrokeInputBatch` persistence).
> - **Programmatic custom brushes** (`BrushFamily(...)`, `BrushCoat`, `BrushTip`, `BrushPaint`, `BrushBehavior`) graduated from `@RestrictTo(LIBRARY_GROUP)` to public API in **`1.1.0-alpha03+`** (in `1.0.0` stable, custom brushes can only be loaded from binary proto assets via `BrushFamily.decode()`).
> - In **`1.0.0` (Stable)**, `BrushFamily` serialization (`BrushFamily.encode()`/`decode()`, `AndroidBrushFamilySerialization`, `BrushFamilyDecodeCallback`) requires `@OptIn(ExperimentalInkCustomBrushApi::class)`. In **`1.1.0-alpha02+`**, `BrushFamily` serialization no longer requires `@OptIn` (and in `1.1.0-alpha05+`, `ExperimentalInkCustomBrushApi` itself is internal `@RestrictTo(LIBRARY_GROUP)`, so do not annotate code with it on `1.1.0-alpha03+`).

---

## Quick Start

For a new project that needs inking:

1. Read [setup.md](references/setup.md) — add all required dependencies
2. Read [drawing-surface.md](references/drawing-surface.md) — create the composable drawing surface
3. Read [stroke-rendering.md](references/stroke-rendering.md) — render finalized strokes
4. Read [stock-brushes.md](references/stock-brushes.md) — use built-in brushes

---

## Task Routing

Based on the required task, read the corresponding reference:

### Setting Up
| Task | Reference |
|---|---|
| Add Ink API dependencies to a project | [setup.md](references/setup.md) |
| Structure the MVVM app architecture | [architecture.md](references/architecture.md) |

### Core Drawing
| Task | Reference |
|---|---|
| Create a Compose drawing surface | [drawing-surface.md](references/drawing-surface.md) |
| Render finalized strokes on a Canvas | [stroke-rendering.md](references/stroke-rendering.md) |
| Use and switch stock brushes (pen, marker, highlighter, dashed, emoji) | [stock-brushes.md](references/stock-brushes.md) |

### Editing Features
| Task | Reference |
|---|---|
| Implement a geometry-based eraser tool | [eraser.md](references/eraser.md) |
| Add undo/redo functionality | [undo-redo.md](references/undo-redo.md) |

### Persistence & Advanced
| Task | Reference |
|---|---|
| Save and load strokes (Room database) | [persistence.md](references/persistence.md) |
| Export drawing as bitmap, motion prediction, drag & drop, textures | [advanced.md](references/advanced.md) |

---

## Anti-Patterns — Do NOT Do These

- ❌ **Do NOT use `InProgressStrokesView`** — that is the View-based API. Use
  `InProgressStrokes` composable instead.
- ❌ **Do NOT omit `ink-nativeloader` or `ink-authoring-android`** — missing
  `ink-nativeloader` causes `UnsatisfiedLinkError` and missing
  `ink-authoring-android` causes `ClassNotFoundException` at runtime.
- ❌ **Do NOT use `View.OnTouchListener` or `MotionEvent` directly** — the
  `InProgressStrokes` composable handles touch input internally.
- ❌ **Do NOT draw strokes manually** with `Canvas.drawPath()` or by iterating
  stroke points — use `CanvasStrokeRenderer.draw(canvas, stroke, transform)`.
- ❌ **Do NOT call `CanvasStrokeRenderer.create(forcePathRendering = true)`** — the
  2-argument `forcePathRendering` overload is `@RestrictTo(LIBRARY_GROUP)` (and
  `@InkInternalOnlyApi` in `1.1.0-alpha09+`). Use the public
  `CanvasStrokeRenderer.create(textureStore)` factory; since `1.0.0` stable, it
  automatically detects software `Bitmap` canvases and falls back to path rendering.
- ❌ **Do NOT rely solely on asynchronous `StateFlow` collection in `onStrokesFinished`** — `InProgressStrokes` removes completed wet strokes immediately after `onStrokesFinished` returns on the UI thread, while `collectAsStateWithLifecycle()` updates on a later run loop, causing a 1-frame wet-to-dry flicker. Always mutate Compose Snapshot State synchronously in the same UI thread run loop (such as the `pendingStrokes` buffer inside `DrawingSurface`; see [drawing-surface.md](references/drawing-surface.md)).
- ❌ **Do NOT store strokes as bitmaps** for undo/redo — store `Stroke` objects
  and re-render them.
- ❌ **Do NOT hardcode Ink API version numbers** — use a version catalog or
  variable so the developer can update easily.
