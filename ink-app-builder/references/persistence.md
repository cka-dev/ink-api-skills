# Persistence

Reference for serializing and deserializing Ink strokes for persistent storage.

## Overview

The Ink API provides serialization primitives through the `ink-storage` module. Strokes are broken into two components for serialization:

1. **Stroke inputs** (`StrokeInputBatch`) — the raw input points, serialized via `encode()`/`decode()` extension functions.
2. **Brush data** — the `BrushFamily`, color, size, and epsilon, serialized either via `AndroidBrushFamilySerialization` (for custom brushes) or via a mapping to known enum values (for stock brushes).

## Serialization Strategy

### Stroke → ByteArray → JSON → Room Database

```
Stroke
  ├── StrokeInputBatch → encode() → ByteArray
  └── Brush
       ├── BrushFamily → Stock brush enum or AndroidBrushFamilySerialization.encode() → ByteArray
       ├── color (Long)
       ├── size (Float)
       └── epsilon (Float)
```

### Serialized Data Classes

```kotlin
import kotlinx.serialization.Serializable

@Serializable
data class SerializedStroke(
    val inputs: ByteArray,          // Encoded StrokeInputBatch
    val brush: SerializedBrush
)

@Serializable
data class SerializedBrush(
    val size: Float,
    val color: Long,                // Color stored as Long (colorLong format)
    val epsilon: Float,
    val stockBrush: SerializedStockBrush,
    val clientBrushFamilyId: String? = null  // For custom brush lookup
)

@Serializable
enum class SerializedStockBrush {
    MarkerLatest,
    PressurePenLatest,
    HighlighterLatest,
    DashedLineLatest,
    EmojiHighlighterHeartLatest,
    EmojiHighlighterStarLatest,
    EmojiHighlighterPoopLatest,
}
```

## Encoding (Stroke → Serialized)

### StrokeInputBatch Encoding

```kotlin
import androidx.ink.storage.encode
import androidx.ink.strokes.Stroke
import java.io.ByteArrayOutputStream

fun encodeInputs(stroke: Stroke): ByteArray {
    return ByteArrayOutputStream().use { outputStream ->
        stroke.inputs.encode(outputStream)
        outputStream.toByteArray()
    }
}
```

> **Note (`1.1.0-alpha` Direct `ByteArray` Overloads)**: In `1.0.0` (Stable), `StrokeInputBatch.encode(outputStream)` / `StrokeInputBatch.decode(inputStream)` and `BrushFamily.encode(outputStream)` / `BrushFamily.decode(inputStream)` only accept `OutputStream` / `InputStream`. In `1.1.0-alpha`, `androidx.ink.storage` adds direct `ByteArray` overloads (`stroke.inputs.encode(): ByteArray`, `StrokeInputBatch.decode(input: ByteArray)`, `brushFamily.encode(): ByteArray`, and `BrushFamily.decode(input: ByteArray)`), while keeping the stream overloads compatible with both versions.

### Brush Encoding (Stock Brushes)

Map stock brush families to enum values for compact storage:

```kotlin
import android.os.Build
import androidx.ink.brush.Brush
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.StockBrushes

private val supportsMiniEmojiTrail = Build.VERSION.SDK_INT >= Build.VERSION_CODES.UPSIDE_DOWN_CAKE

private val stockBrushToEnum = mapOf(
    StockBrushes.marker()       to SerializedStockBrush.MarkerLatest,
    StockBrushes.pressurePen()  to SerializedStockBrush.PressurePenLatest,
    StockBrushes.highlighter()  to SerializedStockBrush.HighlighterLatest,
    StockBrushes.dashedLine()   to SerializedStockBrush.DashedLineLatest,
    StockBrushes.emojiHighlighter(clientTextureId = "emoji-heart", showMiniEmojiTrail = supportsMiniEmojiTrail) to SerializedStockBrush.EmojiHighlighterHeartLatest,
    StockBrushes.emojiHighlighter(clientTextureId = "emoji-star", showMiniEmojiTrail = supportsMiniEmojiTrail)  to SerializedStockBrush.EmojiHighlighterStarLatest,
    StockBrushes.emojiHighlighter(clientTextureId = "emoji-poop", showMiniEmojiTrail = supportsMiniEmojiTrail)  to SerializedStockBrush.EmojiHighlighterPoopLatest,
)

fun serializeBrush(
    brush: Brush,
    customBrushes: Map<String, BrushFamily> = emptyMap(),
): SerializedBrush {
    val customBrushId = customBrushes.entries.firstOrNull { it.value == brush.family }?.key
    return SerializedBrush(
        size = brush.size,
        color = brush.colorLong,
        epsilon = brush.epsilon,
        stockBrush = stockBrushToEnum[brush.family] ?: SerializedStockBrush.MarkerLatest,
        clientBrushFamilyId = customBrushId,
    )
}
```

> **Note**: Avoid calling `brush.family.clientBrushFamilyId` directly — that property is annotated `@RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)` across `1.0.0` and `1.1.0-alpha`. Looking up the ID from your app's `customBrushes: Map<String, BrushFamily>` (or serializing the `BrushFamily` via `BrushFamily.encode()`) uses only public APIs. Also ensure the `showMiniEmojiTrail` argument passed to `StockBrushes.emojiHighlighter(...)` in `stockBrushToEnum` matches the value used when creating the brush (`true` adds three mini-emoji trail `BrushCoat`s — 5 coats total vs. 2 when `false` — which affects `BrushFamily.equals()`).

### Full Stroke Serialization

```kotlin
import androidx.ink.brush.BrushFamily
import androidx.ink.storage.encode
import androidx.ink.strokes.Stroke
import java.io.ByteArrayOutputStream
import kotlinx.serialization.encodeToString
import kotlinx.serialization.json.Json

fun serializeStroke(
    stroke: Stroke,
    customBrushes: Map<String, BrushFamily> = emptyMap(),
): String {
    val encodedInputs = ByteArrayOutputStream().use { outputStream ->
        stroke.inputs.encode(outputStream)
        outputStream.toByteArray()
    }
    val serializedStroke = SerializedStroke(
        inputs = encodedInputs,
        brush = serializeBrush(stroke.brush, customBrushes)
    )
    return Json.encodeToString(serializedStroke)
}
```

## Decoding (Serialized → Stroke)

### StrokeInputBatch Decoding

`StrokeInputBatch.decode()` returns an `ImmutableStrokeInputBatch`:

```kotlin
import androidx.ink.storage.decode
import androidx.ink.strokes.ImmutableStrokeInputBatch
import androidx.ink.strokes.StrokeInputBatch
import java.io.ByteArrayInputStream

fun decodeInputs(data: ByteArray): ImmutableStrokeInputBatch {
    return ByteArrayInputStream(data).use { inputStream ->
        StrokeInputBatch.decode(inputStream)
    }
}
```

### Brush Decoding

```kotlin
import androidx.ink.brush.Brush
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.StockBrushes

private val enumToStockBrush = stockBrushToEnum.entries.associate { (k, v) -> v to k }

fun deserializeBrush(
    serialized: SerializedBrush,
    customBrushes: Map<String, BrushFamily> = emptyMap(),
): Brush {
    val customFamily = serialized.clientBrushFamilyId?.let { customBrushes[it] }
    val family = customFamily
        ?: enumToStockBrush[serialized.stockBrush]
        ?: StockBrushes.marker()
    return Brush.createWithColorLong(
        family = family,
        colorLong = serialized.color,
        size = serialized.size,
        epsilon = serialized.epsilon
    )
}
```

### Full Stroke Deserialization

```kotlin
import androidx.ink.brush.BrushFamily
import androidx.ink.strokes.Stroke
import kotlinx.serialization.json.Json

fun deserializeStroke(
    data: String,
    customBrushes: Map<String, BrushFamily> = emptyMap(),
): Stroke? {
    return try {
        val serialized = Json.decodeFromString<SerializedStroke>(data)
        val inputs = decodeInputs(serialized.inputs)
        val brush = deserializeBrush(serialized.brush, customBrushes)
        Stroke(brush = brush, inputs = inputs)
    } catch (e: Exception) {
        null // Handle corrupted data gracefully
    }
}
```

## Custom Brush Family Serialization

For non-stock brush families, there are two API styles:

1. **Kotlin extension functions** on `BrushFamily` — `encode()` / `decode()` (from `androidx.ink.storage`)
2. **Java-friendly static methods** — `AndroidBrushFamilySerialization.encode()` / `.decode()`

> **Note on `@OptIn(ExperimentalInkCustomBrushApi::class)`**: In **`1.0.0` (Stable)**, `BrushFamily.encode()`/`decode()`, `AndroidBrushFamilySerialization`, and `BrushFamilyDecodeCallback` are annotated `@ExperimentalInkCustomBrushApi` and require `@OptIn(ExperimentalInkCustomBrushApi::class)` (`import androidx.ink.brush.ExperimentalInkCustomBrushApi`). In **`1.1.0-alpha02+`**, these serialization methods are unannotated and require no opt-in (in `1.1.0-alpha05+`, `ExperimentalInkCustomBrushApi` itself is `@RestrictTo(LIBRARY_GROUP)`, so omit the `@OptIn` when targeting `1.1.0-alpha03+`).

### Extension Function Style (Idiomatic Kotlin)

#### Encoding (without textures)

```kotlin
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.ExperimentalInkCustomBrushApi // 1.0.0 stable only
import androidx.ink.storage.encode
import java.io.ByteArrayOutputStream

@OptIn(ExperimentalInkCustomBrushApi::class) // Required in 1.0.0 stable; omit in 1.1.0-alpha03+
fun encodeBrushFamilySimple(family: BrushFamily): ByteArray {
    return ByteArrayOutputStream().use { stream ->
        family.encode(stream)
        stream.toByteArray()
    }
}
```

#### Encoding (with TextureBitmapStore)

`BrushFamily.encode()` accepts an optional `TextureBitmapStore` parameter:

```kotlin
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.ExperimentalInkCustomBrushApi // 1.0.0 stable only
import androidx.ink.brush.TextureBitmapStore
import androidx.ink.storage.encode
import java.io.ByteArrayOutputStream

@OptIn(ExperimentalInkCustomBrushApi::class) // Required in 1.0.0 stable; omit in 1.1.0-alpha03+
fun encodeBrushFamilyWithTextures(
    family: BrushFamily,
    textureStore: TextureBitmapStore
): ByteArray {
    return ByteArrayOutputStream().use { stream ->
        family.encode(stream, textureStore)
        stream.toByteArray()
    }
}
```

#### Decoding (without textures)

```kotlin
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.ExperimentalInkCustomBrushApi // 1.0.0 stable only
import androidx.ink.storage.decode
import java.io.ByteArrayInputStream

@OptIn(ExperimentalInkCustomBrushApi::class) // Required in 1.0.0 stable; omit in 1.1.0-alpha03+
fun decodeBrushFamily(data: ByteArray): BrushFamily {
    return ByteArrayInputStream(data).use { inputStream ->
        // Works in both 1.0.0 stable and 1.1.0-alpha+
        // (In 1.1.0-alpha03+, you can optionally pass maxVersion = Version.MAX_SUPPORTED or Version.DEVELOPMENT)
        BrushFamily.decode(inputStream)
    }
}
```

#### Decoding (with TextureBitmapStore)

```kotlin
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.ExperimentalInkCustomBrushApi // 1.0.0 stable only
import androidx.ink.storage.BrushFamilyDecodeCallback
import androidx.ink.storage.decode
import java.io.ByteArrayInputStream

@OptIn(ExperimentalInkCustomBrushApi::class) // Required in 1.0.0 stable; omit in 1.1.0-alpha03+
fun decodeBrushFamilyWithTextures(
    data: ByteArray,
    textureStore: AppTextureBitmapStore,
): BrushFamily {
    return ByteArrayInputStream(data).use { inputStream ->
        BrushFamily.decode(
            input = inputStream,
            getClientTextureId = BrushFamilyDecodeCallback { id, bitmap ->
                if (bitmap != null) {
                    textureStore.loadTexture(id, bitmap)
                }
                id
            },
        )
    }
}
```

### AndroidBrushFamilySerialization Style (Java-Friendly)

#### Encoding

```kotlin
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.ExperimentalInkCustomBrushApi // 1.0.0 stable only
import androidx.ink.brush.TextureBitmapStore
import androidx.ink.storage.AndroidBrushFamilySerialization
import java.io.ByteArrayOutputStream

@OptIn(ExperimentalInkCustomBrushApi::class) // Required in 1.0.0 stable; omit in 1.1.0-alpha03+
fun encodeBrushFamily(
    family: BrushFamily,
    textureStore: TextureBitmapStore
): ByteArray {
    return ByteArrayOutputStream().use { stream ->
        AndroidBrushFamilySerialization.encode(
            brushFamily = family,
            output = stream,
            textureBitmapStore = textureStore,
        )
        stream.toByteArray()
    }
}
```

#### Decoding

`AndroidBrushFamilySerialization.decode()` requires a `BrushFamilyDecodeCallback` via the `getClientTextureId` parameter:

```kotlin
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.ExperimentalInkCustomBrushApi // 1.0.0 stable only
import androidx.ink.storage.AndroidBrushFamilySerialization
import androidx.ink.storage.BrushFamilyDecodeCallback
import java.io.ByteArrayInputStream

@OptIn(ExperimentalInkCustomBrushApi::class) // Required in 1.0.0 stable; omit in 1.1.0-alpha03+
fun decodeBrushFamily(
    data: ByteArray,
    textureStore: AppTextureBitmapStore
): BrushFamily {
    return ByteArrayInputStream(data).use { inputStream ->
        AndroidBrushFamilySerialization.decode(
            input = inputStream,
            getClientTextureId = BrushFamilyDecodeCallback { id, bitmap ->
                if (bitmap != null) {
                    textureStore.loadTexture(id, bitmap)
                }
                id
            },
        )
    }
}
```

> **Note on `maxVersion: Version` (`1.1.0-alpha03+`)**: In `1.1.0-alpha03+`, `BrushFamily.decode()` and `AndroidBrushFamilySerialization.decode()` expose an optional `maxVersion: Version = Version.MAX_SUPPORTED` parameter (e.g., `maxVersion = Version.DEVELOPMENT` to allow experimental proto features). In `1.0.0` (Stable), the `Version` class does not exist, so omitting `maxVersion` (or using the `decode(input, getClientTextureId)` overload) compiles cleanly on both `1.0.0` and `1.1.0-alpha03+`.

## Room Database Integration

> **Important (Use `ksp`, Never `annotationProcessor`)**: When using Room in a Kotlin project, you **must** apply the `com.google.devtools.ksp` plugin and use `ksp(libs.androidx.room.compiler)` in `app/build.gradle.kts`. Using `annotationProcessor(libs.androidx.room.compiler)` only processes Java sources and is a no-op for Kotlin `@Database` classes, causing `RuntimeException: Cannot find implementation for AppDatabase. AppDatabase_Impl does not exist` at launch. If your project does not have KSP configured (e.g., on bleeding-edge AGP alphas without a matching KSP version), you can persist the `List<String>` of serialized stroke JSON strings and `.inkbrush` byte files directly in `context.filesDir` using `kotlinx.serialization.json.Json` instead of Room.

### Entity with Stroke Data

Use `List<String>` on the entity so Room automatically applies the `@TypeConverter` when reading and writing:

```kotlin
import androidx.room.ColumnInfo
import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity(tableName = "documents")
data class DocumentEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,
    @ColumnInfo(name = "strokes_data")
    val strokesData: List<String> = emptyList(), // Each entry is a serialized Stroke JSON string
)
```

### Room TypeConverter for `List<String>`

```kotlin
import androidx.room.TypeConverter
import kotlinx.serialization.encodeToString
import kotlinx.serialization.json.Json

class Converters {
    private val json = Json { ignoreUnknownKeys = true }

    @TypeConverter
    fun fromStringList(list: List<String>?): String? {
        return list?.let { json.encodeToString(it) }
    }

    @TypeConverter
    fun toStringList(jsonString: String?): List<String> {
        if (jsonString.isNullOrEmpty()) return emptyList()
        return try {
            json.decodeFromString<List<String>>(jsonString)
        } catch (e: Exception) {
            emptyList()
        }
    }
}
```

Register converters on the database:

```kotlin
import androidx.room.Database
import androidx.room.RoomDatabase
import androidx.room.TypeConverters

@Database(entities = [DocumentEntity::class], version = 1)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {
    abstract fun documentDao(): DocumentDao
}
```

### Repository Pattern: Save & Load

Run stroke encoding and decoding on `Dispatchers.IO` so drawings with hundreds of strokes do not block the main thread:

```kotlin
import androidx.ink.strokes.Stroke
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext

class StrokeRepositoryImpl(
    private val dao: DocumentDao,
) : StrokeRepository {
    override suspend fun saveStrokes(documentId: Long, strokes: List<Stroke>) {
        withContext(Dispatchers.IO) {
            val serializedList = strokes.map { serializeStroke(it) }
            val doc = dao.getById(documentId)
            doc?.let { dao.update(it.copy(strokesData = serializedList)) }
        }
    }

    override suspend fun loadStrokes(documentId: Long): List<Stroke> =
        withContext(Dispatchers.IO) {
            val doc = dao.getById(documentId) ?: return@withContext emptyList()
            doc.strokesData.mapNotNull { deserializeStroke(it) }
        }
}
```

## Loading Strokes on ViewModel Init

Load persisted strokes once on initialization and seed the undo/redo history:

```kotlin
init {
    viewModelScope.launch {
        val initialStrokes = repository.loadStrokes(documentId)
        _uiState.update { it.copy(strokes = initialStrokes) }

        // Initialize undo/redo history
        history.clear()
        history.add(initialStrokes)
        historyIndex = 0
        updateUndoRedoState()
    }
}
```

## Common Pitfalls

- **`ByteArray` in `@Serializable` data classes**: Kotlinx serialization handles `ByteArray` but Kotlin `data class` does not use structural (`contentEquals`) equality for `ByteArray` by default. Override `equals()`/`hashCode()` if you compare `SerializedStroke` instances directly.
- **Not handling decode errors gracefully**: Corrupted or schema-evolved data will throw exceptions on decode. Always use `try/catch` and return `null` or `emptyList()` for graceful degradation.
- **Passing `maxVersion = Version.DEVELOPMENT` when targeting `1.0.0` stable**: The `androidx.ink.brush.Version` class became public in `1.1.0-alpha03+` (it does not exist in `1.0.0` stable). Omit `maxVersion` if you need `1.0.0` compatibility, or pass `maxVersion = Version.DEVELOPMENT` / `Version.MAX_SUPPORTED` on `1.1.0-alpha03+`.
- **Re-deserializing on every Room `Flow` emission**: If you collect a Room `Flow<DocumentEntity>` in `init` and call `saveStrokes()` after every stroke, each save triggers a new `Flow` emission and re-deserializes all strokes. Use a one-shot `loadStrokes(documentId)` on `init` while keeping in-memory `history` as the active session's source of truth.
- **Blocking the main thread with serialization**: Stroke serialization is CPU-intensive when documents contain hundreds of strokes. Always run `saveStrokes()` and `loadStrokes()` on `Dispatchers.IO`.
- **Using `annotationProcessor` instead of `ksp` for Room**: `annotationProcessor(libs.androidx.room.compiler)` does not process Kotlin `@Database` classes and crashes at runtime with `AppDatabase_Impl does not exist`. Use `ksp(libs.androidx.room.compiler)` with `com.google.devtools.ksp`, or persist serialized stroke JSON and `.inkbrush` files directly in `context.filesDir`.
- **Not persisting after undo/redo/erase**: Every mutation to the stroke list must trigger a save. Missing a save point means state is lost on process death.
