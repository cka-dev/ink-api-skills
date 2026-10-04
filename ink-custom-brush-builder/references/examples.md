# Complete Brush Recipes

## Overview

This reference provides complete, copy-ready code for common brush types. Each recipe builds a full `BrushFamily` from scratch using data-class constructors, and includes all necessary behaviors, paint configuration, and tip settings.

All recipes use the helper functions from `brush-behaviors.md`. For brevity, those are inlined here.

> **Note:** Programmatic custom brush creation requires `androidx.ink` **`1.1.0-alpha03+`** (where all public custom brush classes and constructors require no `@OptIn`).

## Shared Helpers

```kotlin
import androidx.ink.brush.BrushBehavior
import androidx.ink.brush.BrushCoat
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.BrushPaint
import androidx.ink.brush.BrushTip
import androidx.ink.brush.Brush
import androidx.ink.brush.SelfOverlap
import androidx.ink.brush.behavior.DampingNode
import androidx.ink.brush.behavior.NoiseNode
import androidx.ink.brush.behavior.OutOfRange
import androidx.ink.brush.behavior.ProgressDomain
import androidx.ink.brush.behavior.SourceNode
import androidx.ink.brush.behavior.TargetNode

// --- Pattern builders (tree-based) ---

fun directBehavior(
    source: SourceNode.Source,
    sourceStart: Float, sourceEnd: Float,
    target: TargetNode.Target,
    targetStart: Float, targetEnd: Float,
): BrushBehavior = BrushBehavior(
    terminalNode = TargetNode(
        target = target,
        targetModifierRangeStart = targetStart,
        targetModifierRangeEnd = targetEnd,
        input = SourceNode(
            source = source,
            sourceValueRangeStart = sourceStart,
            sourceValueRangeEnd = sourceEnd,
            sourceOutOfRangeBehavior = OutOfRange.CLAMP,
        ),
    ),
)

fun smoothedBehavior(
    source: SourceNode.Source,
    sourceStart: Float, sourceEnd: Float,
    target: TargetNode.Target,
    targetStart: Float, targetEnd: Float,
    dampingSeconds: Float,
    outOfRange: OutOfRange = OutOfRange.CLAMP,
): BrushBehavior = BrushBehavior(
    terminalNode = TargetNode(
        target = target,
        targetModifierRangeStart = targetStart,
        targetModifierRangeEnd = targetEnd,
        input = DampingNode(
            dampOver = ProgressDomain.TIME_IN_SECONDS,
            strength = dampingSeconds,
            input = SourceNode(
                source = source,
                sourceValueRangeStart = sourceStart,
                sourceValueRangeEnd = sourceEnd,
                sourceOutOfRangeBehavior = outOfRange,
            ),
        ),
    ),
)

fun jitterBehavior(
    target: TargetNode.Target,
    targetStart: Float, targetEnd: Float,
    basePeriod: Float,
): BrushBehavior = BrushBehavior(
    terminalNode = TargetNode(
        target = target,
        targetModifierRangeStart = targetStart,
        targetModifierRangeEnd = targetEnd,
        input = NoiseNode(
            seed = kotlin.random.Random.nextInt(),
            varyOver = ProgressDomain.DISTANCE_IN_CENTIMETERS,
            basePeriod = basePeriod,
        ),
    ),
)
```

---

## Recipe 1: Simple Pressure Pen

A basic pen where stylus pressure controls both size and opacity. Uses `SelfOverlap.ANY` so the mesh renderer can apply the per-vertex `OPACITY_MULTIPLIER` behavior.

**Behaviors:** pressure → size, pressure → opacity

```kotlin
val pressurePen = BrushFamily(
    coats = listOf(
        BrushCoat(
            tip = BrushTip(
                scaleX = 1f,
                scaleY = 1f,
                cornerRounding = 1f,
                behaviors = listOf(
                    // Pressure → Size: 50%–150%
                    smoothedBehavior(
                        source = SourceNode.Source.NORMALIZED_PRESSURE,
                        sourceStart = 0f, sourceEnd = 1f,
                        target = TargetNode.Target.SIZE_MULTIPLIER,
                        targetStart = 0.5f, targetEnd = 1.5f,
                        dampingSeconds = 0.15f,
                    ),
                    // Pressure → Opacity: 30%–100%
                    smoothedBehavior(
                        source = SourceNode.Source.NORMALIZED_PRESSURE,
                        sourceStart = 0f, sourceEnd = 1f,
                        target = TargetNode.Target.OPACITY_MULTIPLIER,
                        targetStart = 0.3f, targetEnd = 1.0f,
                        dampingSeconds = 0.1f,
                    ),
                ),
            ),
            paintPreferences = listOf(
                BrushPaint(
                    selfOverlap = SelfOverlap.ANY,
                    colorFunctions = listOf(
                        BrushPaint.ColorFunction.OpacityMultiplier(1f),
                    ),
                ),
            ),
        ),
    ),
)
```

---

## Recipe 2: Calligraphy Brush

A calligraphy-style brush with a narrow, rotated tip. Tilt controls width, stroke direction controls rotation, producing elegant thick-thin variation.

**Behaviors:** tilt → width, direction → rotation

```kotlin
val calligraphyBrush = BrushFamily(
    coats = listOf(
        BrushCoat(
            tip = BrushTip(
                scaleX = 0.3f,          // Narrow horizontal
                scaleY = 1.0f,          // Full vertical
                cornerRounding = 0.2f,  // Slightly rounded rectangle
                rotationDegrees = 45f,  // 45° default nib rotation (in degrees)
                behaviors = listOf(
                    // Tilt → Width: 1×–2.5×
                    smoothedBehavior(
                        source = SourceNode.Source.TILT_IN_RADIANS,
                        sourceStart = 0f,
                        sourceEnd = (Math.PI / 2).toFloat(),
                        target = TargetNode.Target.WIDTH_MULTIPLIER,
                        targetStart = 1.0f, targetEnd = 2.5f,
                        dampingSeconds = 0.1f,
                    ),
                    // Direction → Rotation: tip follows stroke direction (use REPEAT for cyclic angles)
                    smoothedBehavior(
                        source = SourceNode.Source.DIRECTION_IN_RADIANS,
                        sourceStart = 0f,
                        sourceEnd = (2 * Math.PI).toFloat(),
                        target = TargetNode.Target.ROTATION_OFFSET_IN_RADIANS,
                        targetStart = (-Math.PI).toFloat(),
                        targetEnd = Math.PI.toFloat(),
                        dampingSeconds = 0.05f,
                        outOfRange = OutOfRange.REPEAT,
                    ),
                ),
            ),
            paintPreferences = listOf(
                BrushPaint(
                    selfOverlap = SelfOverlap.DISCARD,
                    colorFunctions = listOf(
                        BrushPaint.ColorFunction.OpacityMultiplier(1f),
                    ),
                ),
            ),
        ),
    ),
)
```

---

## Recipe 3: Shading Pencil

A pencil with tilt-responsive width, pressure-controlled opacity, slant noise for organic feel, and a texture layer for grain. Uses `SelfOverlap.ANY` so the mesh renderer applies per-vertex pressure opacity and accumulates shading density where the stroke overlaps itself.

**Behaviors:** tilt → width, pressure → opacity, noise → slant jitter

```kotlin
val shadingPencil = BrushFamily(
    coats = listOf(
        BrushCoat(
            tip = BrushTip(
                scaleX = 1f,
                scaleY = 1f,
                cornerRounding = 1f,
                behaviors = listOf(
                    // Tilt → Width: tilting widens the stroke
                    smoothedBehavior(
                        source = SourceNode.Source.TILT_IN_RADIANS,
                        sourceStart = 0f,
                        sourceEnd = (Math.PI / 2).toFloat(),
                        target = TargetNode.Target.WIDTH_MULTIPLIER,
                        targetStart = 1.0f, targetEnd = 2.5f,
                        dampingSeconds = 0.1f,
                    ),
                    // Pressure → Opacity: lighter pressure = more transparent
                    smoothedBehavior(
                        source = SourceNode.Source.NORMALIZED_PRESSURE,
                        sourceStart = 0f, sourceEnd = 1f,
                        target = TargetNode.Target.OPACITY_MULTIPLIER,
                        targetStart = 0.3f, targetEnd = 1.0f,
                        dampingSeconds = 0.1f,
                    ),
                    // Noise → Slant: pencil wobble ±0.15 radians
                    jitterBehavior(
                        target = TargetNode.Target.SLANT_OFFSET_IN_RADIANS,
                        targetStart = -0.15f, targetEnd = 0.15f,
                        basePeriod = 0.3f,
                    ),
                ),
            ),
            paintPreferences = listOf(
                BrushPaint(
                    selfOverlap = SelfOverlap.ANY,
                    colorFunctions = listOf(
                        BrushPaint.ColorFunction.OpacityMultiplier(0.8f),
                    ),
                    // Texture layer for pencil grain
                    textureLayers = listOf(
                        BrushPaint.TilingTexture(
                            clientTextureId = "pencil-grain",
                            sizeX = 1.0f,
                            sizeY = 1.0f,
                            sizeUnit = BrushPaint.TextureLayer.SizeUnit.BRUSH_SIZE,
                            blendMode = BrushPaint.TextureLayer.BlendMode.MODULATE,
                        ),
                    ),
                ),
            ),
        ),
    ),
)
```

> **Note:** The texture `"pencil-grain"` must be registered in your `TextureBitmapStore` for the texture to render. See `textures.md`.

---

## Recipe 4: Watercolor / Wet Paint

A wet-paint brush where speed controls opacity (fast = transparent). Uses `SelfOverlap.ANY` for blending and low base opacity.

**Behaviors:** speed → opacity

```kotlin
val watercolorBrush = BrushFamily(
    coats = listOf(
        BrushCoat(
            tip = BrushTip(
                scaleX = 1f,
                scaleY = 1f,
                cornerRounding = 1f,
                behaviors = listOf(
                    // Speed → Opacity: fast strokes fade
                    smoothedBehavior(
                        source = SourceNode.Source.SPEED_IN_MULTIPLES_OF_BRUSH_SIZE_PER_SECOND,
                        sourceStart = 0f, sourceEnd = 8f,
                        target = TargetNode.Target.OPACITY_MULTIPLIER,
                        targetStart = 1.0f, targetEnd = 0.2f,
                        dampingSeconds = 0.3f,
                    ),
                ),
            ),
            paintPreferences = listOf(
                BrushPaint(
                    selfOverlap = SelfOverlap.ANY,
                    colorFunctions = listOf(
                        BrushPaint.ColorFunction.OpacityMultiplier(0.6f),
                    ),
                ),
            ),
        ),
    ),
)
```

### Watercolor with Texture

Add a wash texture for more realistic watercolor appearance:

```kotlin
val watercolorWithTexture = BrushFamily(
    coats = listOf(
        BrushCoat(
            tip = BrushTip(
                scaleX = 1f,
                scaleY = 1f,
                cornerRounding = 1f,
                behaviors = listOf(
                    smoothedBehavior(
                        source = SourceNode.Source.SPEED_IN_MULTIPLES_OF_BRUSH_SIZE_PER_SECOND,
                        sourceStart = 0f, sourceEnd = 8f,
                        target = TargetNode.Target.OPACITY_MULTIPLIER,
                        targetStart = 1.0f, targetEnd = 0.2f,
                        dampingSeconds = 0.3f,
                    ),
                ),
            ),
            paintPreferences = listOf(
                BrushPaint(
                    selfOverlap = SelfOverlap.ANY,
                    colorFunctions = listOf(
                        BrushPaint.ColorFunction.OpacityMultiplier(0.6f),
                    ),
                    textureLayers = listOf(
                        BrushPaint.TilingTexture(
                            clientTextureId = "watercolor-wash",
                            sizeX = 2.0f,
                            sizeY = 2.0f,
                            sizeUnit = BrushPaint.TextureLayer.SizeUnit.BRUSH_SIZE,
                            blendMode = BrushPaint.TextureLayer.BlendMode.MODULATE,
                        ),
                    ),
                ),
            ),
        ),
    ),
)
```

---

## Recipe 5: Multi-Coat Layered Brush

A two-coat brush where the bottom coat creates a soft shadow and the top coat renders the main stroke. Demonstrates using multiple coats with different paints.

```kotlin
val layeredBrush = BrushFamily(
    coats = listOf(
        // Coat 0: Shadow layer (renders first, underneath)
        BrushCoat(
            tip = BrushTip(
                scaleX = 1.3f,   // Slightly wider than main stroke
                scaleY = 1.3f,   // Slightly taller than main stroke
                cornerRounding = 1f,
            ),
            paintPreferences = listOf(
                BrushPaint(
                    selfOverlap = SelfOverlap.DISCARD,
                    colorFunctions = listOf(
                        BrushPaint.ColorFunction.OpacityMultiplier(0.15f),  // Very faint shadow
                    ),
                ),
            ),
        ),
        // Coat 1: Main stroke layer (renders on top)
        BrushCoat(
            tip = BrushTip(
                scaleX = 1f,
                scaleY = 1f,
                cornerRounding = 1f,
                behaviors = listOf(
                    // Pressure → Size on main coat
                    smoothedBehavior(
                        source = SourceNode.Source.NORMALIZED_PRESSURE,
                        sourceStart = 0f, sourceEnd = 1f,
                        target = TargetNode.Target.SIZE_MULTIPLIER,
                        targetStart = 0.5f, targetEnd = 1.5f,
                        dampingSeconds = 0.15f,
                    ),
                ),
            ),
            paintPreferences = listOf(
                BrushPaint(
                    selfOverlap = SelfOverlap.DISCARD,
                    colorFunctions = listOf(
                        BrushPaint.ColorFunction.OpacityMultiplier(1f),
                    ),
                ),
            ),
        ),
    ),
)
```

**Key points about multi-coat brushes:**
- Coats render in list order: coat[0] is the bottom layer, coat[1] is on top.
- Each coat has its own independent tip (with its own behaviors) and paint.
- The shadow coat uses a larger tip scale (`1.3×`) and low opacity (`0.15`) to create a soft halo.
- Both coats use the same brush color and size from `Brush.createWithComposeColor()`, but their tip scales and paint opacities differ.

---

## Creating a Brush from a Recipe

All recipes produce `BrushFamily` objects directly. To use them for rendering, combine with a color and size:

```kotlin
import androidx.ink.brush.Brush
import androidx.ink.brush.compose.createWithComposeColor
import androidx.compose.ui.graphics.Color

val brush = Brush.createWithComposeColor(
    family = pressurePen,
    color = Color.Black,
    size = 15f,
    epsilon = 0.1f,
)
```

No conversion step is needed — `BrushFamily` is used directly with `Brush.createWithComposeColor()`.

## Common Pitfalls

- **Not registering textures** — Textured recipes (shading pencil, watercolor) require a `TextureBitmapStore` with the referenced texture IDs loaded.
- **Conflicting behaviors on same target** — Multiple behaviors targeting `TargetNode.Target.SIZE_MULTIPLIER` combine multiplicatively. Two behaviors that both range 0.5–1.5 can produce sizes from 0.25–2.25.
- **Shadow coat too visible** — Keep shadow coat opacity low (0.1–0.2). Higher values create an obvious double-stroke effect.
- **Not setting `epsilon`** — `Brush.createWithComposeColor()` requires an `epsilon` parameter. A value of `0.1f` works well for most cases.
- **Combining `SelfOverlap.DISCARD` with opacity/color behaviors or `StampingTexture`** — `SelfOverlap.DISCARD` forces path rendering (`CanvasPathRenderer`), which does not support per-vertex opacity (`OPACITY_MULTIPLIER`) or color (`HUE_OFFSET_IN_RADIANS`, `CHROMA_MULTIPLIER`, `LIGHTNESS_OFFSET`) `BrushBehavior`s or `BrushPaint.StampingTexture` on the same `BrushCoat`. Always use `SelfOverlap.ANY` (or `SelfOverlap.ACCUMULATE`) when a coat uses opacity/color behaviors, `StampingTexture`, or visible overlap blending (watercolor, shading pencil).
