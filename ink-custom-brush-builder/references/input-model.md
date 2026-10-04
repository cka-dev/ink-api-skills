# Input Model Configuration

## Overview

The `InputModel` controls how raw stylus/touch input is processed before reaching the brush's behavior graph. It sits at the `BrushFamily` level and affects all coats within the family.

Two models are available on `BrushFamily.InputModel`:
- **`BrushFamily.InputModel.SlidingWindowModel`** — Smooths input with a time-based window and optionally upsamples for higher-frequency rendering. This is the **recommended default** (`BrushFamily.InputModel.DEFAULT_INPUT_MODEL`).
- **`BrushFamily.InputModel.PASSTHROUGH_MODEL`** — Passes raw input directly without any smoothing or upsampling.

> **Note:** In the Kotlin `BrushFamily` constructor API (`1.1.0-alpha03+`), `inputModel` defaults to `BrushFamily.InputModel.DEFAULT_INPUT_MODEL`, which is a `BrushFamily.InputModel.SlidingWindowModel()` with default parameters (`windowDurationMillis = 20L`, `upsamplingFrequencyHz = 180`). You can also construct a custom `BrushFamily.InputModel.SlidingWindowModel(windowDurationMillis, upsamplingFrequencyHz)` or use `BrushFamily.InputModel.PASSTHROUGH_MODEL`, and combine it with `DampingNode` in `BrushBehavior` for per-property temporal/distance smoothing. (In legacy `@RestrictTo` `1.1.0-alpha02`, these were nested directly on `BrushFamily` as `BrushFamily.SlidingWindowModel`, `BrushFamily.DEFAULT_INPUT_MODEL`, and `BrushFamily.PASSTHROUGH_MODEL`.)

## Configuring `InputModel` and Stroke Smoothing in Code

```kotlin
import androidx.ink.brush.BrushCoat
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.BrushPaint
import androidx.ink.brush.BrushTip
import androidx.ink.brush.SelfOverlap
import androidx.ink.brush.BrushBehavior
import androidx.ink.brush.behavior.DampingNode
import androidx.ink.brush.behavior.OutOfRange
import androidx.ink.brush.behavior.ProgressDomain
import androidx.ink.brush.behavior.SourceNode
import androidx.ink.brush.behavior.TargetNode

val smoothBrushFamily = BrushFamily(
    coats = listOf(
        BrushCoat(
            tip = BrushTip(
                scaleX = 1f,
                scaleY = 1f,
                cornerRounding = 1f,
                behaviors = listOf(
                    // Smooth pressure transitions over a 150ms damping window
                    BrushBehavior(
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
                    ),
                ),
            ),
            paintPreferences = listOf(
                BrushPaint(selfOverlap = SelfOverlap.DISCARD),
            ),
        ),
    ),
    // Use BrushFamily.InputModel.DEFAULT_INPUT_MODEL (20ms window, 180Hz upsampling),
    // or customize with BrushFamily.InputModel.SlidingWindowModel(windowDurationMillis = 30L, upsamplingFrequencyHz = 180)
    inputModel = BrushFamily.InputModel.SlidingWindowModel(
        windowDurationMillis = 30L,
        upsamplingFrequencyHz = 180,
    ),
)
```

## SlidingWindowModel

`BrushFamily.InputModel.SlidingWindowModel` applies a sliding-window average to input signals, reducing jitter from noisy stylus/touch hardware. It also supports temporal upsampling for smoother stroke rendering.

### Constructors & Properties

```kotlin
import androidx.ink.brush.BrushFamily

// Default parameters (20ms window, 180Hz upsampling)
val defaultSlidingWindow = BrushFamily.InputModel.SlidingWindowModel()

// Custom parameters
val customSlidingWindow = BrushFamily.InputModel.SlidingWindowModel(
    windowDurationMillis = 30L,
    upsamplingFrequencyHz = 180,
)
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `windowDurationMillis` | `Long` | `20L` | Duration of the smoothing window in milliseconds (typically `1L`–`100L`). Larger values = smoother but more latency. |
| `upsamplingFrequencyHz` | `Int` | `180` | Minimum frequency (in Hz) at which modeled inputs should occur, or `0` to disable upsampling. |

### Typical Values

| Style | `windowDurationMillis` | `upsamplingFrequencyHz` | Character |
|-------|------------------------|-------------------------|-----------|
| Low-latency pen | `10L`–`15L` | `180` | Responsive, minimal latency |
| Standard brush (`DEFAULT_INPUT_MODEL`) | `20L` | `180` | Good balance of smoothness and responsiveness |
| Calligraphy | `30L` | `180`–`240` | Smooth curves, slight lag |
| Slow marker / Airbrush | `50L` | `180` | Very smooth, noticeable lag |

### Impact on Stroke Quality

- **Smaller `windowDurationMillis`** (e.g., `5L`–`15L`) → More responsive, captures rapid movements, but may show jitter from hardware noise.
- **Larger `windowDurationMillis`** (e.g., `30L`–`50L`) → Smoother curves, better for slow deliberate strokes, but introduces perceptible latency.
- **Higher `upsamplingFrequencyHz`** (e.g., `180`–`240`) → Higher rendering frequency, smoother visual curves, slightly higher CPU cost.
- **`upsamplingFrequencyHz = 0`** → Disables upsampling entirely, using only the raw input reporting rate.

## PassthroughModel (`BrushFamily.InputModel.PASSTHROUGH_MODEL`)

Bypasses all smoothing and upsampling (`inputModel = BrushFamily.InputModel.PASSTHROUGH_MODEL`). Raw input samples are passed directly to the behavior graph with only minimal modeling to derive velocity and acceleration.

Use `PASSTHROUGH_MODEL` when:
- You want maximum responsiveness with zero added latency
- The input device already provides clean, high-frequency data
- You are feeding pre-smoothed or synthetic inputs into Ink

## Common Pitfalls

- **Implementing manual Bézier or moving-average filters in app code** — Rely on `BrushFamily.InputModel.DEFAULT_INPUT_MODEL` / `BrushFamily.InputModel.SlidingWindowModel` for stroke path smoothing and `DampingNode` for property smoothing (pressure, tilt, speed).
- **Using non-existent `BrushFamily.SPRING_MODEL` or Protobuf field names** — `SPRING_MODEL` does not exist in `androidx.ink.brush.BrushFamily`, and the Kotlin `SlidingWindowModel` constructor takes `(windowDurationMillis: Long, upsamplingFrequencyHz: Int)` rather than Protobuf's `windowSizeSeconds` / `experimentalUpsamplingPeriodSeconds`.
- **Setting `windowDurationMillis` too high** — Values above `50L` ms create noticeable visual latency. Users perceive a disconnect between their stylus movement and the drawn stroke.
- **Using `PASSTHROUGH_MODEL` with un-damped pressure behaviors** — Raw pressure data from stylus hardware can be noisy. Without `SlidingWindowModel` or a `DampingNode`, pressure-based behaviors may produce jittery size/opacity changes.
- **Not testing on target hardware** — Different stylus devices report at different frequencies (60Hz–240Hz). The optimal smoothing window depends on the input device's reporting rate.
