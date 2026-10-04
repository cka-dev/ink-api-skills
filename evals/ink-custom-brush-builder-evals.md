# Ink Custom Brush Builder — Eval Prompts

Test prompts to validate the `ink-custom-brush-builder` skill produces correct
custom brush code using the Ink API (`1.1.0-alpha03+`).

---

## Eval 1: Pressure-Responsive Brush

**Prompt**: "Create a brush that gets thicker when I press harder with the stylus"

**Must Include**:
- `BrushBehavior` with `terminalNode = TargetNode(...)`
- `SourceNode` using `SourceNode.Source.NORMALIZED_PRESSURE`
- `TargetNode` using `TargetNode.Target.SIZE_MULTIPLIER` or `TargetNode.Target.WIDTH_MULTIPLIER`
- `sourceValueRangeStart = 0f`, `sourceValueRangeEnd = 1f`
- `targetModifierRangeStart` / `targetModifierRangeEnd` (e.g., 0.5 to 1.5)
- `sourceOutOfRangeBehavior = OutOfRange.CLAMP`
- `BrushFamily`, `BrushCoat`, `BrushTip`, `BrushPaint` Kotlin constructors

**Must NOT Include**:
- ❌ Raw `ink.proto.*` builders (`ProtoBrushFamily.newBuilder()`)
- ❌ Manually reading pressure from `MotionEvent` and scaling stroke width
- ❌ Modifying brush size in a touch listener
- ❌ Post-processing stroke geometry

---

## Eval 2: Custom Texture Brush

**Prompt**: "Create a brush that uses a custom bitmap texture for its stroke"

**Must Include**:
- `TextureBitmapStore` implementation (`operator fun get(clientTextureId: String): Bitmap?` or SAM lambda)
- `BrushPaint.TilingTexture` (or `BrushPaint.StampingTexture`) with `clientTextureId`, `sizeX = 1.0f`, `sizeY = 1.0f`
- `blendMode = BrushPaint.TextureLayer.BlendMode.MODULATE` (or another final-layer blend mode whose output alpha is proportional to destination alpha, such as `DST_IN` / `SRC_ATOP`)
- `sizeUnit = BrushPaint.TextureLayer.SizeUnit.BRUSH_SIZE`
- Adding texture layer to `BrushPaint(textureLayers = listOf(...))`
- Loading texture bitmaps from resources or runtime cache

**Must NOT Include**:
- ❌ Raw `ink.proto.*` builders
- ❌ Canvas bitmap stamping in a draw loop
- ❌ `BitmapShader` or `Paint.shader`
- ❌ Drawing bitmap manually at each stroke point

---

## Eval 3: Tilt-Sensitive Calligraphy Brush

**Prompt**: "Create a calligraphy brush that changes width when I tilt the stylus"

**Must Include**:
- `SourceNode` with `SourceNode.Source.TILT_IN_RADIANS`
- `TargetNode` with `TargetNode.Target.WIDTH_MULTIPLIER`
- Source range from `0f` to `(Math.PI / 2).toFloat()`
- `DampingNode` for smooth transitions (optional but recommended)
- Narrow tip shape (asymmetric `scaleX` / `scaleY` and `rotationDegrees` on `BrushTip`)

**Must NOT Include**:
- ❌ Raw `ink.proto.*` builders
- ❌ Reading tilt from `MotionEvent.getAxisValue(AXIS_TILT)`
- ❌ Manually rotating a bitmap stamp

---

## Eval 4: Multi-Coat Brush

**Prompt**: "Create a brush with two layers — a base stroke and a shadow underneath"

**Must Include**:
- `BrushFamily` with `coats = listOf(BrushCoat(...), BrushCoat(...))`
- Each `BrushCoat` having its own `BrushTip` and `BrushPaint`
- Different configurations per coat (e.g., shadow coat with larger scale and low `BrushPaint.ColorFunction.OpacityMultiplier`, main coat on top)
- Correct coat ordering (`coats[0]` = bottom shadow layer, `coats[1]` = top main layer)

**Must NOT Include**:
- ❌ Raw `ink.proto.*` builders (`addCoats()`)
- ❌ Drawing two separate strokes
- ❌ Layered Canvas rendering
- ❌ Post-processing effects

---

## Eval 5: Serialize Brush to File

**Prompt**: "Save my custom brush to a file so I can load it later"

**Must Include**:
- `BrushFamily.encode(outputStream, textureBitmapStore)` or `AndroidBrushFamilySerialization.encode(...)`
- `OutputStream` (e.g., via `contentResolver.openOutputStream(uri)` or `ByteArrayOutputStream`)
- Texture embedding support via `TextureBitmapStore` parameter

**Must NOT Include**:
- ❌ JSON serialization
- ❌ Parcelable / Serializable
- ❌ Proto text format
- ❌ Saving as XML

---

## Eval 6: Load Brush from File

**Prompt**: "Load a custom brush from a .brush file"

**Must Include**:
- `BrushFamily.decode(inputStream, maxVersion)` or `AndroidBrushFamilySerialization.decode(...)`
- `maxVersion = Version.DEVELOPMENT` (or `Version.MAX_SUPPORTED`)
- Texture callback (`BrushFamilyDecodeCallback { id, bitmap -> bitmap?.let { textureStore.loadTexture(id, it) }; id }`)

**Must NOT Include**:
- ❌ `BitmapFactory.decodeStream()` as the primary decoder
- ❌ Manual `ink.proto.*` protobuf parsing
- ❌ GSON / Moshi deserialization

---

## Eval 7: Input Smoothing Configuration

**Prompt**: "Configure my brush for smoother strokes with less jitter"

**Must Include**:
- `BrushFamily` `inputModel` (`BrushFamily.InputModel.SlidingWindowModel(windowDurationMillis = ..., upsamplingFrequencyHz = ...)` or `BrushFamily.InputModel.DEFAULT_INPUT_MODEL`)
- `DampingNode` with `dampOver = ProgressDomain.TIME_IN_SECONDS` and `strength` (or `dampingSource`/`dampingGap` in `1.1.0-alpha03`–`alpha07`) inside `BrushBehavior` for smooth property transitions
- Explanation of `windowDurationMillis` (e.g., `20L`–`30L` ms) and `upsamplingFrequencyHz` (e.g., `180` Hz)

**Must NOT Include**:
- ❌ Manual Bézier curve smoothing
- ❌ Moving average filter in application code
- ❌ Post-processing stroke points

---

## Eval 8: Noise-Based Animated Brush

**Prompt**: "Create a brush where the stroke wobbles randomly for an organic hand-drawn feel"

**Must Include**:
- `NoiseNode` with `seed`, `varyOver = ProgressDomain.DISTANCE_IN_CENTIMETERS`, `basePeriod`
- `TargetNode` with `TargetNode.Target.SLANT_OFFSET_IN_RADIANS` or `TargetNode.Target.WIDTH_MULTIPLIER` (wrapping `input = NoiseNode(...)`)
- Small target range for subtle effect (e.g., `-0.15f` to `0.15f` radians)

**Must NOT Include**:
- ❌ Raw `ink.proto.*` builders
- ❌ `Random.nextFloat()` in a draw loop
- ❌ Perlin noise library
- ❌ Animating canvas offset
- ❌ Timer-based animation
