# Texture Support

## Overview

Textures add visual richness to brush strokes — enabling effects like pencil grain, watercolor wash, and patterned stamps. The Ink API uses a `TextureBitmapStore` functional interface for runtime bitmap resolution and `BrushPaint.TextureLayer` subclasses (`BrushPaint.TilingTexture` and `BrushPaint.StampingTexture`) for configuration.

Key concepts:
- **`TextureBitmapStore`** — Functional interface your app implements. Maps `clientTextureId` strings to `Bitmap` objects at render time.
- **`BrushPaint.TilingTexture` / `BrushPaint.StampingTexture`** — Concrete subclasses of `BrushPaint.TextureLayer` (in `1.1.0-alpha03+`) describing how a texture is applied to paint (tiling vs. stamping, blend mode, size, wrap).

> **Note:** `TextureBitmapStore`, `BrushPaint.TilingTexture`, `BrushPaint.StampingTexture`, `AndroidBrushFamilySerialization`, and `BrushFamily.encode()`/`decode()` are public without opt-in in `1.1.0-alpha03+` (`@OptIn(ExperimentalInkCustomBrushApi::class)` is only required when calling `BrushFamily.encode()`/`decode()` or `AndroidBrushFamilySerialization` on `1.0.0` stable). (In legacy `@RestrictTo` `1.1.0-alpha02`, textures were constructed via `BrushPaint.TextureLayer(..., mapping = Mapping.TILING / STAMPING)`.)

## TextureBitmapStore

`TextureBitmapStore` is a functional interface. The simplest usage:

```kotlin
import android.graphics.Bitmap
import androidx.ink.brush.TextureBitmapStore

// Empty store (returns null for all texture IDs)
val emptyStore = TextureBitmapStore { null }

// Store backed by a map
val bitmapMap = mutableMapOf<String, Bitmap>()
val mapStore = TextureBitmapStore { id -> bitmapMap[id] }
```

### App-Level Texture Store

For production apps, wrap the functional interface with a class that manages texture loading and caching:

```kotlin
import android.content.Context
import android.graphics.Bitmap
import android.graphics.BitmapFactory
import androidx.ink.brush.TextureBitmapStore
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update

class AppTextureBitmapStore(context: Context) : TextureBitmapStore {
    private val resources = context.resources

    // Map texture string IDs to drawable resource IDs
    private val textureResources: Map<String, Int> = mapOf(
        "pencil-grain" to R.drawable.pencil_grain,
        "watercolor-wash" to R.drawable.watercolor_wash,
    )

    // In-memory cache of loaded bitmaps
    private val loadedBitmaps = mutableMapOf<String, Bitmap>()

    // Generation counter for reactive cache invalidation
    private val _generation = MutableStateFlow(0)
    val generation = _generation.asStateFlow()

    override operator fun get(clientTextureId: String): Bitmap? {
        val normalizedId = normalizeId(clientTextureId)
        return loadedBitmaps.getOrPut(normalizedId) {
            textureResources[normalizedId]?.let { resId ->
                BitmapFactory.decodeResource(resources, resId)
            } ?: return null
        }
    }

    /** Load a custom texture bitmap at runtime (e.g., from a URI or decoded brush). */
    fun loadTexture(textureId: String, bitmap: Bitmap) {
        val id = normalizeId(textureId)
        loadedBitmaps[id] = bitmap
        _generation.update { it + 1 }  // Signal observers to invalidate caches
    }

    /** Returns all available texture IDs (both built-in and dynamically loaded). */
    fun getAllIds(): Set<String> = textureResources.keys + loadedBitmaps.keys

    private fun normalizeId(clientTextureId: String): String =
        clientTextureId
            .removePrefix("ink://ink")
            .removePrefix("/texture:")
}
```

### Generation Counter Pattern

The `_generation` `StateFlow` increments every time a new texture is loaded. Because `CanvasStrokeRenderer` (and `InProgressStrokes`'s internal renderer) caches `Paint` / `BitmapShader` instances per `BrushPaint` on first draw, UI layers must recreate `CanvasStrokeRenderer` via `remember(textureStore, generation)` and re-key `InProgressStrokes` via `key(generation)` when textures become available:

```kotlin
import androidx.compose.runtime.getValue
import androidx.compose.runtime.key
import androidx.compose.runtime.remember
import androidx.ink.authoring.compose.InProgressStrokes
import androidx.ink.rendering.android.canvas.CanvasStrokeRenderer
import androidx.lifecycle.compose.collectAsStateWithLifecycle

val generation by textureStore.generation.collectAsStateWithLifecycle()
val renderer = remember(textureStore, generation) {
    CanvasStrokeRenderer.create(textureStore)
}

key(generation) {
    InProgressStrokes(
        defaultBrush = currentBrush,
        nextBrush = onGetNextBrush,
        textureBitmapStore = textureStore,
        onStrokesFinished = onStrokesFinished,
    )
}
```

## TextureLayer Configuration (`TilingTexture` & `StampingTexture`)

In `1.1.0-alpha03+`, `BrushPaint.TextureLayer` is an abstract base class with two concrete subclasses:
- **`BrushPaint.TilingTexture`**: Repeats the texture across vertex positions according to a 2D affine transformation (ideal for paper/pencil grain, canvas texture, watercolor wash).
- **`BrushPaint.StampingTexture`**: Stamps the texture onto each particle of the stroke (used with particle-based tips where `BrushTip.particleGapDistanceScale > 0f` or `particleGapDurationMillis > 0L`).

```kotlin
import androidx.ink.brush.BrushPaint

// 1. Tiling texture (repeats along the stroke)
val tilingTexture = BrushPaint.TilingTexture(
    clientTextureId = "pencil-grain",
    sizeX = 1.0f,
    sizeY = 1.0f,
    sizeUnit = BrushPaint.TextureLayer.SizeUnit.BRUSH_SIZE,
    blendMode = BrushPaint.TextureLayer.BlendMode.MODULATE,
)

// 2. Stamping texture (stamped onto each brush tip particle)
val stampingTexture = BrushPaint.StampingTexture(
    clientTextureId = "star-stamp",
    blendMode = BrushPaint.TextureLayer.BlendMode.MODULATE,
)

// Both TilingTexture and StampingTexture provide Kotlin copy(...) methods (and toBuilder()):
val rotatedTilingTexture = tilingTexture.copy(rotationDegrees = 45f)
val dstInStampingTexture = stampingTexture.copy(blendMode = BrushPaint.TextureLayer.BlendMode.DST_IN)
```

### `TilingTexture` Properties

| Property | Type / Default | Description |
|----------|----------------|-------------|
| `clientTextureId` | `String` | ID that `TextureBitmapStore` resolves to a `Bitmap`. |
| `sizeX` | `Float` (`> 0f`) | Horizontal texture size in `sizeUnit`. `1.0f` with `BRUSH_SIZE` = texture width matches brush size. |
| `sizeY` | `Float` (`> 0f`) | Vertical texture size in `sizeUnit`. `1.0f` with `BRUSH_SIZE` = texture height matches brush size. |
| `offsetX` / `offsetY` | `Float = 0f` | Offset into the texture as a fraction of `sizeX` / `sizeY`. |
| `rotationDegrees` | `Float = 0f` | Rotation angle of the texture in degrees. |
| `sizeUnit` | `BrushPaint.TextureLayer.SizeUnit = STROKE_COORDINATES` | `SizeUnit.BRUSH_SIZE` (relative to brush size) or `SizeUnit.STROKE_COORDINATES`. |
| `origin` | `BrushPaint.TilingTexture.Origin = STROKE_SPACE_ORIGIN` | `Origin.STROKE_SPACE_ORIGIN`, `Origin.FIRST_STROKE_INPUT`, or `Origin.LAST_STROKE_INPUT` (nested on `BrushPaint.TilingTexture.Origin`). |
| `wrapX` / `wrapY` | `BrushPaint.TextureLayer.Wrap = REPEAT` | `Wrap.REPEAT`, `Wrap.MIRROR`, or `Wrap.CLAMP`. |
| `blendMode` | `BrushPaint.TextureLayer.BlendMode = MODULATE` | How the texture combines with the next layer or brush color (`MODULATE`, `DST_IN`, `DST_OUT`, `SRC_ATOP`, `SRC_IN` for the final/only layer; `SRC_OVER`, `DST_OVER`, `SRC`, `DST`, `SRC_OUT`, `DST_ATOP`, `XOR` for intermediate layers). |

### `StampingTexture` Properties

| Property | Type / Default | Description |
|----------|----------------|-------------|
| `clientTextureId` | `String` | ID that `TextureBitmapStore` resolves to a `Bitmap`. |
| `blendMode` | `BrushPaint.TextureLayer.BlendMode = MODULATE` | How the stamped texture combines with the next layer or brush color. (Requires `CanvasMeshRenderer` on Android 14 / API 34+ hardware `Canvas`; incompatible with `SelfOverlap.DISCARD` and `CanvasPathRenderer`.) |

> **Fallback `BrushPaint` for `StampingTexture` on API ≤ 33 or Bitmap Export**: Because `CanvasPathRenderer` cannot render `StampingTexture` (`canDraw` returns `false`), a `BrushCoat` whose only `BrushPaint` uses `StampingTexture` will be skipped on Android 13 and below (API 26–33) and when exporting to a software `Canvas(bitmap)`. To ensure the coat still renders on older devices or bitmap canvases, provide a second fallback `BrushPaint` (e.g., using `TilingTexture` or a solid fill with `SelfOverlap.ANY`) in `BrushCoat(tip = ..., paintPreferences = listOf(stampingPaint, fallbackPaint))`.

### Choosing `BlendMode` (Final Layer vs. Intermediate Layers)

In `BrushPaint`, texture layers are blended sequentially (`layer[0]` → `layer[1]` → ... → `layer[n - 1]`), and the **final (or only) `TextureLayer`** blends the combined texture (`src`) with the brush color (`dst`, which already includes per-vertex `OPACITY_MULTIPLIER` and edge anti-aliasing alpha):

- **Final (or only) `TextureLayer`**: Must use a blend mode whose output alpha is proportional to `Alpha_dst` so that edge anti-aliasing and per-vertex `OPACITY_MULTIPLIER` behaviors are preserved:
  - `BlendMode.MODULATE` (default): `Color_src * Color_dst`, `Alpha_src * Alpha_dst` (tints grayscale/colored textures with the brush color and modulates alpha).
  - `BlendMode.DST_IN`: Keeps the brush color (`Color_dst`) while multiplying its alpha by the texture's alpha (`Alpha_src * Alpha_dst`).
  - `BlendMode.DST_OUT`: Keeps the brush color where the texture is transparent (`(1 - Alpha_src) * Alpha_dst`).
  - `BlendMode.SRC_ATOP` / `BlendMode.SRC_IN`: Uses texture color where the brush stroke is opaque while scaling alpha by `Alpha_dst`.
- **Intermediate `TextureLayer`s only** (when `textureLayers.size > 1`, before the last layer): Modes such as `BlendMode.SRC_OVER`, `DST_OVER`, `SRC`, `DST`, `SRC_OUT`, `DST_ATOP`, and `XOR` do not have output alpha proportional to `Alpha_dst` and must **not** be used on the final `TextureLayer` (doing so overrides anti-aliasing and per-vertex opacity).

## Adding a Texture to a Paint

Create a `BrushPaint` with texture layers using the `BrushPaint` constructor:

```kotlin
import androidx.ink.brush.BrushPaint
import androidx.ink.brush.SelfOverlap

val texturedPaint = BrushPaint(
    textureLayers = listOf(
        BrushPaint.TilingTexture(
            clientTextureId = "pencil-grain",
            sizeX = 1.0f,
            sizeY = 1.0f,
            sizeUnit = BrushPaint.TextureLayer.SizeUnit.BRUSH_SIZE,
            blendMode = BrushPaint.TextureLayer.BlendMode.MODULATE,
        ),
    ),
    colorFunctions = listOf(BrushPaint.ColorFunction.OpacityMultiplier(0.8f)),
    selfOverlap = SelfOverlap.ANY,
)
```

## Loading Textures During Deserialization

When decoding a `BrushFamily` that contains embedded textures, use `BrushFamily.decode()` or `AndroidBrushFamilySerialization.decode()` with an explicit `BrushFamilyDecodeCallback` (required in `1.1.0-alpha` to disambiguate from the `OnDecodeTexturePngBytes` overload):

```kotlin
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.Version
import androidx.ink.storage.AndroidBrushFamilySerialization
import androidx.ink.storage.BrushFamilyDecodeCallback
import androidx.ink.storage.decode
import java.io.InputStream

fun decodeFamilyWithTextures(
    inputStream: InputStream,
    textureStore: AppTextureBitmapStore,
): BrushFamily {
    return BrushFamily.decode(
        input = inputStream,
        maxVersion = Version.DEVELOPMENT,
        getClientTextureId = BrushFamilyDecodeCallback { id, bitmap ->
            bitmap?.let { textureStore.loadTexture(id, it) }
            id  // Return the ID to use for this texture
        },
    )
    // Or via AndroidBrushFamilySerialization.decode(
    //     input = inputStream,
    //     maxVersion = Version.DEVELOPMENT,
    //     getClientTextureId = BrushFamilyDecodeCallback { id, bitmap ->
    //         bitmap?.let { textureStore.loadTexture(id, it) }
    //         id
    //     },
    // )
}
```

## Common Pitfalls

- **Not implementing a `TextureBitmapStore`** — The renderer calls the store at draw time. Without a store, textured brushes render as plain strokes.
- **Using `BlendMode.SRC_OVER` on the final (or only) `TextureLayer`** — `SRC_OVER` output alpha (`Alpha_src + (1 - Alpha_src) * Alpha_dst`) is not proportional to `Alpha_dst`, which overrides edge anti-aliasing and ignores `OPACITY_MULTIPLIER` behaviors wherever the texture is opaque. Use `BlendMode.MODULATE` (default), `DST_IN`, `DST_OUT`, `SRC_ATOP`, or `SRC_IN` on the final `TextureLayer`.
- **Using `StampingTexture` without a fallback `BrushPaint` in `paintPreferences`** — `BrushPaint.StampingTexture` requires `CanvasMeshRenderer` (Android 14 / API 34+ hardware `Canvas`) and cannot be rendered by `CanvasPathRenderer` (API ≤ 33 or software `Canvas(bitmap)`). Provide a fallback `BrushPaint` in `BrushCoat.paintPreferences` if the coat must also render on API ≤ 33 or when exporting to a `Bitmap`.
- **Instantiating abstract `BrushPaint.TextureLayer(...)` directly** — In `1.1.0-alpha03+`, `BrushPaint.TextureLayer` is an abstract class. Instantiate `BrushPaint.TilingTexture(...)` or `BrushPaint.StampingTexture(...)` instead (`SizeUnit`, `BlendMode`, and `Wrap` remain nested on `BrushPaint.TextureLayer.*`, while `Origin` is nested on `BrushPaint.TilingTexture.Origin`).
- **Not normalizing texture IDs** — The Ink API may prefix IDs with `ink://ink/texture:`. Strip this prefix in `TextureBitmapStore` to match internal IDs.
- **Oversized bitmap textures** — Textures are tiled along the stroke, so high-resolution bitmaps waste memory. Keep textures at 256×256 or 512×512 pixels.
- **Not incrementing the generation counter** — After loading new textures, UI components won't re-render unless notified. Increment a generation counter or use another reactive signal.
- **Forgetting `@OptIn(ExperimentalInkCustomBrushApi::class)` on `1.0.0` stable** — Required when decoding/encoding pre-serialized brushes on `1.0.0` stable (not required in `1.1.0-alpha02+`).
