# Ink Custom Brush Builder — Eval Results

**Date:** 2026-10-02
**Skill:** `ink-custom-brush-builder`
**Total Evals:** 8

---

## Summary

| Eval | Title | Score |
|------|-------|-------|
| 1 | Pressure-Responsive Brush | PASS |
| 2 | Custom Texture Brush | PASS |
| 3 | Tilt-Sensitive Calligraphy Brush | PASS |
| 4 | Multi-Coat Brush | PASS |
| 5 | Serialize Brush to File | PASS |
| 6 | Load Brush from File | PASS |
| 7 | Input Smoothing Configuration | PASS |
| 8 | Noise-Based Animated Brush | PASS |

**Overall: 8/8 PASS**

---

## Eval 1: Pressure-Responsive Brush

**Prompt**: "Create a brush that gets thicker when I press harder with the stylus"

**Reference docs consulted**: `brush-behaviors.md`, `brush-hierarchy.md`, `examples.md`

### Generated Response

```kotlin
import androidx.compose.ui.graphics.Color
import androidx.ink.brush.Brush
import androidx.ink.brush.BrushBehavior
import androidx.ink.brush.BrushCoat
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.BrushPaint
import androidx.ink.brush.BrushTip
import androidx.ink.brush.SelfOverlap
import androidx.ink.brush.behavior.DampingNode
import androidx.ink.brush.behavior.OutOfRange
import androidx.ink.brush.behavior.ProgressDomain
import androidx.ink.brush.behavior.SourceNode
import androidx.ink.brush.behavior.TargetNode
import androidx.ink.brush.compose.createWithComposeColor

fun createPressureResponsiveBrush(): Brush {
    val brushFamily = BrushFamily(
        coats = listOf(
            BrushCoat(
                tip = BrushTip(
                    scaleX = 1f,
                    scaleY = 1f,
                    cornerRounding = 1f,
                    behaviors = listOf(
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

    return Brush.createWithComposeColor(
        family = brushFamily,
        color = Color.Black,
        size = 15f,
        epsilon = 0.1f,
    )
}
```

**Score**: PASS

**Must Include checklist**:
- ✅ `BrushBehavior` with `terminalNode = TargetNode(...)`
- ✅ `SourceNode` using `SourceNode.Source.NORMALIZED_PRESSURE`
- ✅ `TargetNode` using `TargetNode.Target.SIZE_MULTIPLIER` or `TargetNode.Target.WIDTH_MULTIPLIER` — uses `TargetNode.Target.SIZE_MULTIPLIER`
- ✅ `sourceValueRangeStart = 0f`, `sourceValueRangeEnd = 1f`
- ✅ `targetModifierRangeStart` / `targetModifierRangeEnd` (`0.5f` to `1.5f`)
- ✅ `sourceOutOfRangeBehavior = OutOfRange.CLAMP`
- ✅ `BrushFamily`, `BrushCoat`, `BrushTip`, `BrushPaint` Kotlin constructors

**Must NOT Include checklist**:
- ✅ No raw `ink.proto.*` builders — correctly absent
- ✅ No manually reading pressure from `MotionEvent` and scaling stroke width — correctly absent
- ✅ No modifying brush size in a touch listener — correctly absent
- ✅ No post-processing stroke geometry — correctly absent

---

## Eval 2: Custom Texture Brush

**Prompt**: "Create a brush that uses a custom bitmap texture for its stroke"

**Reference docs consulted**: `textures.md`, `brush-paint.md`, `brush-hierarchy.md`

### Generated Response

```kotlin
import android.content.Context
import android.graphics.Bitmap
import android.graphics.BitmapFactory
import androidx.ink.brush.BrushCoat
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.BrushPaint
import androidx.ink.brush.BrushTip
import androidx.ink.brush.SelfOverlap
import androidx.ink.brush.TextureBitmapStore
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update

// 1. Implement TextureBitmapStore
class AppTextureBitmapStore(context: Context) : TextureBitmapStore {
    private val resources = context.resources

    private val textureResources: Map<String, Int> = mapOf(
        "pencil-grain" to R.drawable.pencil_grain,
        "watercolor-wash" to R.drawable.watercolor_wash,
    )

    private val loadedBitmaps = mutableMapOf<String, Bitmap>()

    private val _generation = MutableStateFlow(0)
    val generation = _generation.asStateFlow()

    override operator fun get(clientTextureId: String): Bitmap? {
        val id = normalizeId(clientTextureId)
        return loadedBitmaps.getOrPut(id) {
            textureResources[id]?.let { resId ->
                BitmapFactory.decodeResource(resources, resId)
            } ?: return null
        }
    }

    fun loadTexture(textureId: String, bitmap: Bitmap) {
        val id = normalizeId(textureId)
        loadedBitmaps[id] = bitmap
        _generation.update { it + 1 }
    }

    private fun normalizeId(clientTextureId: String): String =
        clientTextureId
            .removePrefix("ink://ink")
            .removePrefix("/texture:")
}

// 2. Build BrushFamily with BrushPaint.TilingTexture
fun createTexturedBrushFamily(
    textureId: String,
    bitmap: Bitmap,
    textureStore: AppTextureBitmapStore,
): BrushFamily {
    // Register the bitmap in the runtime texture store
    textureStore.loadTexture(textureId, bitmap)

    return BrushFamily(
        coats = listOf(
            BrushCoat(
                tip = BrushTip(
                    scaleX = 1f,
                    scaleY = 1f,
                    cornerRounding = 1f,
                ),
                paintPreferences = listOf(
                    BrushPaint(
                        selfOverlap = SelfOverlap.ANY,
                        colorFunctions = listOf(
                            BrushPaint.ColorFunction.OpacityMultiplier(0.8f),
                        ),
                        textureLayers = listOf(
                            BrushPaint.TilingTexture(
                                clientTextureId = textureId,
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
}
```

**Score**: PASS

**Must Include checklist**:
- ✅ `TextureBitmapStore` implementation with `override operator fun get(clientTextureId: String): Bitmap?`
- ✅ `BrushPaint.TilingTexture` with `clientTextureId`, `sizeX = 1.0f`, `sizeY = 1.0f`
- ✅ `blendMode = BrushPaint.TextureLayer.BlendMode.MODULATE`
- ✅ `sizeUnit = BrushPaint.TextureLayer.SizeUnit.BRUSH_SIZE`
- ✅ Adding texture layer to `BrushPaint(textureLayers = listOf(...))`
- ✅ Loading texture bitmaps from resources or runtime cache

**Must NOT Include checklist**:
- ✅ No raw `ink.proto.*` builders — correctly absent
- ✅ No Canvas bitmap stamping in a draw loop — correctly absent
- ✅ No `BitmapShader` or `Paint.shader` — correctly absent
- ✅ No drawing bitmap manually at each stroke point — correctly absent

---

## Eval 3: Tilt-Sensitive Calligraphy Brush

**Prompt**: "Create a calligraphy brush that changes width when I tilt the stylus"

**Reference docs consulted**: `brush-behaviors.md`, `brush-tip.md`, `examples.md`

### Generated Response

```kotlin
import androidx.ink.brush.BrushBehavior
import androidx.ink.brush.BrushCoat
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.BrushPaint
import androidx.ink.brush.BrushTip
import androidx.ink.brush.SelfOverlap
import androidx.ink.brush.behavior.DampingNode
import androidx.ink.brush.behavior.OutOfRange
import androidx.ink.brush.behavior.ProgressDomain
import androidx.ink.brush.behavior.SourceNode
import androidx.ink.brush.behavior.TargetNode

fun createCalligraphyBrush(): BrushFamily {
    return BrushFamily(
        coats = listOf(
            BrushCoat(
                tip = BrushTip(
                    scaleX = 0.3f,          // Narrow horizontal — creates calligraphy nib effect
                    scaleY = 1.0f,          // Full vertical height
                    cornerRounding = 0.2f,  // Slightly rounded rectangle
                    rotationDegrees = 45f,  // 45° nib angle (in degrees)
                    behaviors = listOf(
                        // Tilt → Width: tilting widens the stroke from 1× to 2.5×
                        BrushBehavior(
                            terminalNode = TargetNode(
                                target = TargetNode.Target.WIDTH_MULTIPLIER,
                                targetModifierRangeStart = 1.0f,
                                targetModifierRangeEnd = 2.5f,
                                input = DampingNode(
                                    dampOver = ProgressDomain.TIME_IN_SECONDS,
                                    strength = 0.1f,
                                    input = SourceNode(
                                        source = SourceNode.Source.TILT_IN_RADIANS,
                                        sourceValueRangeStart = 0f,
                                        sourceValueRangeEnd = (Math.PI / 2).toFloat(),
                                        sourceOutOfRangeBehavior = OutOfRange.CLAMP,
                                    ),
                                ),
                            ),
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
}
```

**Score**: PASS

**Must Include checklist**:
- ✅ `SourceNode` with `SourceNode.Source.TILT_IN_RADIANS`
- ✅ `TargetNode` with `TargetNode.Target.WIDTH_MULTIPLIER`
- ✅ Source range from `0f` to `(Math.PI / 2).toFloat()`
- ✅ `DampingNode` for smooth transitions — present with `0.1f` second strength
- ✅ Narrow tip shape (asymmetric `scaleX = 0.3f`, `scaleY = 1.0f`, `rotationDegrees = 45f`)

**Must NOT Include checklist**:
- ✅ No raw `ink.proto.*` builders — correctly absent
- ✅ No reading tilt from `MotionEvent.getAxisValue(AXIS_TILT)` — correctly absent
- ✅ No manually rotating a bitmap stamp — correctly absent

---

## Eval 4: Multi-Coat Brush

**Prompt**: "Create a brush with two layers — a base stroke and a shadow underneath"

**Reference docs consulted**: `brush-hierarchy.md`, `examples.md`

### Generated Response

```kotlin
import androidx.ink.brush.BrushCoat
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.BrushPaint
import androidx.ink.brush.BrushTip
import androidx.ink.brush.SelfOverlap

fun createMultiCoatBrush(): BrushFamily {
    return BrushFamily(
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
                            BrushPaint.ColorFunction.OpacityMultiplier(0.15f),  // Faint shadow
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
}
```

**Score**: PASS

**Must Include checklist**:
- ✅ `BrushFamily` with `coats = listOf(BrushCoat(...), BrushCoat(...))`
- ✅ Each `BrushCoat` having its own `BrushTip` and `BrushPaint`
- ✅ Different configurations per coat — shadow coat has larger scale (`1.3f`) and low opacity (`0.15f`); main coat has normal scale (`1f`) and full opacity (`1f`)
- ✅ Correct coat ordering (`coats[0]` = bottom shadow layer, `coats[1]` = top main layer)

**Must NOT Include checklist**:
- ✅ No raw `ink.proto.*` builders (`addCoats()`) — correctly absent
- ✅ No drawing two separate strokes — correctly absent
- ✅ No layered Canvas rendering — correctly absent
- ✅ No post-processing effects — correctly absent

---

## Eval 5: Serialize Brush to File

**Prompt**: "Save my custom brush to a file so I can load it later"

**Reference docs consulted**: `serialization.md`

### Generated Response

```kotlin
import android.content.Context
import android.net.Uri
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.TextureBitmapStore
import androidx.ink.storage.AndroidBrushFamilySerialization
import androidx.ink.storage.encode

// Option 1: Kotlin extension function on BrushFamily (writes GZIP-compressed bytes with embedded textures)
fun saveBrushFamilyToFile(
    context: Context,
    uri: Uri,  // e.g. "my_custom_brush.brush"
    brushFamily: BrushFamily,
    textureStore: TextureBitmapStore,
) {
    context.contentResolver.openOutputStream(uri)?.use { outputStream ->
        brushFamily.encode(outputStream, textureStore)
    }
}

// Option 2: AndroidBrushFamilySerialization
fun saveBrushFamilyAndroid(
    context: Context,
    uri: Uri,
    brushFamily: BrushFamily,
    textureStore: TextureBitmapStore,
) {
    context.contentResolver.openOutputStream(uri)?.use { outputStream ->
        AndroidBrushFamilySerialization.encode(
            brushFamily = brushFamily,
            output = outputStream,
            textureBitmapStore = textureStore,
        )
    }
}
```

**Score**: PASS

**Must Include checklist**:
- ✅ `BrushFamily.encode(outputStream, textureBitmapStore)` or `AndroidBrushFamilySerialization.encode(...)`
- ✅ `OutputStream` via `contentResolver.openOutputStream(uri)`
- ✅ Texture embedding support via `TextureBitmapStore` parameter

**Must NOT Include checklist**:
- ✅ No JSON serialization — correctly absent
- ✅ No Parcelable / Serializable — correctly absent
- ✅ No Proto text format — correctly absent
- ✅ No saving as XML — correctly absent

---

## Eval 6: Load Brush from File

**Prompt**: "Load a custom brush from a .brush file"

**Reference docs consulted**: `serialization.md`, `textures.md`

### Generated Response

```kotlin
import android.content.Context
import android.net.Uri
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.Version
import androidx.ink.storage.AndroidBrushFamilySerialization
import androidx.ink.storage.BrushFamilyDecodeCallback
import androidx.ink.storage.decode

fun loadBrushFamilyFromFile(
    context: Context,
    uri: Uri,  // points to a .brush file
    textureStore: AppTextureBitmapStore,
): BrushFamily? {
    return context.contentResolver.openInputStream(uri)?.use { inputStream ->
        BrushFamily.decode(
            input = inputStream,
            maxVersion = Version.DEVELOPMENT,
            getClientTextureId = BrushFamilyDecodeCallback { id, bitmap ->
                bitmap?.let { textureStore.loadTexture(id, it) }
                id  // Return the client texture ID
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
}
```

**Score**: PASS

**Must Include checklist**:
- ✅ `BrushFamily.decode(inputStream, maxVersion)` or `AndroidBrushFamilySerialization.decode(...)`
- ✅ `maxVersion = Version.DEVELOPMENT`
- ✅ Texture callback (`BrushFamilyDecodeCallback { id, bitmap -> bitmap?.let { textureStore.loadTexture(id, it) }; id }`)

**Must NOT Include checklist**:
- ✅ No `BitmapFactory.decodeStream()` as the primary decoder — correctly absent
- ✅ No manual `ink.proto.*` protobuf parsing — correctly absent
- ✅ No GSON / Moshi deserialization — correctly absent

---

## Eval 7: Input Smoothing Configuration

**Prompt**: "Configure my brush for smoother strokes with less jitter"

**Reference docs consulted**: `input-model.md`, `brush-behaviors.md`

### Generated Response

```kotlin
import androidx.ink.brush.BrushBehavior
import androidx.ink.brush.BrushCoat
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.BrushPaint
import androidx.ink.brush.BrushTip
import androidx.ink.brush.SelfOverlap
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
                    // DampingNode smooths noisy pressure/tilt changes over time
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
    // Configure SlidingWindowModel (default is 20ms window, 180Hz upsampling;
    // increasing windowDurationMillis to 30L produces smoother strokes with less jitter)
    inputModel = BrushFamily.InputModel.SlidingWindowModel(
        windowDurationMillis = 30L,
        upsamplingFrequencyHz = 180,
    ),
)
```

**Score**: PASS

**Must Include checklist**:
- ✅ `BrushFamily` `inputModel` (`BrushFamily.InputModel.SlidingWindowModel(windowDurationMillis = 30L, upsamplingFrequencyHz = 180)`)
- ✅ `DampingNode` with `dampOver = ProgressDomain.TIME_IN_SECONDS` and `strength` inside `BrushBehavior`
- ✅ Explanation of `windowDurationMillis` (`20L`–`30L` ms) and `upsamplingFrequencyHz` (`180` Hz)

**Must NOT Include checklist**:
- ✅ No manual Bézier curve smoothing — correctly absent
- ✅ No moving average filter in application code — correctly absent
- ✅ No post-processing stroke points — correctly absent

---

## Eval 8: Noise-Based Animated Brush

**Prompt**: "Create a brush where the stroke wobbles randomly for an organic hand-drawn feel"

**Reference docs consulted**: `brush-behaviors.md`, `examples.md`

### Generated Response

```kotlin
import androidx.ink.brush.BrushBehavior
import androidx.ink.brush.BrushCoat
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.BrushPaint
import androidx.ink.brush.BrushTip
import androidx.ink.brush.SelfOverlap
import androidx.ink.brush.behavior.NoiseNode
import androidx.ink.brush.behavior.ProgressDomain
import androidx.ink.brush.behavior.TargetNode

fun createWobblyBrush(): BrushFamily {
    return BrushFamily(
        coats = listOf(
            BrushCoat(
                tip = BrushTip(
                    scaleX = 1f,
                    scaleY = 1f,
                    cornerRounding = 1f,
                    behaviors = listOf(
                        // Noise → Slant: wobble effect of ±0.15 radians over distance
                        BrushBehavior(
                            terminalNode = TargetNode(
                                target = TargetNode.Target.SLANT_OFFSET_IN_RADIANS,
                                targetModifierRangeStart = -0.15f,
                                targetModifierRangeEnd = 0.15f,
                                input = NoiseNode(
                                    seed = kotlin.random.Random.nextInt(),
                                    varyOver = ProgressDomain.DISTANCE_IN_CENTIMETERS,
                                    basePeriod = 0.3f,
                                ),
                            ),
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
}
```

**Score**: PASS

**Must Include checklist**:
- ✅ `NoiseNode` with `seed`, `varyOver = ProgressDomain.DISTANCE_IN_CENTIMETERS`, `basePeriod`
- ✅ `TargetNode` with `TargetNode.Target.SLANT_OFFSET_IN_RADIANS` wrapping `input = NoiseNode(...)`
- ✅ Small target range for subtle effect (`-0.15f` to `0.15f` radians)

**Must NOT Include checklist**:
- ✅ No raw `ink.proto.*` builders — correctly absent
- ✅ No `Random.nextFloat()` in a draw loop — correctly absent
- ✅ No Perlin noise library — correctly absent
- ✅ No animating canvas offset — correctly absent
- ✅ No timer-based animation — correctly absent

---

## Overall Assessment

| Metric | Value |
|--------|-------|
| **Total Evals** | 8 |
| **PASS** | 8 |
| **PARTIAL** | 0 |
| **FAIL** | 0 |
| **Pass Rate** | 100% |

### Skill Documentation Quality

The `ink-custom-brush-builder` skill documentation is aligned end-to-end on the Kotlin `androidx.ink.brush.*` and `androidx.ink.brush.behavior.*` API:

1. **Task routing table** — `SKILL.md` routes directly to the relevant reference doc for each task.
2. **Complete code examples** — Every reference doc includes copy-ready Kotlin data-class constructor code with accurate imports.
3. **Anti-patterns documented** — `SKILL.md` explicitly prohibits raw `ink.proto.*` builders, manual `MotionEvent` pressure scaling, Canvas bitmap stamping, JSON serialization, and manual Bézier smoothing.
4. **Cross-reference coverage** — `examples.md` provides 5 complete recipes (pressure pen, calligraphy, shading pencil, watercolor, multi-coat) using tree-based `BrushBehavior` builders.
5. **All 11 behavior nodes documented** — `brush-behaviors.md` covers `SourceNode`, `ConstantNode`, `NoiseNode`, `ToolTypeFilterNode`, `DampingNode`, `ResponseNode`, `IntegralNode`, `BinaryOpNode`, `InterpolationNode`, `TargetNode`, and `PolarTargetNode`.
