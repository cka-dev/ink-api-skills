# Brush Tip Configuration

## Overview

`BrushTip` defines the geometric shape of the brush mark at each point along the stroke. It controls the tip's size, aspect ratio, corner shape, and static rotation. Behaviors (dynamic modifications) are also attached to the tip via the `behaviors` parameter.

## Properties

| Property | Type | Range | Default | Description |
|----------|------|-------|---------|-------------|
| `scaleX` | Float | 0+ (at least one of `scaleX`/`scaleY` > 0) | 1.0 | Horizontal scale of the tip shape. Multiplied by brush `size`. |
| `scaleY` | Float | 0+ (at least one of `scaleX`/`scaleY` > 0) | 1.0 | Vertical scale of the tip shape. Multiplied by brush `size`. |
| `cornerRounding` | Float | 0–1 | 1.0 | Shape interpolation: 0 = sharp rectangle/parallelogram, 1 = fully rounded (circle/pill). |
| `slantDegrees` | Float | −90.0–90.0 (degrees) | 0.0 | Slant/shear angle of the tip shape in **degrees** prior to `rotationDegrees`. |
| `pinch` | Float | 0–1 | 0.0 | Pinches the top two corners together to create trapezoidal/triangular shapes (0 = rectangle/parallelogram, 1 = triangle). |
| `rotationDegrees` | Float | degrees | 0.0 | Static rotation offset of the tip shape in **degrees**. |
| `particleGapDistanceScale` | Float | 0+ | 0.0 | Distance between discrete stamp particles as a multiple of brush size (0 = continuous stroke). |
| `particleGapDurationMillis` | Long | 0+ | 0L | Time interval in ms between discrete stamp particles (0 = continuous stroke). |
| `behaviors` | List | — | emptyList() | List of `BrushBehavior` graphs for dynamic property modification. |

## How ScaleX / ScaleY Control Tip Shape

The tip shape is an ellipse (or rectangle, depending on `cornerRounding`) whose dimensions are `scaleX × size` horizontally and `scaleY × size` vertically.

| Configuration | scaleX | scaleY | Result |
|---------------|--------|--------|--------|
| Circular tip | 1.0 | 1.0 | Uniform round mark |
| Wide flat stroke | 2.0 | 0.5 | Broad horizontal mark |
| Tall narrow stroke | 0.3 | 1.0 | Thin vertical mark |
| Large uniform | 2.0 | 2.0 | Double-size round mark |

## Corner Rounding Progression

`cornerRounding` interpolates the base shape between a rectangle and an ellipse:

| Value | Shape |
|-------|-------|
| 0.0 | Sharp rectangular corners |
| 0.25 | Slightly rounded rectangle |
| 0.5 | Rounded rectangle (stadium shape) |
| 0.75 | Nearly elliptical |
| 1.0 | Fully circular/elliptical **(default)** |

> **Note:** When `scaleX == scaleY`, `slantDegrees == 0f`, and `pinch == 0f`, `cornerRounding = 1f` produces a circle while `cornerRounding = 0f` produces a square. Rectangular/pill differences are most pronounced when `scaleX ≠ scaleY`.

## Code Examples

### Default Tip Configuration

```kotlin
import androidx.ink.brush.BrushTip

val defaultTip = BrushTip(
    scaleX = 1f,
    scaleY = 1f,
    cornerRounding = 1f,
)
```

### Calligraphy-Style Narrow Tip

```kotlin
import androidx.ink.brush.BrushTip

val calligraphyTip = BrushTip(
    scaleX = 0.3f,          // Narrow horizontal
    scaleY = 1.0f,          // Full vertical height
    cornerRounding = 0.2f,  // Slightly rounded rectangle
    rotationDegrees = 45f,  // 45° nib angle (in degrees)
)
```

### Flat Marker Tip

```kotlin
import androidx.ink.brush.BrushTip

val markerTip = BrushTip(
    scaleX = 2.0f,         // Wide
    scaleY = 0.5f,         // Short
    cornerRounding = 0.4f, // Rounded rectangle
)
```

### Tip with Dynamic Behaviors

Behaviors are attached directly via the `behaviors` parameter:

```kotlin
import androidx.ink.brush.BrushBehavior
import androidx.ink.brush.BrushTip
import androidx.ink.brush.behavior.SourceNode
import androidx.ink.brush.behavior.DampingNode
import androidx.ink.brush.behavior.TargetNode
import androidx.ink.brush.behavior.OutOfRange
import androidx.ink.brush.behavior.ProgressDomain

val behavior = BrushBehavior(
    terminalNode = TargetNode(
        target = TargetNode.Target.SIZE_MULTIPLIER,
        targetModifierRangeStart = 0.5f,
        targetModifierRangeEnd = 1.5f,
        input = DampingNode(
            dampOver = ProgressDomain.TIME_IN_SECONDS,
            strength = 0.15f,
            input = SourceNode(
                source = SourceNode.Source.NORMALIZED_PRESSURE,
                sourceValueRangeStart = 0f,
                sourceValueRangeEnd = 1f,
                sourceOutOfRangeBehavior = OutOfRange.CLAMP,
            ),
        ),
    ),
)

val tipWithBehavior = BrushTip(
    scaleX = 1f,
    scaleY = 1f,
    cornerRounding = 1f,
    behaviors = listOf(behavior),
)
```

### Modifying a Tip (`copy`)

`BrushTip` is immutable and provides a Kotlin `copy(...)` method (and `toBuilder()` for Java callers) with default arguments for all 9 tip properties:

```kotlin
val angledCalligraphyTip = calligraphyTip.copy(
    rotationDegrees = 30f,
    pinch = 0.25f,
)
```

## Common Pitfalls

- **Setting both `scaleX` and `scaleY` to 0** — At least one of `scaleX` or `scaleY` must be strictly greater than zero. Setting either to `0f` collapses the tip in that dimension.
- **`BrushTip` uses degrees, while `BrushBehavior` uses radians** — On `BrushTip`, static angles are specified in **degrees** (`slantDegrees = 15f`, `rotationDegrees = 45f`). In `BrushBehavior` nodes (`SourceNode.Source.TILT_IN_RADIANS`, `TargetNode.Target.ROTATION_OFFSET_IN_RADIANS`, `TargetNode.Target.SLANT_OFFSET_IN_RADIANS`), dynamic angles are in **radians** (`(Math.PI / 4).toFloat()`). Do not mix them up or use proto field names (`rotation`, `slant`).
- **Passing `opacityMultiplier` to `BrushTip`** — `BrushTip` does not have an `opacityMultiplier` constructor parameter. Set static coat opacity via `BrushPaint.ColorFunction.OpacityMultiplier(alpha)` on `BrushPaint`, or dynamic opacity via `TargetNode.Target.OPACITY_MULTIPLIER` in a `BrushBehavior`.
- **Not considering dynamic behaviors** — Static tip properties set the baseline. `BrushBehavior` graphs (attached via `behaviors = listOf(...)`) dynamically modify these properties at runtime via multipliers and offsets.
