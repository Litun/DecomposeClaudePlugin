---
description: Guide for integrating Decompose components with Jetpack/Multiplatform Compose — state, Children, animations, previews
allowed-tools: Read, Grep, Glob
---

You are helping wire Decompose components to Compose UI. Follow these patterns exactly.

## Observing State

Convert `Value<T>` to Compose `State<T>` with `subscribeAsState()`.

```kotlin
import com.arkivanov.decompose.extensions.compose.subscribeAsState

@Composable
fun CounterContent(component: CounterComponent, modifier: Modifier = Modifier) {
    val model by component.model.subscribeAsState()  // auto-subscribes and unsubscribes

    Column(modifier = modifier) {
        Text(text = model.count.toString())
        Button(onClick = component::onIncrementClicked) { Text("Increment") }
    }
}
```

**Rules:**
- Always use `by` delegation (triggers recomposition on change)
- Never subscribe manually in Composable — `subscribeAsState()` handles lifecycle
- Expose a `Model` data class from the component, not raw mutable state

## Rendering Child Stack

Use the `Children` composable from `decompose-extensions-compose`.

```kotlin
import com.arkivanov.decompose.extensions.compose.stack.Children
import com.arkivanov.decompose.extensions.compose.stack.animation.fade
import com.arkivanov.decompose.extensions.compose.stack.animation.scale
import com.arkivanov.decompose.extensions.compose.stack.animation.stackAnimation

@Composable
fun RootContent(component: RootComponent, modifier: Modifier = Modifier) {
    Children(
        stack = component.stack,
        modifier = modifier,
        animation = stackAnimation(fade() + scale()),
    ) {
        when (val child = it.instance) {
            is RootComponent.Child.ListChild -> ListContent(child.component)
            is RootComponent.Child.DetailsChild -> DetailsContent(child.component)
        }
    }
}
```

**Available animations** (can be combined with `+`):
- `fade()` — cross-fade
- `scale()` — shrink/grow
- `slide()` — slide left/right
- `stackAnimation { direction, isInitial, content -> ... }` — fully custom

## Predictive Back Gesture (Android 13+)

```kotlin
import com.arkivanov.decompose.ExperimentalDecomposeApi
import com.arkivanov.decompose.extensions.compose.stack.animation.predictiveBackAnimation

@OptIn(ExperimentalDecomposeApi::class)
@Composable
fun RootContent(component: RootComponent, modifier: Modifier = Modifier) {
    Children(
        stack = component.stack,
        modifier = modifier,
        animation = predictiveBackAnimation(
            backHandler = component.backHandler,
            fallbackAnimation = stackAnimation(fade() + scale()),
            onBack = component::onBackClicked,
        ),
    ) {
        // content
    }
}
```

The component interface must expose `backHandler`:
```kotlin
interface RootComponent : BackHandlerOwner {
    val stack: Value<ChildStack<*, Child>>
    fun onBackClicked(toIndex: Int)
}
```

## Rendering Child Slot (Dialogs/Modals)

```kotlin
@Composable
fun CounterContent(component: CounterComponent) {
    // ... main content ...

    val dialogSlot by component.dialogSlot.subscribeAsState()
    dialogSlot.child?.instance?.also { dialog ->
        AlertDialog(
            onDismissRequest = dialog::onDismissClicked,
            text = { Text(dialog.message) },
            confirmButton = { Button(onClick = dialog::onDismissClicked) { Text("OK") } },
        )
    }
}
```

## Rendering Child Pages (Pager)

```kotlin
import com.arkivanov.decompose.extensions.compose.pages.ChildPages
import com.arkivanov.decompose.extensions.compose.pages.PagesScrollAnimation

@Composable
fun GalleryContent(component: GalleryComponent, modifier: Modifier = Modifier) {
    ChildPages(
        pages = component.pages,
        onPageSelected = component::selectPage,
        modifier = modifier,
        scrollAnimation = PagesScrollAnimation.Default,
    ) { _, page ->
        ImageContent(component = page, modifier = Modifier.fillMaxSize())
    }
}
```

## Rendering Child Panels (Multi-Pane)

```kotlin
import com.arkivanov.decompose.extensions.compose.panels.ChildPanels
import com.arkivanov.decompose.extensions.compose.panels.HorizontalChildPanelsLayout

@Composable
fun MultiPaneContent(component: MultiPaneComponent, modifier: Modifier = Modifier) {
    val panels by component.panels.subscribeAsState()

    BoxWithConstraints(modifier = modifier) {
        val mode = when {
            maxWidth >= 1200.dp -> ChildPanelsMode.TRIPLE
            maxWidth >= 800.dp -> ChildPanelsMode.DUAL
            else -> ChildPanelsMode.SINGLE
        }

        LaunchedEffect(mode) {
            component.setMode(mode)
        }

        ChildPanels(
            panels = panels,
            mainChild = { ArticleListContent(it.instance) },
            detailsChild = { ArticleDetailsContent(it.instance) },
            layout = HorizontalChildPanelsLayout(
                dualWeights = Pair(0.4f, 0.6f),
                tripleWeights = Triple(0.3f, 0.4f, 0.3f),
            ),
        )
    }
}
```

## Tab Navigation with Bottom Bar

```kotlin
@Composable
fun TabsContent(component: TabsComponent, modifier: Modifier = Modifier) {
    Column(modifier = modifier) {
        Children(
            stack = component.stack,
            modifier = Modifier.weight(1f),
        ) {
            when (val child = it.instance) {
                is TabsComponent.Child.HomeChild -> HomeContent(child.component)
                is TabsComponent.Child.ProfileChild -> ProfileContent(child.component)
            }
        }
        BottomNavBar(component)
    }
}

@Composable
private fun BottomNavBar(component: TabsComponent) {
    val stack by component.stack.subscribeAsState()
    val active = stack.active.instance

    NavigationBar {
        NavigationBarItem(
            selected = active is TabsComponent.Child.HomeChild,
            onClick = component::onHomeTabClicked,
            icon = { Icon(Icons.Default.Home, null) },
            label = { Text("Home") },
        )
        NavigationBarItem(
            selected = active is TabsComponent.Child.ProfileChild,
            onClick = component::onProfileTabClicked,
            icon = { Icon(Icons.Default.Person, null) },
            label = { Text("Profile") },
        )
    }
}
```

The component uses `bringToFront` (not `push`) for tab switching:
```kotlin
fun onHomeTabClicked() = nav.bringToFront(Config.Home)
fun onProfileTabClicked() = nav.bringToFront(Config.Profile)
```

## Desktop: LifecycleController

Required on Desktop — syncs component lifecycle with window state.

```kotlin
fun main() {
    val lifecycle = LifecycleRegistry()
    val root = runOnUiThread { DefaultRootComponent(DefaultComponentContext(lifecycle)) }

    application {
        val windowState = rememberWindowState()
        LifecycleController(lifecycle, windowState)   // <-- required on Desktop

        Window(onCloseRequest = ::exitApplication, state = windowState) {
            RootContent(root)
        }
    }
}
```

## Compose Previews

Always extract an interface so you can create a fake for previews.

```kotlin
class PreviewCounterComponent : CounterComponent {
    override val model: Value<CounterComponent.Model> =
        MutableValue(CounterComponent.Model(count = 42))
    override val dialogSlot: Value<ChildSlot<*, DialogComponent>> =
        MutableValue(ChildSlot())
    override fun onIncrementClicked() {}
    override fun onInfoClicked() {}
}

@Preview
@Composable
fun CounterContentPreview() {
    CounterContent(PreviewCounterComponent())
}
```

Minimal `ComponentContext` for previews (if needed for child creation):
```kotlin
val PreviewContext: ComponentContext = DefaultComponentContext(LifecycleRegistry())
```

## Back Handling in Compose

Prefer Decompose's `BackHandler` over Jetpack's to respect the component hierarchy:

```kotlin
@Composable
fun EditorContent(component: EditorComponent) {
    val hasChanges by component.hasUnsavedChanges.subscribeAsState()

    // Jetpack BackHandler — registered at root dispatcher, always fires
    // BackHandler(enabled = hasChanges) { component.onBackClicked() }  // AVOID

    // Better: use component's backHandler, respects component hierarchy
    DisposableEffect(component.backHandler, hasChanges) {
        val callback = BackCallback(isEnabled = hasChanges) { component.onBackClicked() }
        component.backHandler.register(callback)
        onDispose { component.backHandler.unregister(callback) }
    }
}
```

## Critical Warnings

- **Never create the root component inside `@Composable`** — may run on background thread
- **`subscribeAsState()` handles subscription lifecycle** — don't call `subscribe()`/`unsubscribe()` manually in Compose
- **`Children()` manages `SaveableStateHolder`** — don't wrap it in `rememberSaveableStateHolder` yourself
- **Desktop requires `LifecycleController`** — without it components won't receive lifecycle events
- **Use `bringToFront` not `push` for tabs** — push duplicates the config; bringToFront moves it to top
- **Jetpack `BackHandler {}` composable bypasses component hierarchy** — prefer Decompose's `BackHandler`

## Dependencies

```kotlin
// build.gradle.kts
commonMain.dependencies {
    implementation("com.arkivanov.decompose:decompose:$decomposeVersion")
    implementation("com.arkivanov.decompose:extensions-compose:$decomposeVersion")
}
// For shared transitions / experimental features:
// implementation("com.arkivanov.decompose:extensions-compose-experimental:$decomposeVersion")
```
