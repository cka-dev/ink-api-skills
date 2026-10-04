# Brush Behaviors

## Overview

A `BrushBehavior` is a tree-based node graph that dynamically modifies brush properties during stroke rendering. The graph flows: **Source → [Operators] → Target**, where nodes chain via the `input` parameter.

Behaviors are attached to a `BrushTip` and are evaluated per-input-point. A tip can have multiple behaviors, each operating independently.

Each behavior wraps a single `terminalNode` (a `TerminalNode` — either `TargetNode` or `PolarTargetNode`), which chains backwards through operator nodes to one or more source/value nodes. This forms a tree: `BrushBehavior(terminalNode = TargetNode(input = DampingNode(input = SourceNode(...))))`.

## All 11 Node Types

### 1. SourceNode

Reads an input signal from stylus/touch input. Normalizes the raw value into the range defined by `sourceValueRangeStart` and `sourceValueRangeEnd`.

```kotlin
import androidx.ink.brush.behavior.SourceNode
import androidx.ink.brush.behavior.OutOfRange

val source = SourceNode(
    source = SourceNode.Source.NORMALIZED_PRESSURE,
    sourceValueRangeStart = 0f,
    sourceValueRangeEnd = 1f,
    sourceOutOfRangeBehavior = OutOfRange.CLAMP,
)
```

**Available Sources (All 37 Public `SourceNode.Source` Constants):**

#### Stylus & Touch Orientation / Pressure

| Source | Typical Range | Description |
|--------|--------------|-------------|
| `SourceNode.Source.NORMALIZED_PRESSURE` | `[0, 1]` | Stylus or touch pressure (0 = no contact, 1 = max) |
| `SourceNode.Source.TILT_IN_RADIANS` | `[0, π/2]` | Stylus tilt angle from perpendicular |
| `SourceNode.Source.TILT_X_IN_RADIANS` | `[-π/2, π/2]` | Stylus tilt along the X axis (requires both tilt and orientation) |
| `SourceNode.Source.TILT_Y_IN_RADIANS` | `[-π/2, π/2]` | Stylus tilt along the Y axis (requires both tilt and orientation) |
| `SourceNode.Source.ORIENTATION_IN_RADIANS` | `[0, 2π)` | Stylus barrel orientation in `[0, 2π)` |
| `SourceNode.Source.ORIENTATION_ABOUT_ZERO_IN_RADIANS` | `(-π, π]` | Stylus barrel orientation centered around zero in `(-π, π]` |

#### Speed, Velocity & Direction

| Source | Typical Range | Description |
|--------|--------------|-------------|
| `SourceNode.Source.SPEED_IN_MULTIPLES_OF_BRUSH_SIZE_PER_SECOND` | `[0, ∞)` | Absolute speed in multiples of brush size per second |
| `SourceNode.Source.VELOCITY_X_IN_MULTIPLES_OF_BRUSH_SIZE_PER_SECOND` | `(-∞, ∞)` | Signed X velocity in multiples of brush size per second |
| `SourceNode.Source.VELOCITY_Y_IN_MULTIPLES_OF_BRUSH_SIZE_PER_SECOND` | `(-∞, ∞)` | Signed Y velocity in multiples of brush size per second |
| `SourceNode.Source.SPEED_IN_CENTIMETERS_PER_SECOND` | `[0, ∞)` | Absolute speed in centimeters per second |
| `SourceNode.Source.VELOCITY_X_IN_CENTIMETERS_PER_SECOND` | `(-∞, ∞)` | Signed X velocity in centimeters per second |
| `SourceNode.Source.VELOCITY_Y_IN_CENTIMETERS_PER_SECOND` | `(-∞, ∞)` | Signed Y velocity in centimeters per second |
| `SourceNode.Source.DIRECTION_IN_RADIANS` | `[0, 2π)` | Direction of travel in stroke space (`0` = +X, `π/2` = +Y) |
| `SourceNode.Source.DIRECTION_ABOUT_ZERO_IN_RADIANS` | `(-π, π]` | Direction of travel in stroke space in `(-π, π]` |
| `SourceNode.Source.NORMALIZED_DIRECTION_X` | `[-1, 1]` | Signed X component of normalized direction of travel |
| `SourceNode.Source.NORMALIZED_DIRECTION_Y` | `[-1, 1]` | Signed Y component of normalized direction of travel |

#### Acceleration

| Source | Typical Range | Description |
|--------|--------------|-------------|
| `SourceNode.Source.ACCELERATION_IN_MULTIPLES_OF_BRUSH_SIZE_PER_SECOND_SQUARED` | `[0, ∞)` | Absolute acceleration in multiples of brush size/s² |
| `SourceNode.Source.ACCELERATION_X_IN_MULTIPLES_OF_BRUSH_SIZE_PER_SECOND_SQUARED` | `(-∞, ∞)` | Signed X acceleration in multiples of brush size/s² |
| `SourceNode.Source.ACCELERATION_Y_IN_MULTIPLES_OF_BRUSH_SIZE_PER_SECOND_SQUARED` | `(-∞, ∞)` | Signed Y acceleration in multiples of brush size/s² |
| `SourceNode.Source.ACCELERATION_FORWARD_IN_MULTIPLES_OF_BRUSH_SIZE_PER_SECOND_SQUARED` | `(-∞, ∞)` | Signed forward acceleration along velocity in multiples of brush size/s² |
| `SourceNode.Source.ACCELERATION_LATERAL_IN_MULTIPLES_OF_BRUSH_SIZE_PER_SECOND_SQUARED` | `(-∞, ∞)` | Signed lateral acceleration perpendicular to velocity in multiples of brush size/s² |
| `SourceNode.Source.ACCELERATION_IN_CENTIMETERS_PER_SECOND_SQUARED` | `[0, ∞)` | Absolute acceleration in cm/s² |
| `SourceNode.Source.ACCELERATION_X_IN_CENTIMETERS_PER_SECOND_SQUARED` | `(-∞, ∞)` | Signed X acceleration in cm/s² |
| `SourceNode.Source.ACCELERATION_Y_IN_CENTIMETERS_PER_SECOND_SQUARED` | `(-∞, ∞)` | Signed Y acceleration in cm/s² |
| `SourceNode.Source.ACCELERATION_FORWARD_IN_CENTIMETERS_PER_SECOND_SQUARED` | `(-∞, ∞)` | Signed forward acceleration along velocity in cm/s² |
| `SourceNode.Source.ACCELERATION_LATERAL_IN_CENTIMETERS_PER_SECOND_SQUARED` | `(-∞, ∞)` | Signed lateral acceleration perpendicular to velocity in cm/s² |

#### Distance & Time

| Source | Typical Range | Description |
|--------|--------------|-------------|
| `SourceNode.Source.DISTANCE_TRAVELED_IN_MULTIPLES_OF_BRUSH_SIZE` | `[0, ∞)` | Cumulative distance from stroke start in multiples of brush size |
| `SourceNode.Source.DISTANCE_TRAVELED_IN_CENTIMETERS` | `[0, ∞)` | Cumulative distance from stroke start in centimeters |
| `SourceNode.Source.DISTANCE_REMAINING_IN_MULTIPLES_OF_BRUSH_SIZE` | `[0, ∞)` | Remaining distance to current stroke end in multiples of brush size |
| `SourceNode.Source.DISTANCE_REMAINING_AS_FRACTION_OF_STROKE_LENGTH` | `[0, 1]` | Remaining distance to current stroke end as a fraction of total stroke length |
| `SourceNode.Source.PREDICTED_DISTANCE_TRAVELED_IN_MULTIPLES_OF_BRUSH_SIZE` | `[0, ∞)` | Distance traveled in predicted stroke portion in multiples of brush size |
| `SourceNode.Source.PREDICTED_DISTANCE_TRAVELED_IN_CENTIMETERS` | `[0, ∞)` | Distance traveled in predicted stroke portion in centimeters |
| `SourceNode.Source.TIME_OF_INPUT_IN_SECONDS` | `[0, ∞)` | Elapsed time from stroke start to this input in seconds |
| `SourceNode.Source.TIME_FROM_INPUT_TO_STROKE_END_IN_SECONDS` | `[0, ∞)` | Elapsed time from this input to last input in stroke in seconds |
| `SourceNode.Source.PREDICTED_TIME_ELAPSED_IN_SECONDS` | `[0, ∞)` | Elapsed prediction time from last real input in seconds |
| `SourceNode.Source.TIME_SINCE_INPUT_IN_SECONDS` | `[0, ∞)` | Live time elapsed since this input in seconds (requires `OutOfRange.CLAMP`) |
| `SourceNode.Source.TIME_SINCE_STROKE_END_IN_SECONDS` | `[0, ∞)` | Live time elapsed since final stroke input in seconds (requires `OutOfRange.CLAMP`) |

### 2. ConstantNode

Outputs a fixed float value. Useful as one input to a `BinaryOpNode` or `InterpolationNode`.

```kotlin
import androidx.ink.brush.behavior.ConstantNode

val constant = ConstantNode(0.5f)
```

### 3. NoiseNode

Generates procedural noise for organic, randomized variation.

```kotlin
import androidx.ink.brush.behavior.NoiseNode
import androidx.ink.brush.behavior.ProgressDomain

val noise = NoiseNode(
    seed = kotlin.random.Random.nextInt(),
    varyOver = ProgressDomain.DISTANCE_IN_CENTIMETERS,
    basePeriod = 0.3f,
)
```

| Property | Description |
|----------|-------------|
| `seed` | Random seed for the noise generator. Different seeds produce different patterns. |
| `varyOver` | `ProgressDomain` controlling how noise evolves (`TIME_IN_SECONDS`, `DISTANCE_IN_CENTIMETERS`, or `DISTANCE_IN_MULTIPLES_OF_BRUSH_SIZE`). |
| `basePeriod` | Base wavelength of the noise pattern in the domain's units. Smaller = more rapid variation. |

### 4. ToolTypeFilterNode

Enables or disables the behavior per input tool type. Passes through the input signal only if the current tool type is in the enabled set.

```kotlin
import androidx.ink.brush.InputToolType
import androidx.ink.brush.behavior.SourceNode
import androidx.ink.brush.behavior.ToolTypeFilterNode

val source = SourceNode(
    source = SourceNode.Source.NORMALIZED_PRESSURE,
    sourceValueRangeStart = 0f,
    sourceValueRangeEnd = 1f,
)
val filtered = ToolTypeFilterNode(
    enabledToolTypes = setOf(InputToolType.STYLUS),
    input = source,
)
```

**Tool types (`InputToolType`):** Use `setOf()` to combine types: `InputToolType.STYLUS`, `InputToolType.TOUCH`, `InputToolType.MOUSE`, `InputToolType.UNKNOWN`.

### 5. DampingNode

Smooths the input signal over time or distance for gradual transitions. Reduces jitter and creates more fluid property changes. Takes a `ValueNode` input via the `input` parameter.

```kotlin
import androidx.ink.brush.behavior.DampingNode
import androidx.ink.brush.behavior.OutOfRange
import androidx.ink.brush.behavior.ProgressDomain
import androidx.ink.brush.behavior.SourceNode

val source = SourceNode(
    source = SourceNode.Source.NORMALIZED_PRESSURE,
    sourceValueRangeStart = 0f,
    sourceValueRangeEnd = 1f,
    sourceOutOfRangeBehavior = OutOfRange.CLAMP,
)
val damped = DampingNode(
    dampOver = ProgressDomain.TIME_IN_SECONDS,
    strength = 0.15f,
    input = source,
)
```

| Property | Description |
|----------|-------------|
| `dampOver` | `ProgressDomain` over which to smooth (`TIME_IN_SECONDS`, `DISTANCE_IN_CENTIMETERS`, or `DISTANCE_IN_MULTIPLES_OF_BRUSH_SIZE`; named `dampingSource` in `1.1.0-alpha03`–`alpha07`). |
| `strength` | Smoothing window size / damping strength in the domain's units. Larger = smoother but more lag (named `dampingGap` in `1.1.0-alpha03`–`alpha07`). |

### 6. ResponseNode

Applies a non-linear response (easing) curve to a `ValueNode` input (e.g., making light stylus pressure ramp up more gently or aggressively).

```kotlin
import androidx.ink.brush.behavior.EasingFunction
import androidx.ink.brush.behavior.ResponseNode
import androidx.ink.brush.behavior.SourceNode

val source = SourceNode(
    source = SourceNode.Source.NORMALIZED_PRESSURE,
    sourceValueRangeStart = 0f,
    sourceValueRangeEnd = 1f,
)
val curved = ResponseNode(
    responseCurve = EasingFunction.Predefined.EASE_IN_OUT,
    input = source,
)
```

**Available `EasingFunction` options:**
- **Predefined curves (`EasingFunction.Predefined`):** `LINEAR`, `EASE`, `EASE_IN`, `EASE_OUT`, `EASE_IN_OUT`, `STEP_START`, `STEP_END`
- **Custom cubic Bézier (`EasingFunction.CubicBezier`):** `EasingFunction.CubicBezier(p1 = ImmutableVec(x1, y1), p2 = ImmutableVec(x2, y2))` (`p1.x`, `p2.x` in `[0f, 1f]`)
- **Piecewise linear (`EasingFunction.Linear`):** `EasingFunction.Linear(points = listOf(ImmutableVec(0.5f, 0.25f)))` (interior points between `(0, 0)` and `(1, 1)`)
- **Step function (`EasingFunction.Steps`):** `EasingFunction.Steps(stepCount = 4, stepPosition = EasingFunction.StepPosition.JUMP_END)` (`JUMP_END`, `JUMP_START`, `JUMP_BOTH`, `JUMP_NONE`)

### 7. IntegralNode

Integrates (accumulates) the input signal over a progress domain. Takes a `ValueNode` input via the `input` parameter.

```kotlin
import androidx.ink.brush.behavior.IntegralNode
import androidx.ink.brush.behavior.OutOfRange
import androidx.ink.brush.behavior.ProgressDomain
import androidx.ink.brush.behavior.SourceNode

val source = SourceNode(
    source = SourceNode.Source.NORMALIZED_PRESSURE,
    sourceValueRangeStart = 0f,
    sourceValueRangeEnd = 1f,
)
val integral = IntegralNode(
    integrateOver = ProgressDomain.TIME_IN_SECONDS,
    integralValueRangeStart = 0f,
    integralValueRangeEnd = 1f,
    integralOutOfRangeBehavior = OutOfRange.CLAMP,
    input = source,
)
```

### 8. BinaryOpNode

Combines two value nodes with an arithmetic or null-coalescing operation. Takes two inputs: `firstInput` and `secondInput`.

```kotlin
import androidx.ink.brush.behavior.BinaryOpNode
import androidx.ink.brush.behavior.ConstantNode
import androidx.ink.brush.behavior.SourceNode

val a = SourceNode(
    source = SourceNode.Source.NORMALIZED_PRESSURE,
    sourceValueRangeStart = 0f,
    sourceValueRangeEnd = 1f,
)
val b = ConstantNode(0.5f)
val product = BinaryOpNode(
    operation = BinaryOpNode.BinaryOp.PRODUCT,
    firstInput = a,
    secondInput = b,
)
```

**Operations (`BinaryOpNode.BinaryOp`):**
- `PRODUCT` (`A * B`), `SUM` (`A + B`), `MIN` (`min(A, B)`), `MAX` (`max(A, B)`)
- `AND_THEN` (`null` if `firstInput` is `null`, else `secondInput`)
- `OR_ELSE` (`firstInput` if non-`null`, else `secondInput`)
- `XOR_ELSE` (whichever input is non-`null` if exactly one is non-`null`, else `null`)

### 9. InterpolationNode

Interpolates between `startInput` and `endInput` using `paramInput` (`LERP` or `INVERSE_LERP`).

```kotlin
import androidx.ink.brush.behavior.ConstantNode
import androidx.ink.brush.behavior.InterpolationNode
import androidx.ink.brush.behavior.SourceNode

val source = SourceNode(
    source = SourceNode.Source.NORMALIZED_PRESSURE,
    sourceValueRangeStart = 0f,
    sourceValueRangeEnd = 1f,
)
val interpolated = InterpolationNode(
    interpolation = InterpolationNode.Interpolation.LERP,
    paramInput = source,
    startInput = ConstantNode(0.2f),
    endInput = ConstantNode(1.0f),
)
```

### 10. TargetNode

Applies the input signal to a scalar brush property. This is the primary terminal node of a behavior chain. Takes a `ValueNode` input via the `input` parameter.

```kotlin
import androidx.ink.brush.behavior.SourceNode
import androidx.ink.brush.behavior.TargetNode

val source = SourceNode(
    source = SourceNode.Source.NORMALIZED_PRESSURE,
    sourceValueRangeStart = 0f,
    sourceValueRangeEnd = 1f,
)
val target = TargetNode(
    target = TargetNode.Target.SIZE_MULTIPLIER,
    targetModifierRangeStart = 0.5f,
    targetModifierRangeEnd = 1.5f,
    input = source,
)
```

The input (0–1 normalized) is mapped to the range `[targetModifierRangeStart, targetModifierRangeEnd]` and applied as a multiplier or offset.

**Available Scalar Targets (16 constants):**

| Target | Type | Effect |
|--------|------|--------|
| `TargetNode.Target.SIZE_MULTIPLIER` | Multiplier | Overall tip size |
| `TargetNode.Target.WIDTH_MULTIPLIER` | Multiplier | Horizontal tip dimension |
| `TargetNode.Target.HEIGHT_MULTIPLIER` | Multiplier | Vertical tip dimension |
| `TargetNode.Target.SLANT_OFFSET_IN_RADIANS` | Offset | Tip slant/shear angle |
| `TargetNode.Target.ROTATION_OFFSET_IN_RADIANS` | Offset | Tip rotation angle |
| `TargetNode.Target.CORNER_ROUNDING_OFFSET` | Offset | Tip corner rounding |
| `TargetNode.Target.PINCH_OFFSET` | Offset | Tip pinch factor |
| `TargetNode.Target.OPACITY_MULTIPLIER` | Multiplier | Stroke opacity |
| `TargetNode.Target.HUE_OFFSET_IN_RADIANS` | Offset | Color hue shift |
| `TargetNode.Target.CHROMA_MULTIPLIER` | Multiplier | Color chroma/saturation (named `SATURATION_MULTIPLIER` in `1.1.0-alpha03`–`alpha07`) |
| `TargetNode.Target.LIGHTNESS_OFFSET` | Offset | Color perceived lightness (named `LUMINOSITY_OFFSET` in `1.1.0-alpha03`–`alpha07`) |
| `TargetNode.Target.PAINT_ANIMATION_PROGRESS_OFFSET` | Offset | Animated `BrushPaint` progress offset (`@ExperimentalInkAnimationApi`; named `TEXTURE_ANIMATION_PROGRESS_OFFSET` in `1.1.0-alpha03`–`alpha07`) |
| `TargetNode.Target.POSITION_OFFSET_X_IN_MULTIPLES_OF_BRUSH_SIZE` | Offset | Horizontal position offset |
| `TargetNode.Target.POSITION_OFFSET_Y_IN_MULTIPLES_OF_BRUSH_SIZE` | Offset | Vertical position offset |
| `TargetNode.Target.POSITION_OFFSET_FORWARD_IN_MULTIPLES_OF_BRUSH_SIZE` | Offset | Forward position offset along stroke |
| `TargetNode.Target.POSITION_OFFSET_LATERAL_IN_MULTIPLES_OF_BRUSH_SIZE` | Offset | Lateral position scatter |

### 11. PolarTargetNode

An alternative terminal node (`TerminalNode`) that applies a polar vector (angle + magnitude) to a 2D vector target property (such as position offset).

**Available Polar Targets:**
- `PolarTargetNode.PolarTarget.POSITION_OFFSET_ABSOLUTE_IN_RADIANS_AND_MULTIPLES_OF_BRUSH_SIZE` (angle relative to positive X-axis)
- `PolarTargetNode.PolarTarget.POSITION_OFFSET_RELATIVE_IN_RADIANS_AND_MULTIPLES_OF_BRUSH_SIZE` (angle relative to stroke travel direction)

```kotlin
import androidx.ink.brush.behavior.NoiseNode
import androidx.ink.brush.behavior.PolarTargetNode
import androidx.ink.brush.behavior.ProgressDomain
import androidx.ink.brush.behavior.SourceNode

val polarTarget = PolarTargetNode(
    target = PolarTargetNode.PolarTarget.POSITION_OFFSET_ABSOLUTE_IN_RADIANS_AND_MULTIPLES_OF_BRUSH_SIZE,
    angleRangeStart = 0f,
    angleRangeEnd = (2 * Math.PI).toFloat(),
    angleInput = NoiseNode(
        seed = 101,
        varyOver = ProgressDomain.DISTANCE_IN_CENTIMETERS,
        basePeriod = 0.2f,
    ),
    magnitudeRangeStart = 0f,
    magnitudeRangeEnd = 0.5f,
    magnitudeInput = SourceNode(
        source = SourceNode.Source.NORMALIZED_PRESSURE,
        sourceValueRangeStart = 0f,
        sourceValueRangeEnd = 1f,
    ),
)
```

## OutOfRange Behavior

Controls what happens when a source value falls outside `[sourceValueRangeStart, sourceValueRangeEnd]`:

| Value | Behavior | Best For |
|-------|----------|----------|
| `OutOfRange.CLAMP` | Clamps to nearest boundary | Most sources (pressure, tilt, speed) |
| `OutOfRange.REPEAT` | Wraps around cyclically | Cyclic sources (direction, orientation) |
| `OutOfRange.MIRROR` | Ping-pong mirror at boundaries | Oscillating effects |

## ProgressDomain Values

| Value | Description |
|-------|-------------|
| `ProgressDomain.TIME_IN_SECONDS` | Progress measured in elapsed seconds |
| `ProgressDomain.DISTANCE_IN_CENTIMETERS` | Progress measured in distance traveled (cm) |
| `ProgressDomain.DISTANCE_IN_MULTIPLES_OF_BRUSH_SIZE` | Progress measured in multiples of the brush size |

## Common Patterns

### Pattern 1: Direct — Source → Target

The simplest behavior: one source signal maps directly to one target property.

```kotlin
import androidx.ink.brush.BrushBehavior
import androidx.ink.brush.behavior.SourceNode
import androidx.ink.brush.behavior.TargetNode
import androidx.ink.brush.behavior.OutOfRange

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
```

### Pattern 2: Smoothed — Source → Damping → Target

Adds damping between source and target for smooth, lag-free transitions.

```kotlin
import androidx.ink.brush.BrushBehavior
import androidx.ink.brush.behavior.DampingNode
import androidx.ink.brush.behavior.OutOfRange
import androidx.ink.brush.behavior.ProgressDomain
import androidx.ink.brush.behavior.SourceNode
import androidx.ink.brush.behavior.TargetNode

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
```

### Pattern 3: Jitter — Noise → Target

Procedural noise creates organic variation without requiring input signals.

```kotlin
import androidx.ink.brush.BrushBehavior
import androidx.ink.brush.behavior.NoiseNode
import androidx.ink.brush.behavior.ProgressDomain
import androidx.ink.brush.behavior.TargetNode

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

## Complete Examples

### Pressure → Size (smoothed)

Stylus pressure scales the tip from 50% to 150%, with 0.15s damping:

```kotlin
import androidx.ink.brush.behavior.SourceNode
import androidx.ink.brush.behavior.TargetNode

val pressureToSize = smoothedBehavior(
    source = SourceNode.Source.NORMALIZED_PRESSURE,
    sourceStart = 0f, sourceEnd = 1f,
    target = TargetNode.Target.SIZE_MULTIPLIER,
    targetStart = 0.5f, targetEnd = 1.5f,
    dampingSeconds = 0.15f,
)
```

### Tilt → Width (smoothed)

Tilting the stylus widens the stroke from 1× to 2.5×:

```kotlin
import androidx.ink.brush.behavior.SourceNode
import androidx.ink.brush.behavior.TargetNode

val tiltToWidth = smoothedBehavior(
    source = SourceNode.Source.TILT_IN_RADIANS,
    sourceStart = 0f, sourceEnd = (Math.PI / 2).toFloat(),
    target = TargetNode.Target.WIDTH_MULTIPLIER,
    targetStart = 1.0f, targetEnd = 2.5f,
    dampingSeconds = 0.1f,
)
```

### Speed → Opacity (smoothed)

Fast strokes fade to 20% opacity:

```kotlin
import androidx.ink.brush.behavior.SourceNode
import androidx.ink.brush.behavior.TargetNode

val speedToOpacity = smoothedBehavior(
    source = SourceNode.Source.SPEED_IN_MULTIPLES_OF_BRUSH_SIZE_PER_SECOND,
    sourceStart = 0f, sourceEnd = 8f,
    target = TargetNode.Target.OPACITY_MULTIPLIER,
    targetStart = 1.0f, targetEnd = 0.2f,
    dampingSeconds = 0.3f,
)
```

### Noise → Slant Jitter

Pencil-like slant wobble of ±0.15 radians:

```kotlin
import androidx.ink.brush.behavior.TargetNode

val slantJitter = jitterBehavior(
    target = TargetNode.Target.SLANT_OFFSET_IN_RADIANS,
    targetStart = -0.15f, targetEnd = 0.15f,
    basePeriod = 0.3f,
)
```

### Adding Behaviors to a BrushTip

```kotlin
import androidx.ink.brush.BrushBehavior
import androidx.ink.brush.BrushTip

val tip = BrushTip(
    scaleX = 1f,
    scaleY = 1f,
    cornerRounding = 1f,
    behaviors = listOf(pressureToSize, tiltToWidth, speedToOpacity),
)
```

### Complete Behavior Pipeline

Building a full behavior from scratch, inline:

```kotlin
import androidx.ink.brush.BrushBehavior
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
```

## Common Pitfalls

- **Tree structure matters** — Nodes chain via `input` parameters. The `TerminalNode` (`TargetNode` or `PolarTargetNode`) is always the outermost node, wrapping operators that wrap sources. Reversing the nesting order produces incorrect behavior.
- **Forgetting `sourceOutOfRangeBehavior`** — The default may not match your intent. Always set it explicitly.
- **Using `OutOfRange.CLAMP` for direction** — Direction wraps around 2π. Use `OutOfRange.REPEAT` for cyclic sources.
- **`targetModifierRangeStart == targetModifierRangeEnd`** — This produces a constant value regardless of input, effectively disabling the behavior.
- **Excessive damping `strength` (>0.5s, or `dampingGap` in `1.1.0-alpha03`–`alpha07`)** — Creates noticeable latency between input and visual response.
- **Not providing a seed for NoiseNode** — Without a unique seed, noise patterns may repeat identically across strokes.
- **Stacking conflicting behaviors** — Multiple behaviors targeting the same property combine multiplicatively (for multipliers) or additively (for offsets). Unintended stacking can produce extreme values.
