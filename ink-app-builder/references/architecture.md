# Architecture

Reference for the recommended MVVM architecture when building an Ink-based drawing app with Jetpack Compose.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Compose UI Layer                      │
│                                                         │
│  DrawingScreen                                          │
│    ├── DrawingSurface (Canvas + InProgressStrokes)       │
│    ├── DrawingToolbox (brush picker, undo/redo, eraser) │
│    └── TopAppBar (title, export, navigation)            │
│                                                         │
│  Observes: uiState, currentBrush, isEraserMode,        │
│            canUndo, canRedo                             │
└─────────────┬───────────────────────────────────────────┘
              │ collectAsStateWithLifecycle()
              │ Method calls (onStrokesFinished, undo, redo, etc.)
┌─────────────▼───────────────────────────────────────────┐
│                    ViewModel Layer                        │
│                                                         │
│  DrawingViewModel                                       │
│    ├── _uiState: MutableStateFlow<DrawingUiState>       │
│    ├── _selectedBrush: MutableStateFlow<Brush>          │
│    ├── _isEraserMode: MutableStateFlow<Boolean>         │
│    ├── _canUndo / _canRedo: MutableStateFlow<Boolean>   │
│    ├── history: MutableList<List<Stroke>>               │
│    ├── historyIndex: Int                                │
│    └── textureStore: TextureBitmapStore                 │
│                                                         │
│  Operations: onStrokesFinished(), undo(), redo(),       │
│              erase(), changeBrush(), saveStrokes()      │
└─────────────┬───────────────────────────────────────────┘
              │ suspend function calls
              │ Flow collection
┌─────────────▼───────────────────────────────────────────┐
│                   Repository Layer                       │
│                                                         │
│  StrokeRepository                                       │
│    ├── saveStrokes(documentId, List<Stroke>)            │
│    ├── loadStrokes(documentId): List<Stroke>            │
│    └── getDocumentStream(documentId): Flow<Document>    │
│                                                         │
│  Converters (serialization/deserialization)              │
└─────────────┬───────────────────────────────────────────┘
              │ Room DAO calls
┌─────────────▼───────────────────────────────────────────┐
│                   Data Layer (Room)                       │
│                                                         │
│  DocumentEntity (strokesData: List<String>)             │
│  DocumentDao (CRUD operations)                          │
│  AppDatabase (@TypeConverters)                          │
└─────────────────────────────────────────────────────────┘
```

## UI State

Define a single state class for the drawing screen:

```kotlin
import androidx.ink.strokes.Stroke

data class DrawingUiState(
    val strokes: List<Stroke> = emptyList(),
    val documentTitle: String = "",
    // Add other UI-relevant fields as needed
)
```

## ViewModel

The ViewModel is the central coordinator. It holds all mutable state and exposes read-only `StateFlow`s to the UI.

### Key Responsibilities

| Responsibility | StateFlow | Methods |
|---|---|---|
| Stroke list | `uiState.strokes` | `onStrokesFinished()`, `clearStrokes()` |
| Current brush | `currentBrush` | `getCurrentBrush()`, `changeBrush()`, `changeBrushColor()`, `changeBrushSize()` |
| Eraser mode | `isEraserMode` | `setEraserMode()`, `erase()`, `startErase()`, `endErase()` |
| Undo/Redo | `canUndo`, `canRedo` | `undo()`, `redo()` |
| Persistence | — | `saveStrokes()` (internal) |

### Skeleton

```kotlin
import androidx.ink.brush.Brush
import androidx.ink.geometry.MutableVec
import androidx.ink.strokes.Stroke
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import dagger.hilt.android.lifecycle.HiltViewModel
import javax.inject.Inject
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch

@HiltViewModel
class DrawingViewModel @Inject constructor(
    private val repository: StrokeRepository,
    val textureStore: AppTextureBitmapStore,
) : ViewModel() {

    private val _uiState = MutableStateFlow(DrawingUiState())
    val uiState: StateFlow<DrawingUiState> = _uiState.asStateFlow()

    private val _selectedBrush = MutableStateFlow(createDefaultBrush())
    val currentBrush: StateFlow<Brush> = _selectedBrush.asStateFlow()

    fun getCurrentBrush(): Brush = _selectedBrush.value

    private val _isEraserMode = MutableStateFlow(false)
    val isEraserMode: StateFlow<Boolean> = _isEraserMode.asStateFlow()

    private val _canUndo = MutableStateFlow(false)
    val canUndo: StateFlow<Boolean> = _canUndo.asStateFlow()

    private val _canRedo = MutableStateFlow(false)
    val canRedo: StateFlow<Boolean> = _canRedo.asStateFlow()

    // Undo/redo history
    private val history = mutableListOf<List<Stroke>>()
    private var historyIndex = -1

    // Eraser state
    private var previousPoint: MutableVec? = null
    private val eraserPadding = 50f

    init {
        loadDocument()
    }
}
```

## Composable ↔ ViewModel Connection

### `collectAsStateWithLifecycle` Pattern

```kotlin
import androidx.compose.runtime.Composable
import androidx.compose.runtime.CompositionLocalProvider
import androidx.compose.runtime.getValue
import androidx.compose.runtime.remember
import androidx.hilt.lifecycle.viewmodel.compose.hiltViewModel // Or androidx.hilt.navigation.compose.hiltViewModel on Hilt < 1.3.0
import androidx.ink.rendering.android.canvas.CanvasStrokeRenderer
import androidx.lifecycle.compose.collectAsStateWithLifecycle

@Composable
fun DrawingScreen(
    viewModel: DrawingViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    val currentBrush by viewModel.currentBrush.collectAsStateWithLifecycle()
    val isEraserMode by viewModel.isEraserMode.collectAsStateWithLifecycle()
    val canUndo by viewModel.canUndo.collectAsStateWithLifecycle()
    val canRedo by viewModel.canRedo.collectAsStateWithLifecycle()

    val textureStore = viewModel.textureStore
    val cacheGen by textureStore.generation.collectAsStateWithLifecycle()
    val renderer = remember(textureStore, cacheGen) {
        CanvasStrokeRenderer.create(textureStore)
    }

    CompositionLocalProvider(LocalTextureStore provides textureStore) {
        DrawingSurface(
            strokes = uiState.strokes,
            canvasStrokeRenderer = renderer,
            currentBrush = currentBrush,
            onGetNextBrush = { viewModel.getCurrentBrush() },
            onStrokesFinished = { viewModel.onStrokesFinished(it) },
            isEraserMode = isEraserMode,
            onErase = { x, y -> viewModel.erase(x, y) },
            onEraseStart = { viewModel.startErase() },
            onEraseEnd = { viewModel.endErase() },
        )
    }
}
```

### Why `collectAsStateWithLifecycle` + Synchronous Handoff in `DrawingSurface`?

- `collectAsStateWithLifecycle()` automatically stops collection when the lifecycle is below `STARTED` (e.g., app is backgrounded), preventing unnecessary recompositions after `onStop`. Prefer it over plain `collectAsState()` for `StateFlow`s in the ViewModel.
- **Important (Wet-to-Dry Handoff)**: Because `collectAsStateWithLifecycle()` collects `StateFlow` emissions asynchronously via a coroutine (`produceState`), updating `_uiState` in `viewModel.onStrokesFinished()` does not update `uiState.strokes` until the next UI run loop — whereas `InProgressStrokes` removes the completed wet stroke immediately after `onStrokesFinished` returns. Ensure `DrawingSurface` includes the synchronous `pendingStrokes` Compose state buffer (see [drawing-surface.md](drawing-surface.md)) so the dry `Canvas` is invalidated in the exact same UI thread run loop without a 1-frame flicker.

## Hilt Dependency Injection

### ViewModel Injection

```kotlin
import androidx.lifecycle.SavedStateHandle
import androidx.lifecycle.ViewModel
import dagger.hilt.android.lifecycle.HiltViewModel
import javax.inject.Inject

@HiltViewModel
class DrawingViewModel @Inject constructor(
    private val repository: StrokeRepository,
    val textureStore: AppTextureBitmapStore,
    savedStateHandle: SavedStateHandle,  // For navigation arguments
) : ViewModel()
```

### Module for Singletons

```kotlin
import android.content.Context
import androidx.room.Room
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.android.qualifiers.ApplicationContext
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object AppModule {

    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(context, AppDatabase::class.java, "app.db").build()
    }

    @Provides
    fun provideDocumentDao(database: AppDatabase): DocumentDao {
        return database.documentDao()
    }

    @Provides
    @Singleton
    fun provideRepository(dao: DocumentDao): StrokeRepository {
        return StrokeRepositoryImpl(dao)
    }
}
```

### TextureBitmapStore as Singleton

The `TextureBitmapStore` should be a `@Singleton` so that loaded textures persist across screen rotations and navigation:

```kotlin
import android.content.Context
import androidx.ink.brush.TextureBitmapStore
import dagger.hilt.android.qualifiers.ApplicationContext
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class AppTextureBitmapStore @Inject constructor(
    @ApplicationContext context: Context
) : TextureBitmapStore {
    // ...
}
```

## Repository Pattern

The repository encapsulates all data operations and provides a clean API to the ViewModel:

```kotlin
import androidx.ink.strokes.Stroke
import kotlinx.coroutines.flow.Flow

interface StrokeRepository {
    fun getDocumentStream(documentId: Long): Flow<Document?>
    suspend fun saveStrokes(documentId: Long, strokes: List<Stroke>)
    suspend fun loadStrokes(documentId: Long): List<Stroke>
}
```

The ViewModel should never directly access DAOs or serialization utilities.

## Persisting on Every Mutation vs. `onCleared()`

Persist strokes immediately (on `Dispatchers.IO`) after each user action that mutates the stroke list (`onStrokesFinished()`, `endErase()`, `undo()`, `redo()`, `clearStrokes()`).

> **Warning**: Do **not** rely on `viewModelScope.launch { saveStrokes() }` inside `ViewModel.onCleared()`. `viewModelScope` is cancelled when the ViewModel is cleared, so any suspend function launched on `viewModelScope` inside `onCleared()` will be cancelled immediately. If you need background work to survive ViewModel destruction, inject an application-scoped `CoroutineScope` into the repository.

## Common Pitfalls

- **Exposing `MutableStateFlow` to UI**: Always expose read-only `StateFlow` via `asStateFlow()`. The UI should never mutate ViewModel state directly.
- **Creating `CanvasStrokeRenderer` in ViewModel**: The renderer should be created in the Composable layer because it depends on the texture store's generation state and should be re-created on texture changes.
- **Forgetting `@HiltViewModel`**: Without this annotation, Hilt cannot inject the ViewModel and `hiltViewModel()` will crash.
- **Not using `SavedStateHandle` for navigation args**: When navigating to a drawing screen with a document ID, extract it from `SavedStateHandle` rather than passing it via constructor parameters.
- **Tight coupling between layers**: The ViewModel should depend on the repository interface, not the concrete implementation. This enables testing with fake repositories.
