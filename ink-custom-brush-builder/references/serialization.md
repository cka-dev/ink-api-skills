# Serialization

## Overview

Custom brush families can be serialized to and deserialized from byte streams. This enables saving brushes to files, storing them in databases, loading from bundled resources, and sharing between devices.

Two serialization APIs are available in `androidx.ink.storage`:
- **`BrushFamily.encode()` / `BrushFamily.decode()`** — Idiomatic Kotlin extension functions (`import androidx.ink.storage.encode` and `import androidx.ink.storage.decode`). On Android, these include overloads that accept a `TextureBitmapStore` or `BrushFamilyDecodeCallback`.
- **`AndroidBrushFamilySerialization.encode()` / `.decode()`** — `@JvmStatic` utility object exposing the Android `TextureBitmapStore` / `BrushFamilyDecodeCallback` overloads without competing with the multiplatform `ByteArray`/`OnDecodeTexturePngBytes` overloads.

> **Note on `@OptIn(ExperimentalInkCustomBrushApi::class)`**: In **`1.1.0-alpha02+`** (`1.1.0-alpha03+`), `BrushFamily.encode()`/`decode()`, `AndroidBrushFamilySerialization`, and `BrushFamilyDecodeCallback` are unannotated and require **no `@OptIn`** (in `1.1.0-alpha05+`, `ExperimentalInkCustomBrushApi` itself is internal `@RestrictTo(LIBRARY_GROUP)`). If decoding/encoding pre-serialized brushes on **`1.0.0` (Stable)**, add `@OptIn(ExperimentalInkCustomBrushApi::class)` (`import androidx.ink.brush.ExperimentalInkCustomBrushApi`) and omit `maxVersion`.

## Encoding (Serialization)

### Basic Encoding (No Textures)

```kotlin
import androidx.ink.brush.BrushFamily
import androidx.ink.storage.encode
import java.io.ByteArrayOutputStream

fun encodeBrushFamily(brushFamily: BrushFamily): ByteArray {
    return ByteArrayOutputStream().use { stream ->
        brushFamily.encode(stream)
        stream.toByteArray()
    }
    // Or in 1.1.0-alpha, use the 0-arg ByteArray convenience overload directly:
    // return brushFamily.encode()
}
```

### Encoding with Textures

When encoding with a `TextureBitmapStore`, write to an `OutputStream` (`brushFamily.encode(stream, textureBitmapStore)`):

```kotlin
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.TextureBitmapStore
import androidx.ink.storage.encode
import java.io.ByteArrayOutputStream

fun encodeBrushFamilyWithTextures(
    brushFamily: BrushFamily,
    textureBitmapStore: TextureBitmapStore,
): ByteArray {
    return ByteArrayOutputStream().use { stream ->
        brushFamily.encode(stream, textureBitmapStore)
        stream.toByteArray()
    }
}
```

### Android-Specific Encoding (`AndroidBrushFamilySerialization`)

```kotlin
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.TextureBitmapStore
import androidx.ink.storage.AndroidBrushFamilySerialization
import java.io.ByteArrayOutputStream

fun androidEncode(
    brushFamily: BrushFamily,
    textureBitmapStore: TextureBitmapStore,
): ByteArray {
    return ByteArrayOutputStream().use { stream ->
        AndroidBrushFamilySerialization.encode(
            brushFamily = brushFamily,
            output = stream,
            textureBitmapStore = textureBitmapStore,
        )
        stream.toByteArray()
    }
}
```

### Saving to File

Pass your `TextureBitmapStore` when saving a `BrushFamily` to a file so any referenced texture bitmaps are embedded in the serialized output:

```kotlin
import android.content.Context
import android.net.Uri
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.TextureBitmapStore
import androidx.ink.storage.encode

fun saveBrushToFile(
    context: Context,
    uri: Uri,
    brushFamily: BrushFamily,
    textureBitmapStore: TextureBitmapStore = TextureBitmapStore { null },
) {
    context.contentResolver.openOutputStream(uri)?.use { outputStream ->
        brushFamily.encode(outputStream, textureBitmapStore)
    }
}
```

## Decoding (Deserialization)

### Basic Decoding

```kotlin
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.Version
import androidx.ink.storage.decode
import java.io.InputStream

fun decodeBrushFamily(inputStream: InputStream): BrushFamily {
    return BrushFamily.decode(
        inputStream,
        maxVersion = Version.DEVELOPMENT,
    )
}

// Or in 1.1.0-alpha03+, decode directly from a ByteArray without ByteArrayInputStream:
fun decodeBrushFamilyDirect(bytes: ByteArray): BrushFamily =
    BrushFamily.decode(input = bytes, maxVersion = Version.DEVELOPMENT)
```

### Decoding with Texture Callback

When the brush contains embedded textures, provide a `BrushFamilyDecodeCallback` to register decoded bitmaps in your `TextureBitmapStore`.

> **Important (Overload Disambiguation in `1.1.0-alpha`)**: `BrushFamily.Companion.decode` has two 3-parameter overloads in `androidx.ink.storage` — one taking `getClientTextureId: BrushFamilyDecodeCallback` (`(String, Bitmap?) -> String`) and one taking `onDecodeTexture: OnDecodeTexturePngBytes?` (`(String, ByteArray?) -> String`). Do **not** pass an untyped trailing lambda `{ id, bitmap -> ... }` to `BrushFamily.decode(inputStream, maxVersion) { ... }`, as Kotlin cannot disambiguate between `Bitmap?` and `ByteArray?`. Always wrap the callback in `BrushFamilyDecodeCallback { id, bitmap -> ... }` (or use `AndroidBrushFamilySerialization.decode(...)`):

```kotlin
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.Version
import androidx.ink.storage.AndroidBrushFamilySerialization
import androidx.ink.storage.BrushFamilyDecodeCallback
import androidx.ink.storage.decode
import java.io.InputStream

fun decodeBrushFamilyWithTextures(
    inputStream: InputStream,
    textureStore: AppTextureBitmapStore,
): BrushFamily {
    return BrushFamily.decode(
        input = inputStream,
        maxVersion = Version.DEVELOPMENT,
        getClientTextureId = BrushFamilyDecodeCallback { id, bitmap ->
            bitmap?.let { textureStore.loadTexture(id, it) }
            id
        },
    )
}

// Equivalent call via AndroidBrushFamilySerialization
fun decodeBrushFamilyWithTexturesAndroid(
    inputStream: InputStream,
    textureStore: AppTextureBitmapStore,
): BrushFamily {
    return AndroidBrushFamilySerialization.decode(
        input = inputStream,
        maxVersion = Version.DEVELOPMENT,
        getClientTextureId = BrushFamilyDecodeCallback { id, bitmap ->
            bitmap?.let { textureStore.loadTexture(id, it) }
            id
        },
    )
}
```

### From Android Raw Resources

For brushes bundled as raw resources in the APK:

```kotlin
import android.content.Context
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.Version
import androidx.ink.storage.BrushFamilyDecodeCallback
import androidx.ink.storage.decode

fun loadFromRawResource(
    context: Context,
    resourceId: Int,
    textureStore: AppTextureBitmapStore,
): BrushFamily {
    return context.resources.openRawResource(resourceId).use { inputStream ->
        BrushFamily.decode(
            input = inputStream,
            maxVersion = Version.DEVELOPMENT,
            getClientTextureId = BrushFamilyDecodeCallback { id, bitmap ->
                bitmap?.let { textureStore.loadTexture(id, it) }
                id
            },
        )
    }
}
```

## Saving to Room Database

Store brush data as a `ByteArray` column in a Room entity:

```kotlin
import androidx.room.ColumnInfo
import androidx.room.Entity
import androidx.room.PrimaryKey

// Entity
@Entity(tableName = "custom_brushes")
data class CustomBrushEntity(
    @PrimaryKey val name: String,
    @ColumnInfo(typeAffinity = ColumnInfo.BLOB)
    val brushBytes: ByteArray  // Encoded brush family bytes
)
```

### Saving

```kotlin
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.TextureBitmapStore
import androidx.ink.storage.encode
import java.io.ByteArrayOutputStream

fun serializeForDatabase(
    brushFamily: BrushFamily,
    textureBitmapStore: TextureBitmapStore = TextureBitmapStore { null },
): ByteArray {
    return ByteArrayOutputStream().use { stream ->
        brushFamily.encode(stream, textureBitmapStore)
        stream.toByteArray()
    }
}

// Usage (pass textureStore to embed any custom brush textures in the BLOB)
val entity = CustomBrushEntity(
    name = "My Custom Brush",
    brushBytes = serializeForDatabase(brushFamily, textureStore),
)
customBrushDao.saveCustomBrush(entity)
```

### Loading

```kotlin
import androidx.ink.brush.BrushFamily
import androidx.ink.brush.Version
import androidx.ink.storage.BrushFamilyDecodeCallback
import androidx.ink.storage.decode
import java.io.ByteArrayInputStream

fun deserializeFromDatabase(
    brushBytes: ByteArray,
    textureStore: AppTextureBitmapStore? = null,
): BrushFamily {
    return ByteArrayInputStream(brushBytes).use { inputStream ->
        BrushFamily.decode(
            input = inputStream,
            maxVersion = Version.DEVELOPMENT,
            getClientTextureId = BrushFamilyDecodeCallback { id, bitmap ->
                if (bitmap != null && textureStore != null) {
                    textureStore.loadTexture(id, bitmap)
                }
                id
            },
        )
    }
}
```

## TextureBitmapStore

`TextureBitmapStore` is a functional interface that maps texture IDs to bitmaps. Used during encoding to embed texture data:

```kotlin
import android.graphics.Bitmap
import androidx.ink.brush.TextureBitmapStore

// Empty store (returns null for all texture IDs)
val emptyStore = TextureBitmapStore { null }

// Store backed by a map
val bitmapMap = mutableMapOf<String, Bitmap>()
val mapStore = TextureBitmapStore { id -> bitmapMap[id] }
```

## Version Parameter (`1.1.0-alpha03+`)

In `1.1.0-alpha03+`, `BrushFamily.decode()` and `AndroidBrushFamilySerialization.decode()` expose an optional `maxVersion: Version = Version.MAX_SUPPORTED` parameter:
- `Version.DEVELOPMENT` is used during development to allow decoding all features, including experimental ones.
- `Version.MAX_SUPPORTED` (the default) allows all stable/supported features for that library version.
- **Note for `1.0.0` (Stable)**: The `androidx.ink.brush.Version` class does not exist in `1.0.0` stable (and was `@RestrictTo(LIBRARY_GROUP)` prior to `1.1.0-alpha03`); when decoding pre-serialized brushes on `1.0.0` stable, omit `maxVersion` (e.g., `BrushFamily.decode(inputStream)` or `AndroidBrushFamilySerialization.decode(inputStream, getClientTextureId)`), which compiles cleanly on both `1.0.0` and `1.1.0-alpha03+`.

## Common Pitfalls

- **Forgetting the texture callback** — Without a `BrushFamilyDecodeCallback`, embedded textures may be silently dropped during decoding. The brush will render without textures.
- **Not using `Dispatchers.IO`** — Serialization involves file I/O and compression. Always run on a background dispatcher.
- **Mixing up `encode`/`decode` APIs** — `BrushFamily.encode/decode` and `AndroidBrushFamilySerialization.encode/decode` produce compatible formats, but use the same pair consistently.
- **Not handling decode exceptions** — Invalid or corrupted data will throw exceptions. Always wrap decode calls in try/catch.
- **Version-specific `@OptIn(ExperimentalInkCustomBrushApi::class)`** — Required when calling `BrushFamily.encode()`/`decode()` or `AndroidBrushFamilySerialization` on `1.0.0` stable, and not needed in `1.1.0-alpha02+`.
