---
description: Guide for choosing and implementing the right Decompose navigation model (Stack, Slot, Pages, Panels, Items)
allowed-tools: Read, Grep, Glob
---

You are helping implement navigation with Decompose. Choose the right model and follow these patterns exactly.

## Choosing a Navigation Model

| Scenario | Model |
|----------|-------|
| Screens pushed onto a stack (list → details, login → home) | **ChildStack** |
| One optional child at a time (dialog, modal, bottom sheet) | **ChildSlot** |
| Horizontal pager / tab-with-swipe (image gallery, onboarding) | **ChildPages** |
| Responsive master-detail layout (list + details side by side) | **ChildPanels** |
| Lazy list where each item is a live component | **ChildItems** |
| None of the above (carousel, custom state machine) | **Generic Navigation** |

Multiple navigation models in one component are fine — just use different `key` values.

---

## Configuration Requirements

Every navigation model uses **configurations** — serializable descriptions of what to show.

```kotlin
@Serializable  // kotlinx-serialization plugin required
private sealed interface Config {
    @Serializable data object List : Config
    @Serializable data class Details(val itemId: Long) : Config
    @Serializable data class Profile(val userId: String) : Config
}
```

**Rules:**
- Must be `@Serializable` (or pass `serializer = null` to disable state saving)
- Must be immutable (`data class`/`data object` with `val` properties only)
- Must implement `equals`/`hashCode` correctly — `data class` does this automatically
- Keep small — Android bundle limit applies (~500 KB total)
- **ChildStack**: configurations must be unique within the stack

---

## ChildStack — Screen Stack Navigation

Use for sequential screens: push forward, pop back.

```kotlin
interface RootComponent {
    val stack: Value<ChildStack<*, Child>>
    fun onBackClicked(toIndex: Int)  // iOS multi-pop support

    sealed class Child {
        class ListChild(val component: ListComponent) : Child()
        class DetailsChild(val component: DetailsComponent) : Child()
    }
}

class DefaultRootComponent(
    componentContext: ComponentContext,
) : RootComponent, ComponentContext by componentContext {

    private val nav = StackNavigation<Config>()

    override val stack: Value<ChildStack<*, RootComponent.Child>> =
        childStack(
            source = nav,
            serializer = Config.serializer(),
            initialConfiguration = Config.List,
            handleBackButton = true,   // auto-pop on back press
            childFactory = ::createChild,
        )

    private fun createChild(config: Config, ctx: ComponentContext): RootComponent.Child =
        when (config) {
            is Config.List -> ListChild(DefaultListComponent(ctx, onItemSelected = { nav.pushNew(Config.Details(it)) }))
            is Config.Details -> DetailsChild(DefaultDetailsComponent(ctx, config.itemId, onFinished = nav::pop))
        }

    override fun onBackClicked(toIndex: Int) = nav.popTo(index = toIndex)

    @Serializable private sealed interface Config {
        @Serializable data object List : Config
        @Serializable data class Details(val itemId: Long) : Config
    }
}
```

**Key StackNavigation operations:**
- `nav.push(config)` — push new screen (fails if config already in stack)
- `nav.pushNew(config)` — push only if not already active
- `nav.pushToFront(config)` — push and remove any existing instance
- `nav.pop()` — go back one screen
- `nav.popTo(index = n)` — pop to specific index (iOS swipe-back)
- `nav.popWhile { it !is Config.Home }` — pop until condition
- `nav.replaceCurrent(config)` — replace active screen
- `nav.replaceAll(config)` — clear stack with single screen
- `nav.bringToFront(config)` — move config to top without duplicating (use for tabs)

**Delivering result back to previous screen:**
```kotlin
onDeleted = { itemId ->
    nav.pop {  // onComplete callback, runs after navigation
        (stack.active.instance as? ListChild)?.component?.onItemDeleted(itemId)
    }
}
```

---

## ChildSlot — Single Optional Child (Dialogs/Modals)

Use when you need to show one optional child at a time.

```kotlin
interface CounterComponent {
    val dialogSlot: Value<ChildSlot<*, DialogComponent>>
    fun onInfoClicked()
}

class DefaultCounterComponent(componentContext: ComponentContext) : CounterComponent, ComponentContext by componentContext {

    private val dialogNav = SlotNavigation<DialogConfig>()

    override val dialogSlot: Value<ChildSlot<*, DialogComponent>> =
        childSlot(
            source = dialogNav,
            serializer = null,             // dialogs usually don't need persistence
            handleBackButton = true,       // dismiss on back press
            childFactory = { config, _ ->
                DefaultDialogComponent(
                    message = "Value: ${config.value}",
                    onDismissed = dialogNav::dismiss,
                )
            },
        )

    override fun onInfoClicked() {
        dialogNav.activate(DialogConfig(value = 42))
    }

    @Serializable private data class DialogConfig(val value: Int)
}
```

**Key SlotNavigation operations:**
- `dialogNav.activate(config)` — show the child
- `dialogNav.dismiss()` — hide the child

**In Compose:**
```kotlin
val slot by component.dialogSlot.subscribeAsState()
slot.child?.instance?.also { dialog ->
    DialogContent(dialog)
}
```

---

## ChildPages — Pager Navigation

Use for horizontal/vertical pagers where all pages can be alive simultaneously.

```kotlin
interface GalleryComponent {
    val pages: Value<ChildPages<*, ImageComponent>>
    fun selectPage(index: Int)
}

class DefaultGalleryComponent(componentContext: ComponentContext) : GalleryComponent, ComponentContext by componentContext {

    private val nav = PagesNavigation<ImageId>()

    override val pages: Value<ChildPages<*, ImageComponent>> =
        childPages(
            source = nav,
            serializer = ImageId.serializer(),
            initialPages = { Pages(items = ImageId.entries, selectedIndex = 0) },
            childFactory = { id, ctx -> DefaultImageComponent(ctx, id) },
        )

    override fun selectPage(index: Int) = nav.select(index = index)
}
```

**Key PagesNavigation operations:**
- `nav.select(index)` — jump to page
- `nav.selectNext()` / `nav.selectPrev()` — navigate one step
- `nav.selectFirst()` / `nav.selectLast()`
- `nav.setItems(items, selectedIndex)` — replace all items

---

## ChildPanels — Multi-Pane Layout

Use for responsive master-detail UIs (phone: single pane, tablet: side by side).

```kotlin
class DefaultMultiPaneComponent(componentContext: ComponentContext) : MultiPaneComponent, ComponentContext by componentContext {

    private val nav = PanelsNavigation<Unit, DetailsConfig, ExtraConfig>()

    val panels: Value<ChildPanels<Unit, MainChild, DetailsConfig, DetailsChild, ExtraConfig, ExtraChild>> =
        childPanels(
            source = nav,
            initialPanels = { Panels(main = Unit) },
            serializers = Triple(null, DetailsConfig.serializer(), ExtraConfig.serializer()),
            handleBackButton = true,
            mainFactory = { _, ctx -> MainChild(DefaultListComponent(ctx, onItemSelected = { id -> nav.navigate { it.copy(details = DetailsConfig(id)) } })) },
            detailsFactory = { config, ctx -> DetailsChild(DefaultDetailsComponent(ctx, config.itemId, onFinished = { nav.navigate { it.copy(details = null) } })) },
            extraFactory = { config, ctx -> ExtraChild(DefaultExtraComponent(ctx, config)) },
        )

    fun setMode(mode: ChildPanelsMode) {
        nav.navigate { it.copy(mode = mode) }
    }
}
```

**Modes:** `ChildPanelsMode.SINGLE` (phone), `DUAL` (tablet), `TRIPLE` (large tablet/desktop)

In Compose, use `BoxWithConstraints` to determine mode and `LaunchedEffect` to push it to the component:
```kotlin
BoxWithConstraints {
    val mode = when {
        maxWidth >= 1200.dp -> ChildPanelsMode.TRIPLE
        maxWidth >= 800.dp -> ChildPanelsMode.DUAL
        else -> ChildPanelsMode.SINGLE
    }
    LaunchedEffect(mode) {
        component.setMode(mode)
    }
    ChildPanels(panels = panels, ...)
}
```

---

## Multiple Navigation Models in One Component

When you have two navigation models of the same type, supply distinct keys:

```kotlin
private val mainStack = childStack(source = mainNav, key = "MainStack", ...)
private val sideStack = childStack(source = sideNav, key = "SideStack", ...)
```

---

## Deep Linking

Override `initialConfiguration` or `initialStack` using a URL parameter:

```kotlin
class DefaultRootComponent(
    componentContext: ComponentContext,
    deepLinkUrl: String? = null,
) : RootComponent, ComponentContext by componentContext {

    override val stack = childStack(
        source = nav,
        serializer = Config.serializer(),
        initialStack = { buildInitialStack(deepLinkUrl) },  // returns List<Config>
        childFactory = ::createChild,
    )

    private fun buildInitialStack(url: String?): List<Config> =
        listOf(Config.Home) + when {
            url?.contains("/item/") == true -> listOf(Config.Details(url.substringAfterLast("/").toLong()))
            else -> emptyList()
        }
}
```

---

## Critical Warnings

- **Always navigate on the Main thread** — background navigation causes undefined behavior
- **ChildStack: configurations must be unique** — duplicate configs throw by default (enable `DecomposeSettings.duplicateConfigurationsEnabled` to allow duplicates)
- **Use `bringToFront` for tab switching**, NOT `push` — `push` duplicates, `bringToFront` moves to top
- **Use `pushNew` instead of `push`** to guard against double-taps
- **Android bundle size limit** — keep configurations lean, no large data blobs
- **Pass `serializer = null`** if you intentionally don't want state saved (e.g., transient dialogs)
- **Multiple stacks need unique `key`** — default key is `"default"`, collision crashes
