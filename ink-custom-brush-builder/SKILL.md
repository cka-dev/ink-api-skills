---
name: ink-custom-brush-builder
description: >-
  Guides developers in creating custom brushes for an Android app or Android
  project using the Android Jetpack Ink API (androidx.ink). Use this skill
  when building custom BrushFamily definitions, configuring BrushTip shapes,
  adding pressure, tilt, speed, or noise sensitivity via BrushBehavior node
  graphs, adding tiling or particle stamping textures, creating multi-coat
  brushes, or serializing and deserializing custom brushes. Delivers complete
  Kotlin BrushFamily, BrushCoat, BrushTip, BrushPaint, BrushBehavior,
  TextureLayer, and ColorFunction implementations.
---

# Building Custom Brushes with the Android Ink API

Custom brushes in the Ink API are built around the **BrushFamily** hierarchy:

```
BrushFamily
├── InputModel (BrushFamily.InputModel.SlidingWindowModel or PASSTHROUGH_MODEL)
└── BrushCoat[] (one or more layers)
    ├── BrushTip (shape: scale, rotation, corner rounding)
    │   └── BrushBehavior[] (dynamic response to input)
    │       └── Tree: TargetNode(input = [Operators](input = SourceNode))
    └── BrushPaint (visual appearance)
        ├── SelfOverlap (how overlapping parts blend)
        ├── TextureLayer[] (BrushPaint.TilingTexture or StampingTexture)
        └── BrushPaint.ColorFunction[] (opacity/color modifiers)
```

A `Brush` is then created by combining a `BrushFamily` with a color and size:
```kotlin
val brush = Brush.createWithComposeColor(
    family = myCustomFamily,
    color = Color.Black,
    size = 10f,
    epsilon = 0.1f
)
```

> **⚠️ Version & Experimental API Requirement**:
> - **Requires `androidx.ink` `1.1.0-alpha03+`**: In `1.0.0` (Stable) (and `1.1.0-alpha01`–`alpha02`), `BrushFamily(...)`, `BrushCoat`, `BrushTip`, `BrushPaint`, `BrushBehavior`, and `androidx.ink.brush.behavior.*` nodes were `@RestrictTo(LIBRARY_GROUP)` (in `1.0.0` stable, custom brushes can only be loaded from binary proto streams via `BrushFamily.decode()`). Starting in **`1.1.0-alpha03+`**, the entire custom brush creation API graduated to public API (`BrushPaint.TilingTexture` / `BrushPaint.StampingTexture`, `BrushPaint.ColorFunction`, `BrushFamily.InputModel.SlidingWindowModel`, and all behavior nodes).
> - **No `@OptIn` Required in `1.1.0-alpha03+`**: All public custom brush classes and `ink-storage` `BrushFamily` serialization APIs (`BrushFamily.encode()`/`decode()`, `AndroidBrushFamilySerialization`, `BrushFamilyDecodeCallback`) are unannotated in `1.1.0-alpha03+` and require **no `@OptIn`** (`ExperimentalInkCustomBrushApi` itself is internal `@RestrictTo(LIBRARY_GROUP)` in `1.1.0-alpha05+`, and is only public/required on `BrushFamily` serialization in `1.0.0` stable).

---

## Quick Start

1. Add `androidx.ink:ink-brush`, `androidx.ink:ink-brush-compose`, `androidx.ink:ink-storage`, and `androidx.ink:ink-nativeloader` (version `1.1.0-alpha03+` or newer) to `app/build.gradle.kts`.
2. Read [brush-hierarchy.md](references/brush-hierarchy.md) — understand `BrushFamily` → `BrushCoat` → `BrushTip` + `BrushPaint`
3. Read [brush-behaviors.md](references/brush-behaviors.md) — add pressure, tilt, speed, or noise responsiveness
4. Read [examples.md](references/examples.md) — start from a complete recipe (pressure pen, calligraphy, shading pencil, watercolor, multi-coat)

---

## Task Routing

Based on the required task, read the corresponding reference:

### Understanding the Structure
| Task | Reference |
|---|---|
| Understand the BrushFamily hierarchy | [brush-hierarchy.md](references/brush-hierarchy.md) |
| Configure brush tip shape | [brush-tip.md](references/brush-tip.md) |
| Configure paint properties (overlap, color) | [brush-paint.md](references/brush-paint.md) |

### Adding Dynamic Behavior
| Task | Reference |
|---|---|
| Make a brush respond to pressure, tilt, or speed | [brush-behaviors.md](references/brush-behaviors.md) |
| Configure input smoothing / upsampling (`InputModel`) | [input-model.md](references/input-model.md) |

### Textures & Visuals
| Task | Reference |
|---|---|
| Add texture layers and `TextureBitmapStore` to a brush | [textures.md](references/textures.md) |

### Saving & Loading
| Task | Reference |
|---|---|
| Serialize/deserialize brush families | [serialization.md](references/serialization.md) |

### Complete Recipes
| Task | Reference |
|---|---|
| Complete brush recipes (pressure pen, calligraphy, pencil, watercolor, sparkle) | [examples.md](references/examples.md) |

---

## Anti-Patterns — Do NOT Do These

- ❌ **Do NOT use raw `ink.proto.*` builders** to construct brushes — use the
  Kotlin constructors (`BrushFamily`, `BrushCoat`, `BrushTip`, `BrushPaint`,
  `BrushBehavior` from `androidx.ink.brush.*`).
- ❌ **Do NOT manually scale stroke width** in a touch listener for pressure
  sensitivity — use `BrushBehavior` with `SourceNode.Source.NORMALIZED_PRESSURE` →
  `TargetNode.Target.SIZE_MULTIPLIER` (or `WIDTH_MULTIPLIER`).
- ❌ **Do NOT use Canvas bitmap stamping** for textured brushes — use
  `BrushPaint.TilingTexture` / `BrushPaint.StampingTexture` with a `TextureBitmapStore`.
- ❌ **Do NOT serialize brushes as JSON or Parcelable** — use the `ink-storage`
  `BrushFamily.encode()`/`decode()` or `AndroidBrushFamilySerialization` API.
- ❌ **Do NOT implement Bézier smoothing manually** — rely on the brush
  `InputModel` (`BrushFamily.InputModel.SlidingWindowModel`) for input smoothing.
- ❌ **Do NOT use random number generators** in draw loops for animated
  effects — use `NoiseNode` in `BrushBehavior`.
