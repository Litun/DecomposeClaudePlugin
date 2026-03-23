---
name: decompose-platform
description: This skill should be used when the user is setting up Decompose on a specific platform for the first time or debugging a platform-specific issue, mentions "defaultComponentContext", "DefaultComponentContext", "LifecycleRegistry", iOS Decompose setup, Swift Decompose integration, "StateValue", "StackView", "ApplicationLifecycle", "RootHolder", Desktop Compose Decompose setup, "LifecycleController", web/JS Decompose lifecycle, "attachToDocument", root component initialization, or asks how to set up the root component in an Activity, Fragment, AppDelegate, main function, or iOS app.
version: 1.0.0
---

You are helping with platform-specific Decompose integration. Follow the correct pattern for the target platform.

## Android — Activity (recommended)

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val root = DefaultRootComponent(
            componentContext = defaultComponentContext(),  // lifecycle + state + back, one call only
        )

        setContent {
            MaterialTheme { RootContent(component = root, modifier = Modifier.fillMaxSize()) }
        }
    }
}
```

**Warnings:**
- `defaultComponentContext()` — call **only once** in `onCreate`, calling twice crashes
- `retainedComponent()` — same restriction
- Do NOT pass `Activity`/`Context`/`View` into `retainedInstance`

**Fragment variant:** `defaultComponentContext(onBackPressedDispatcher = requireActivity().onBackPressedDispatcher)`

**Retain root across config changes:** `val root = retainedComponent { DefaultRootComponent(it) }`

**Gradle:**
```kotlin
implementation("com.arkivanov.decompose:decompose:$decomposeVersion")
implementation("com.arkivanov.decompose:extensions-compose:$decomposeVersion")
```

---

## iOS with SwiftUI

**Required — copy these Swift files from `sample/app-ios/app-ios/DecomposeHelpers/` into your project (cannot be a library):**
- `StateValue.swift` — `@StateValue` property wrapper for observing `Value<T>`
- `StackView.swift` — native SwiftUI navigation for `ChildStack`
- `MutableValue.swift` — stub `Value` for previews
- `SimpleChildStack.swift` — stub `ChildStack` for previews

### Option A: ApplicationLifecycle (app-scoped root)

```swift
class AppDelegate: NSObject, UIApplicationDelegate {
    let root: RootComponent = DefaultRootComponent(
        componentContext: DefaultComponentContext(lifecycle: ApplicationLifecycle())
    )
}

@main struct iOSApp: App {
    @UIApplicationDelegateAdaptor(AppDelegate.self) var appDelegate
    var body: some Scene {
        WindowGroup { RootView(root: appDelegate.root) }
    }
}
```

### Option B: Manual lifecycle (recommended for multi-window or complex structure)

```swift
class RootHolder: ObservableObject {
    let lifecycle: LifecycleRegistry
    let root: RootComponent
    init() {
        lifecycle = LifecycleRegistryKt.LifecycleRegistry()
        root = DefaultRootComponent(componentContext: DefaultComponentContext(lifecycle: lifecycle))
        LifecycleRegistryExtKt.create(lifecycle)
    }
    deinit { LifecycleRegistryExtKt.destroy(lifecycle) }
}

@main struct iOSApp: App {
    @UIApplicationDelegateAdaptor(AppDelegate.self) var appDelegate
    @Environment(\.scenePhase) var scenePhase
    var rootHolder: RootHolder { appDelegate.rootHolder }
    var body: some Scene {
        WindowGroup {
            RootView(rootHolder.root)
                .onChange(of: scenePhase) { phase in
                    switch phase {
                    case .background: LifecycleRegistryExtKt.stop(rootHolder.lifecycle)
                    case .inactive:   LifecycleRegistryExtKt.pause(rootHolder.lifecycle)
                    case .active:     LifecycleRegistryExtKt.resume(rootHolder.lifecycle)
                    @unknown default: break
                    }
                }
        }
    }
}
```

### Observing Value in SwiftUI

```swift
struct CounterView: View {
    let component: CounterComponent
    @StateValue private var model: CounterComponentModel
    init(_ component: CounterComponent) {
        self.component = component
        _model = StateValue(component.model)
    }
    var body: some View {
        VStack {
            Text("\(model.count)")
            Button("Increment") { component.onIncrementClicked() }
        }
    }
}
```

### ChildStack in SwiftUI

```swift
StackView(
    stackValue: StateValue(root.stack),
    getTitle: { child in
        switch child {
        case is RootComponentChild.ListChild: return "List"
        case is RootComponentChild.DetailsChild: return "Details"
        default: return ""
        }
    },
    onBack: root.onBackClicked,
    childContent: { child in
        switch child {
        case let c as RootComponentChild.ListChild: ListView(c.component)
        case let c as RootComponentChild.DetailsChild: DetailsView(c.component)
        default: EmptyView()
        }
    }
)
```

---

## iOS with Compose Multiplatform

```kotlin
// shared/src/iosMain/kotlin/RootViewController.kt
fun rootViewController(root: RootComponent): UIViewController =
    ComposeUIViewController { RootContent(component = root) }
```

```swift
struct RootView: UIViewControllerRepresentable {
    let root: RootComponent
    func makeUIViewController(context: Context) -> UIViewController {
        RootViewControllerKt.rootViewController(root: root)
    }
    func updateUIViewController(_ uiViewController: UIViewController, context: Context) {}
}
```

Use the same `AppDelegate`/lifecycle setup as the SwiftUI approach above.

---

## Desktop (JVM/Compose)

```kotlin
fun main() {
    val lifecycle = LifecycleRegistry()
    val stateKeeper = StateKeeperDispatcher(
        File("saved_state.dat").takeIf { it.exists() }?.readSerializableContainer()
    )

    // Must be on UI thread BEFORE application { }
    val root = runOnUiThread {
        DefaultRootComponent(componentContext = DefaultComponentContext(lifecycle = lifecycle, stateKeeper = stateKeeper))
    }

    application {
        val windowState = rememberWindowState()
        LifecycleController(lifecycle, windowState)  // required — syncs lifecycle with window

        Window(onCloseRequest = ::exitApplication, state = windowState, title = "My App") {
            RootContent(root)
        }
    }
}

fun <T> runOnUiThread(block: () -> T): T {
    if (SwingUtilities.isEventDispatchThread()) return block()
    var result: T? = null
    SwingUtilities.invokeAndWait { result = block() }
    @Suppress("UNCHECKED_CAST")
    return result as T
}
```

**ProGuard (release builds):** `-keep class com.arkivanov.decompose.extensions.compose.mainthread.SwingMainThreadChecker`

**Warnings:** `LifecycleController` required; always `runOnUiThread` before `application { }`.

---

## Web (Kotlin/JS or Wasm)

```kotlin
@OptIn(ExperimentalDecomposeApi::class)
fun main() {
    val lifecycle = LifecycleRegistry()
    val root = DefaultRootComponent(componentContext = DefaultComponentContext(lifecycle = lifecycle))
    lifecycle.attachToDocument()
    createRoot(document.getElementById("app")!!).render(RootContent.create { component = root })
}

private fun LifecycleRegistry.attachToDocument() {
    fun update() { if (document.visibilityState == DocumentVisibilityState.visible) resume() else stop() }
    update()
    document.addEventListener(EventType("visibilitychange"), { update() })
}
```

**With browser history:** pass `webHistoryController = DefaultWebHistoryController()` and `deepLinkUrl = window.location.href`.

---

## Platform Comparison

| Platform | ComponentContext creation | Lifecycle source |
|----------|--------------------------|------------------|
| Android | `defaultComponentContext()` | Activity lifecycle (auto) |
| Android retained | `retainedComponent { }` | Activity lifecycle (auto) |
| iOS | `DefaultComponentContext(ApplicationLifecycle())` or manual `RootHolder` | `scenePhase` |
| Desktop | `DefaultComponentContext(LifecycleRegistry())` + `runOnUiThread` | `LifecycleController` |
| Web/JS | `DefaultComponentContext(LifecycleRegistry())` | `visibilitychange` event |

## Critical Warnings

- **`StateValue`/`StackView` Swift files cannot be a library** — must copy source into every iOS project
- **iOS `ApplicationLifecycle()` is experimental** — use `RootHolder` for multi-window apps
- **Desktop requires `LifecycleController`** — without it components silently never start/resume
- **Always `runOnUiThread` on Desktop** before `application { }`
- **Web: attach to `visibilitychange`** — otherwise lifecycle stays RESUMED when tab is hidden
