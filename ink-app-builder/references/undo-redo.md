# Undo / Redo

Reference for implementing undo/redo history for stroke operations.

## History Stack Pattern

The undo/redo system uses a linear history list where each entry is a snapshot of the complete stroke list at that point in time. A `historyIndex` pointer tracks the current position in the history.

### State

```kotlin
import androidx.ink.strokes.Stroke
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update
import kotlinx.coroutines.launch

class DrawingViewModel : ViewModel() {
    // Full history of stroke states
    private val history = mutableListOf<List<Stroke>>()

    // Current position in history (-1 = no history)
    private var historyIndex = -1

    // Observable undo/redo availability
    private val _canUndo = MutableStateFlow(false)
    val canUndo: StateFlow<Boolean> = _canUndo.asStateFlow()

    private val _canRedo = MutableStateFlow(false)
    val canRedo: StateFlow<Boolean> = _canRedo.asStateFlow()
}
```

## Initialization

When loading existing strokes (e.g., from persistent storage), initialize the history:

```kotlin
init {
    viewModelScope.launch {
        val initialStrokes = repository.loadStrokes(documentId)
        _uiState.update { it.copy(strokes = initialStrokes) }

        // Seed history with the initial state
        history.clear()
        history.add(initialStrokes)
        historyIndex = 0
        updateUndoRedoState()
    }
}
```

## Core Operations

### `updateStrokes()` — Record a New State

Called whenever the stroke list changes (adding strokes, erasing strokes, clearing):

```kotlin
private fun updateStrokes(newStrokes: List<Stroke>) {
    // Truncate any "future" history beyond current index
    if (historyIndex < history.size - 1) {
        history.subList(historyIndex + 1, history.size).clear()
    }

    // Append the new state and advance the index
    history.add(newStrokes)
    historyIndex++

    // Update UI state
    _uiState.update { it.copy(strokes = newStrokes) }
    updateUndoRedoState()
}
```

**Key behavior**: When the user makes a new edit after undoing, all "future" entries are discarded. This is the standard linear undo model.

```
History:  [A] [B] [C] [D]
                   ^
              historyIndex = 2 (after undoing D)

User draws E:
History:  [A] [B] [C] [E]   ← D is discarded
                        ^
              historyIndex = 3
```

### `updateUndoRedoState()`

```kotlin
private fun updateUndoRedoState() {
    _canUndo.value = historyIndex > 0
    _canRedo.value = historyIndex < history.size - 1
}
```

### `undo()`

```kotlin
fun undo() {
    if (canUndo.value) {
        historyIndex--
        _uiState.update { it.copy(strokes = history[historyIndex]) }
        updateUndoRedoState()
        viewModelScope.launch { saveStrokes() }
    }
}
```

### `redo()`

```kotlin
fun redo() {
    if (canRedo.value) {
        historyIndex++
        _uiState.update { it.copy(strokes = history[historyIndex]) }
        updateUndoRedoState()
        viewModelScope.launch { saveStrokes() }
    }
}
```

## Integration Points

### With `onStrokesFinished` (Drawing)

```kotlin
fun onStrokesFinished(finishedStrokes: List<Stroke>) {
    val current = history.getOrElse(historyIndex) { emptyList() }
    val newStrokes = current + finishedStrokes
    updateStrokes(newStrokes)
    viewModelScope.launch { saveStrokes() }
}
```

### With Eraser

```kotlin
fun erase(x: Float, y: Float) {
    val strokesBefore = history.getOrElse(historyIndex) { emptyList() }
    val strokesAfter = eraseIntersectingStrokes(x, y, strokesBefore)
    if (strokesAfter.size != strokesBefore.size) {
        updateStrokes(strokesAfter)
    }
}
```

### With Clear All

```kotlin
fun clearStrokes() {
    if (_uiState.value.strokes.isNotEmpty()) {
        updateStrokes(emptyList())
        viewModelScope.launch { saveStrokes() }
    }
}
```

## Saving After Undo/Redo

Both `undo()` and `redo()` trigger `saveStrokes()` to persist the current state. The save function reads from the history at the current index:

```kotlin
suspend fun saveStrokes() {
    if (historyIndex >= 0 && historyIndex < history.size) {
        val strokesToSave = history[historyIndex]
        repository.saveStrokes(documentId, strokesToSave)
    } else if (history.isEmpty()) {
        repository.saveStrokes(documentId, emptyList())
    }
}
```

## Exposing to Compose UI

```kotlin
import androidx.compose.material3.Icon
import androidx.compose.material3.IconButton
import androidx.compose.runtime.getValue
import androidx.compose.ui.res.painterResource
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow

// In ViewModel
val canUndo: StateFlow<Boolean> = _canUndo.asStateFlow()
val canRedo: StateFlow<Boolean> = _canRedo.asStateFlow()

// In Composable
val canUndo by viewModel.canUndo.collectAsStateWithLifecycle()
val canRedo by viewModel.canRedo.collectAsStateWithLifecycle()

IconButton(
    onClick = { viewModel.undo() },
    enabled = canUndo
) {
    Icon(painter = painterResource(R.drawable.undo), contentDescription = "Undo")
}

IconButton(
    onClick = { viewModel.redo() },
    enabled = canRedo
) {
    Icon(painter = painterResource(R.drawable.redo), contentDescription = "Redo")
}
```

## Memory Considerations

Each history entry holds a `List<Stroke>`, which references `Stroke` objects. Because strokes are immutable, entries share `Stroke` instances — only the list allocation is duplicated, not the stroke data.

For apps with very long editing sessions, consider capping history length:

```kotlin
private val maxHistorySize = 50

private fun updateStrokes(newStrokes: List<Stroke>) {
    if (historyIndex < history.size - 1) {
        history.subList(historyIndex + 1, history.size).clear()
    }
    history.add(newStrokes)
    historyIndex++

    // Trim oldest entries if history exceeds max
    while (history.size > maxHistorySize) {
        history.removeAt(0)
        historyIndex--
    }

    _uiState.update { it.copy(strokes = newStrokes) }
    updateUndoRedoState()
}
```

## Common Pitfalls

- **Not truncating future on new edit**: If you append without clearing `history.subList(historyIndex + 1, ...)`, redo will navigate to stale states that don't account for the new edit.
- **Forgetting to seed history on init**: If history is empty when the user first draws, `historyIndex` math breaks. Always add the initial state (even if it's `emptyList()`).
- **Not saving after undo/redo**: The persistent storage should reflect the current view state. Forgetting `saveStrokes()` after undo/redo means closing and reopening the app loses the undo/redo result.
- **Reading strokes from `_uiState` instead of `history[historyIndex]`**: Always use the history as the source of truth. The UI state is a projection of the history.
- **Unbounded history growth**: In long sessions, history can consume significant memory. Consider capping with `maxHistorySize`.
