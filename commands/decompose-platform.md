---
description: Platform-specific Decompose setup for Android, iOS (SwiftUI + Compose), Desktop (JVM), and Web (JS/Wasm)
allowed-tools: Read, Grep, Glob
---

You are helping with platform-specific Decompose integration. Follow the correct pattern for the target platform.

## Android

### Setup in Activity (recommended)

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val root = DefaultRootComponent(
            componentContext = defaultComponentContext(),
        )

        setContent {
            MaterialTheme {
                RootContent(component = root, modifier = Modifier.fillMaxSize())
            }
        }
    }
}
```

### Setup in Fragment

```kotlin
class RootFragment : Fragment() {
    private lateinit var root: RootComponent

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        root = DefaultRootComponent(
            componentContext = defaultComponentContext(
                onBackPressedDispatcher = requireActivity().onBackPressedDispatcher
            )
        )
    }

    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View =
        ComposeView(requireContext()).apply {
            setContent { RootContent(root) }
        }
}
```

### Retaining the root component across config changes (optional)

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val root = retainedComponent { DefaultRootComponent(it) }
        setContent { RootContent(root) }
    }
}
```

**Android warnings:**
- `defaultComponentContext()` — call **only once** in `onCreate`, never again
- `retainedComponent()` — same restriction, call only once in `onCreate`
- Do NOT pass `Activity`/`Context`/`View` into `retainedInstance` — memory leak

### Gradle dependencies (Android)

```kotlin
// build.gradle.kts
implementation("com.arkivanov.decompose:decompose:$decomposeVersion")
implementation("com.arkivanov.decompose:extensions-compose:$decomposeVersion")
// Optional: AndroidX integration (use AndroidX ViewModel/Lifecycle with Decompose)
// implementation("com.arkivanov.decompose:jetpack-component-context:$decomposeVersion")
```

---

## iOS with SwiftUI

**Required:** Copy these files from `sample/app-ios/app-ios/DecomposeHelpers/` into your Xcode project — they cannot be distributed as a library:
- `StateValue.swift` — property wrapper to observe `Value<T>` in SwiftUI
- `StackView.swift` — native SwiftUI navigation for `ChildStack`
- `MutableValue.swift` — for stubbing `Value` in SwiftUI Previews
- `SimpleChildStack.swift` — for stubbing `ChildStack` in SwiftUI Previews

### Option A: ApplicationLifecycle (simpler — use when root lives in app scope)

```swift
// AppDelegate.swift
class AppDelegate: NSObject, UIApplicationDelegate {
    let root: RootComponent = DefaultRootComponent(
        componentContext: DefaultComponentContext(lifecycle: ApplicationLifecycle())
    )
}

// App entrypoint
@main
struct iOSApp: App {
    @UIApplicationDelegateAdaptor(AppDelegate.self) var appDelegate

    var body: some Scene {
        WindowGroup {
            RootView(root: appDelegate.root)
        }
    }
}
```

### Option B: Manual lifecycle (recommended for multi-window or complex app structure)

```swift
// RootHolder.swift
class RootHolder: ObservableObject {
    let lifecycle: LifecycleRegistry
    let root: RootComponent

    init() {
        lifecycle = LifecycleRegistryKt.LifecycleRegistry()
        root = DefaultRootComponent(
            componentContext: DefaultComponentContext(lifecycle: lifecycle)
        )
        LifecycleRegistryExtKt.create(lifecycle)
    }

    deinit {
        LifecycleRegistryExtKt.destroy(lifecycle)
    }
}

// App entrypoint
@main
struct iOSApp: App {
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
struct RootView: View {
    let root: RootComponent

    var body: some View {
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
    }
}
```

---

## iOS with Compose Multiplatform

```kotlin
// shared/src/iosMain/kotlin/RootViewController.kt
import androidx.compose.ui.window.ComposeUIViewController
import platform.UIKit.UIViewController

fun rootViewController(root: RootComponent): UIViewController =
    ComposeUIViewController {
        RootContent(component = root)
    }
```

```swift
// RootView.swift
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
    val backDispatcher = DefaultBackPressedDispatcher()

    val stateKeeper = StateKeeperDispatcher(
        File("saved_state.dat").takeIf { it.exists() }?.readSerializableContainer()
    )

    // Must be on UI thread BEFORE application { }
    val root = runOnUiThread {
        DefaultRootComponent(
            componentContext = DefaultComponentContext(
                lifecycle = lifecycle,
                stateKeeper = stateKeeper,
                backHandler = backDispatcher,
            )
        )
    }

    application {
        val windowState = rememberWindowState()
        LifecycleController(lifecycle, windowState)

        Window(
            onCloseRequest = ::exitApplication,
            state = windowState,
            title = "My App",
            onKeyEvent = { event ->
                if (event.key == Key.Escape && event.type == KeyEventType.KeyUp) {
                    backDispatcher.back()
                } else false
            },
        ) {
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

**ProGuard rule** (required for release builds):
```
-keep class com.arkivanov.decompose.extensions.compose.mainthread.SwingMainThreadChecker
```

**Desktop warnings:**
- `LifecycleController` is **required** — without it components never receive START/RESUME
- Always `runOnUiThread` for root component creation
- Add Escape key → back dispatcher for keyboard navigation

---

## Web (Kotlin/JS or Wasm)

```kotlin
@OptIn(ExperimentalDecomposeApi::class)
fun main() {
    val lifecycle = LifecycleRegistry()

    val root = DefaultRootComponent(
        componentContext = DefaultComponentContext(lifecycle = lifecycle),
    )

    // Attach lifecycle to document visibility
    lifecycle.attachToDocument()

    // Render (React example)
    createRoot(document.getElementById("app")!!).render(
        RootContent.create { component = root }
    )
}

private fun LifecycleRegistry.attachToDocument() {
    fun update() {
        if (document.visibilityState == DocumentVisibilityState.visible) resume() else stop()
    }
    update()
    document.addEventListener(EventType("visibilitychange"), { update() })
}
```

### Web with browser history (deep links + back button)

```kotlin
val root = DefaultRootComponent(
    componentContext = DefaultComponentContext(lifecycle = lifecycle),
    webHistoryController = DefaultWebHistoryController(),
    deepLinkUrl = window.location.href,
)
```

The component must implement `WebNavigationOwner` and expose `WebNavigation<Config>`.

---

## Platform Comparison Summary

| Platform | ComponentContext creation | Lifecycle source |
|----------|--------------------------|------------------|
| Android | `defaultComponentContext()` in Activity | Activity lifecycle (auto) |
| Android retained | `retainedComponent { }` in Activity | Activity lifecycle (auto) |
| iOS (Compose/SwiftUI) | `DefaultComponentContext(ApplicationLifecycle())` or manual | `scenePhase` → manual calls |
| Desktop | `DefaultComponentContext(LifecycleRegistry())` + `runOnUiThread` | `LifecycleController(lifecycle, windowState)` |
| Web/JS | `DefaultComponentContext(LifecycleRegistry())` | `visibilitychange` event |

## Critical Warnings

- **`StateValue` and `StackView` Swift files cannot be published as a library** — must copy source into every iOS project
- **iOS `ApplicationLifecycle()` is experimental** — use manual `RootHolder` if you have multiple `UIViewController`s or complex lifecycle
- **Desktop requires `LifecycleController`** — skip it and components silently never start/resume
- **Always `runOnUiThread` on Desktop before `application { }`** — Compose runs on a different thread
- **Web: attach lifecycle to `visibilitychange`** — otherwise components stay RESUMED even when tab is hidden
