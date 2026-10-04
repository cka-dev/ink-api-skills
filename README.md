# Ink API Skills

AI agent skills for building inking apps and custom brushes with the
[Android Jetpack Ink API](https://developer.android.com/jetpack/androidx/releases/ink)
(`androidx.ink`).

## Skills

### 🖊️ ink-app-builder

Guides developers through building complete inking/drawing apps with Jetpack
Compose. Covers:

- Project setup and dependencies
- Drawing surfaces (`InProgressStrokes`)
- Stroke rendering (`CanvasStrokeRenderer`)
- Stock brushes (pen, marker, highlighter, etc.)
- Eraser tool (geometry-based intersection)
- Undo/redo
- Stroke persistence (Room database)
- MVVM architecture patterns
- Advanced features (motion prediction, bitmap export, drag & drop)

📁 [`ink-app-builder/SKILL.md`](ink-app-builder/SKILL.md)

### 🎨 ink-custom-brush-builder

Guides developers through creating custom brushes using the full `BrushFamily`
hierarchy. Covers:

- BrushFamily → BrushCoat → BrushTip + BrushPaint structure
- Brush tip shape configuration
- Paint properties (overlap, color functions)
- Dynamic behaviors (pressure, tilt, speed, noise)
- Texture layers
- Brush serialization/deserialization
- Input model configuration
- Complete recipes (pressure pen, calligraphy, shading pencil, etc.)

📁 [`ink-custom-brush-builder/SKILL.md`](ink-custom-brush-builder/SKILL.md)

## Scope & Constraints

| Constraint | Value |
|---|---|
| **Platform** | Jetpack Compose only (Compose-first) |
| **Versions (`ink-app-builder`)** | Compatible with **`1.0.0` (stable)** and **`1.1.0+`** (uses version catalog `<latest_version>`) |
| **Versions (`ink-custom-brush-builder`)** | Programmatic custom brush creation (`BrushFamily`, `BrushCoat`, `BrushTip`, `BrushPaint`, `BrushBehavior`, `androidx.ink.brush.behavior.*`) requires **`1.1.0-alpha03+`** (where the custom brush API graduated from `@RestrictTo(LIBRARY_GROUP)` and `@ExperimentalInkCustomBrushApi` to public API; in `1.0.0` stable, custom brushes are `@RestrictTo(LIBRARY_GROUP)` and can only be loaded from binary proto streams via `BrushFamily.decode()`) |
| **Experimental APIs** | In `1.0.0` stable, `@OptIn(ExperimentalInkCustomBrushApi::class)` is required for `BrushFamily.encode()`/`decode()` and `AndroidBrushFamilySerialization`. In `1.1.0-alpha03+`, public custom brush and `BrushFamily` serialization APIs require **no `@OptIn`** (`ExperimentalInkCustomBrushApi` itself is internal `@RestrictTo(LIBRARY_GROUP)` in `1.1.0-alpha05+`). |

## Eval Suite

The `evals/` directory contains test prompts and a coverage matrix to validate
skill correctness:

- [`ink-app-builder-evals.md`](evals/ink-app-builder-evals.md) — 10 test prompts
- [`results-ink-app-builder.md`](evals/results-ink-app-builder.md) — Reference responses and self-evaluations for `ink-app-builder`
- [`ink-custom-brush-builder-evals.md`](evals/ink-custom-brush-builder-evals.md) — 8 test prompts
- [`results-ink-custom-brush-builder.md`](evals/results-ink-custom-brush-builder.md) — Reference responses and self-evaluations for `ink-custom-brush-builder`
- [`coverage-matrix.md`](evals/coverage-matrix.md) — API surface coverage checklist

## Installation

Copy or symlink the skill directories into your agent's skills folder:

```bash
# Example for Antigravity/Gemini agents
cp -r ink-app-builder/ ~/.gemini/config/plugins/your-plugin/skills/
cp -r ink-custom-brush-builder/ ~/.gemini/config/plugins/your-plugin/skills/
```

## Source

These skills were developed using:
- **[Cahier](https://github.com/android/cahier)** — Google's official Android
  Ink API hero sample app
- **[Ink API Documentation](https://developer.android.com/develop/ui/compose/touch-input/stylus-input/ink-api)** —
  Official developer guides and API reference

## License

Apache License 2.0
