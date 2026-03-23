---
description: Guide for building Decompose components correctly — structure, lifecycle, state, instance retaining, and back handling
allowed-tools: Read, Grep, Glob
---

You are helping write a Decompose component. Follow these patterns exactly.

## Component Structure

Always use `interface + DefaultXxxComponent`. Never extend a library base class.

```kotlin
interface CounterComponent {
    val model: Value<Model>
    fun onIncrementClicked()

    data class Model(val count: Int = 0)
}

class DefaultCounterComponent(
    componentContext: ComponentContext,
    private val onFinished: () -> Unit,
) : CounterComponent, ComponentContext by componentContext {

    private val _model = MutableValue(CounterComponent.Model())
    override val model: Value<CounterComponent.Model> = _model

    override fun onIncrementClicked() {
        _model.update { it.copy(count = it.count + 1) }
    }
}
```

**Rules:**
- `ComponentContext by componentContext` — always delegate, never extend
- Constructor receives `componentContext: ComponentContext` as first parameter
- Parent callbacks (navigation, results) come via constructor lambdas
- Expose `Value<T>` (immutable), hold `MutableValue<T>` privately
- `MutableValue.update { }` for state mutations — call only on the main thread

## Value vs Other State Holders

`Value<T>` is Decompose's multiplatform observable. Prefer it for cross-platform components.

```kotlin
// Good — works on all platforms, observable in Compose/SwiftUI/React
val state: Value<State> = _state

// Also acceptable if you're coroutines-only (Android/JVM):
val state: StateFlow<State>
```

`Value` is NOT a coroutine — no `collect`, use `subscribe`/`subscribeAsState()` in Compose.

## Lifecycle

Components get lifecycle automatically. Subscribe only in `init` or lifecycle callbacks.

```kotlin
class DefaultSomeComponent(componentContext: ComponentContext) : ComponentContext by componentContext {
    init {
        // Correct: lifecycle subscription in init
        lifecycle.doOnStart { /* start polling */ }
        lifecycle.doOnStop { /* stop polling */ }
        lifecycle.doOnDestroy { /* cleanup */ }
    }
}
```

Lifecycle states: `INITIALIZED → CREATED → STARTED → RESUMED → STOPPED → DESTROYED`
- Active component is `RESUMED`
- Back-stack components are `CREATED` (still alive, stopped)

## State Preservation (survives process death + config changes)

Use `stateKeeper` for UI state that must survive process death.

```kotlin
@Serializable
private data class State(val query: String = "", val selectedId: Long? = null)

class DefaultSearchComponent(componentContext: ComponentContext) : ComponentContext by componentContext {

    private var state: State by saveable(serializer = State.serializer(), init = ::State)
}
```

Manual version (when you need more control):
```kotlin
private var state = stateKeeper.consume("STATE", State.serializer()) ?: State()

init {
    stateKeeper.register("STATE", State.serializer()) { state }
}
```

**Rules:**
- State class must be `@Serializable`
- Keep serialized state small — Android bundle has ~500KB limit
- `stateKeeper.consume()` removes the saved value (call only once)

## Instance Retaining (survives config changes, like ViewModel)

Use `retainedInstance` to keep objects alive across Android configuration changes.

```kotlin
class DefaultTimerComponent(componentContext: ComponentContext) : ComponentContext by componentContext {

    private val timer = retainedInstance { Timer() }

    private class Timer : InstanceKeeper.Instance {
        var ticks = 0
            private set

        init { /* start timer */ }

        override fun onDestroy() {
            // Called when component is permanently destroyed (not just rotated)
        }
    }
}
```

**Rules:**
- The retained class must NOT be `inner` — no references to the outer component
- Do NOT pass `Activity`, `Context`, or `View` into a retained instance — memory leak
- Implement `InstanceKeeper.Instance` and clean up in `onDestroy()`

## Combining state preservation + instance retaining

```kotlin
class DefaultComponent(componentContext: ComponentContext) : ComponentContext by componentContext {

    private val logic by saveable(serializer = Logic.State.serializer(), state = { it.state }) { savedState ->
        retainedInstance { Logic(savedState) }
    }

    private class Logic(savedState: Logic.State?) : InstanceKeeper.Instance {
        var state = savedState ?: Logic.State()
            private set

        @Serializable data class State(val items: List<String> = emptyList())
        override fun onDestroy() {}
    }
}
```

## Back Button Handling

Navigation models handle back automatically when `handleBackButton = true`. For custom handling:

```kotlin
class DefaultEditorComponent(componentContext: ComponentContext) : ComponentContext by componentContext {

    private val backCallback = BackCallback(isEnabled = false) {
        // called when back is pressed and isEnabled = true
        showDiscardDialog()
    }

    init {
        backHandler.register(backCallback)
    }

    fun onFormChanged() {
        backCallback.isEnabled = true  // enable when there are unsaved changes
    }
}
```

Priority: last-registered wins. Use `priority = Int.MAX_VALUE` to always intercept first.

## Preview / Test Doubles

Always extract an interface — this enables Compose Previews and integration tests.

```kotlin
// Fake for previews
class PreviewCounterComponent : CounterComponent {
    override val model: Value<Model> = MutableValue(Model(count = 42))
    override fun onIncrementClicked() {}
}

// Minimal ComponentContext for previews
internal val PreviewContext: ComponentContext = DefaultComponentContext(LifecycleRegistry())
```

## Critical Warnings

- **Root component must be created on the UI/Main thread, NEVER inside a `@Composable` function** — compositions can run on background threads
- **`defaultComponentContext()` must be called only once per Activity/Fragment lifetime** — calling twice crashes
- **`retainedComponent()` has the same restriction** — call only once in `onCreate`
- **Navigate only on Main thread** — background navigation causes crashes
- **`MutableValue` is thread-safe but subscribe/update only on main thread** to avoid races
- **Don't make retained inner classes** — captures outer component → memory leak

## Quick Reference

| Need | API |
|------|-----|
| Component context delegation | `class Foo(ctx: ComponentContext) : ComponentContext by ctx` |
| Observable state | `MutableValue<T>` / `Value<T>` |
| Update state | `_state.update { it.copy(...) }` |
| Survive config change | `retainedInstance { }` |
| Survive process death | `saveable(serializer = ...)` or `stateKeeper` |
| Lifecycle events | `lifecycle.doOnStart/Stop/Destroy { }` |
| Custom back handling | `backHandler.register(BackCallback { })` |
| Auto back in navigation | `handleBackButton = true` in `childStack()`/`childSlot()` |
