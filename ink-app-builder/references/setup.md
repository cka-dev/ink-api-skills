# Setup & Dependencies

Reference for configuring an Android project to use the Jetpack Ink API with Compose.

## Required Dependencies

All Ink modules share a single version. Use a Gradle version catalog (`libs.versions.toml`) to pin them uniformly.

### Version Catalog (`gradle/libs.versions.toml`)

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

# Motion prediction (optional but recommended)
androidx-input-motionprediction = { module = "androidx.input:input-motionprediction", version.ref = "inputMotionPrediction" }
```

### `app/build.gradle.kts`

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

## Choosing an Ink Version (`1.0.0` Stable vs. `1.1.0-alpha`)

| Version | What's Supported | When to Use |
|---|---|---|
| **`1.0.0` (Stable)** | Full core inking pipeline: `InProgressStrokes`, `CanvasStrokeRenderer`, `StockBrushes`, geometry intersection (`eraser`), `StrokeInputBatch.encode()`/`decode()`, and loading pre-serialized brushes via `BrushFamily.decode()`. | Production apps using `StockBrushes` or pre-compiled `.brush` binary assets. |
| **`1.1.0-alpha03+`** | Everything in `1.0.0`, **plus** public Kotlin constructors for programmatic custom brushes (`BrushFamily(...)`, `BrushCoat`, `BrushTip`, `BrushPaint`, `BrushPaint.TilingTexture` / `StampingTexture`, `BrushPaint.ColorFunction`, `BrushBehavior`, and `androidx.ink.brush.behavior.*` nodes). | Apps that build or modify custom `BrushFamily` definitions in Kotlin code (see `ink-custom-brush-builder`). In `1.0.0` stable (and `1.1.0-alpha01`–`alpha02`), those constructors were `@RestrictTo(LIBRARY_GROUP)`. |

> **Note on `1.1.0-alpha03+`**: Starting in `1.1.0-alpha03`, `BrushFamily(...)`, `BrushCoat`, `BrushTip`, `BrushPaint`, `BrushBehavior`, `Version`, and the `androidx.ink.brush.behavior.*` node classes graduated from `@RestrictTo(LIBRARY_GROUP)` and `@ExperimentalInkCustomBrushApi` to public API, `BrushPaint.TextureLayer` became an abstract class with concrete subclasses `BrushPaint.TilingTexture` and `BrushPaint.StampingTexture`, `ColorFunction` was nested inside `BrushPaint.ColorFunction`, and `SlidingWindowModel` was nested inside `BrushFamily.InputModel.SlidingWindowModel`. Additionally, `@ExperimentalInkCustomBrushApi` is no longer required on `BrushFamily.encode()`/`decode()` or `AndroidBrushFamilySerialization` in `1.1.0-alpha02+` (it is only required on those serialization APIs in `1.0.0` stable, and in `1.1.0-alpha05+` the `ExperimentalInkCustomBrushApi` annotation itself is `@RestrictTo(LIBRARY_GROUP)`).

## Module Responsibilities

| Module | Purpose |
|---|---|
| `ink-authoring` | Core stroke-input machinery |
| `ink-authoring-android` | Android platform binding for authoring |
| `ink-authoring-compose` | `InProgressStrokes` composable for Compose |
| `ink-brush` | `Brush`, `BrushFamily`, `StockBrushes` types |
| `ink-brush-compose` | Compose color extension functions (`createWithComposeColor`, `copyWithComposeColor`, `composeColor`) |
| `ink-geometry` | Geometric primitives (`MutableSegment`, `MutableParallelogram`, `Intersection`, `AffineTransform`) |
| `ink-geometry-compose` | Compose interop for geometry types |
| `ink-nativeloader` | Loads native rendering libraries |
| `ink-rendering` | `CanvasStrokeRenderer` for drawing completed strokes to a Canvas |
| `ink-strokes` | `Stroke`, `StrokeInputBatch` data types |
| `ink-storage` | Serialization/deserialization of strokes and brush families (`encode`/`decode`, `AndroidBrushFamilySerialization`) |

## Min SDK Requirement

The Ink API requires **`minSdk = 26`** (Android 8.0) or higher.

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

## Common Pitfalls

- **Missing `ink-nativeloader`**: Without this module, the rendering engine will fail at runtime with a `UnsatisfiedLinkError`. Always include it.
- **Version mismatch across Ink modules**: All `ink-*` artifacts MUST use the same version. A version catalog with a single `ink` version ref prevents drift.
- **Missing Compose modules**: If you only add `ink-brush` but not `ink-brush-compose`, extension functions like `Brush.createWithComposeColor()` will be unavailable.
- **Missing `ink-authoring-android`**: The `ink-authoring-compose` module depends on `ink-authoring-android` at runtime. Omitting it causes `ClassNotFoundException`.
- **Forgetting motion prediction**: While optional, `androidx.input:input-motionprediction` significantly improves drawing smoothness. Include it for any production-quality app.
