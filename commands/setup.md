---
description: Quick-start or verify Decompose setup in the current project. Scans build files and platform entry points, fetches latest docs/samples if needed, then fixes every issue found directly in the project files.
allowed-tools: Read, Grep, Glob, WebFetch, Edit, Write, Bash
---

You are setting up or verifying Decompose in the current project. Work through the steps below in order.

---

## Step 1 — Detect project structure & platforms

**Scan build files:**
- Glob for `build.gradle.kts`, `build.gradle`, `libs.versions.toml` across the project
- Grep for `com.arkivanov.decompose` in all found build files
- Record the current version if found, or mark as **missing**

**Detect targeted platforms** (run in parallel):
- **Android:** Glob `**/AndroidManifest.xml`
- **iOS:** Glob `**/*.xcodeproj` and `**/*.swift`
- **Desktop:** Grep `application {` in files matching `**/main.kt`
- **Web:** Glob `**/jsMain` and `**/wasmJsMain` source directories

**For each detected platform check what is already set up:**
- Android: Grep for `defaultComponentContext` in Activity/Fragment Kotlin files
- iOS SwiftUI: Glob for `StateValue.swift`, `StackView.swift`, `MutableValue.swift`, `SimpleChildStack.swift`; grep for `ApplicationLifecycle` or `LifecycleRegistry` in Swift files
- iOS Compose MP: Grep `iosMain` source set for `ComposeUIViewController`
- Desktop: Grep `main.kt` for `LifecycleController`, `LifecycleRegistry`, `runOnUiThread`
- Web: Grep `jsMain`/`wasmJsMain` entry file for `attachToDocument`, `LifecycleRegistry`
- KMP iOS export: Grep `build.gradle.kts` for `export("com.arkivanov.decompose")`

Summarise findings before moving to Step 2.

---

## Step 2 — Fetch latest docs (only what is needed)

Fetch selectively based on Step 1 findings:

- **Always fetch** — latest stable version:
  `https://github.com/arkivanov/Decompose/releases/latest`

- **If Decompose dependency is missing or version unknown:**
  `https://arkivanov.github.io/Decompose/getting-started/installation/`

- **If iOS platform detected AND any helper Swift file is missing:**
  Fetch the directory listing to find raw file URLs:
  `https://github.com/arkivanov/Decompose/tree/master/sample/app-ios/app-ios/DecomposeHelpers`
  Then fetch each missing file's raw content from:
  `https://raw.githubusercontent.com/arkivanov/Decompose/master/sample/app-ios/app-ios/DecomposeHelpers/<filename>`

- **If Desktop platform detected AND setup is incomplete:**
  `https://github.com/arkivanov/Decompose/tree/master/sample/app-desktop/src/jvmMain/kotlin`

- **If Web platform detected AND setup is incomplete:**
  `https://github.com/arkivanov/Decompose/tree/master/sample/app-web`

---

## Step 3 — Fix every issue found

Apply all fixes directly to the project files. Use the patterns below.

### Dependencies

**If Decompose is missing from `libs.versions.toml`:**
Add version and library aliases:
```toml
[versions]
decompose = "<latest-version>"

[libraries]
decompose = { module = "com.arkivanov.decompose:decompose", version.ref = "decompose" }
decompose-extensions-compose = { module = "com.arkivanov.decompose:extensions-compose", version.ref = "decompose" }
```

**If Decompose is missing from `build.gradle.kts` (no version catalog):**
```kotlin
implementation("com.arkivanov.decompose:decompose:<latest-version>")
implementation("com.arkivanov.decompose:extensions-compose:<latest-version>")
```

Add to `commonMain` for KMP projects; to the `android` block for Android-only projects.

### Android

If `defaultComponentContext()` is missing in the main Activity, add it in `onCreate` before `setContent`:
```kotlin
val root = DefaultRootComponent(
    componentContext = defaultComponentContext(),
)
```

To retain across config changes use `retainedComponent { DefaultRootComponent(it) }` instead.

**Warnings to communicate:**
- Call `defaultComponentContext()` / `retainedComponent()` **only once** in `onCreate`
- Do NOT pass `Activity`/`Context`/`View` into `retainedInstance`

### iOS SwiftUI — helper files

If any of `StateValue.swift`, `StackView.swift`, `MutableValue.swift`, `SimpleChildStack.swift` are missing, fetch and write them from the Decompose sample repo (Step 2 URLs). These files **cannot be a library** — they must be copied directly into the Xcode project.

If `ApplicationLifecycle`/`LifecycleRegistry` lifecycle wiring is missing, create `RootHolder.swift`:
```swift
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

    deinit { LifecycleRegistryExtKt.destroy(lifecycle) }
}
```

And wire `scenePhase` in the App entry point:
```swift
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

### iOS Compose Multiplatform

If `ComposeUIViewController` bridge is missing in `iosMain`, create `shared/src/iosMain/kotlin/RootViewController.kt`:
```kotlin
fun rootViewController(root: RootComponent): UIViewController =
    ComposeUIViewController { RootContent(component = root) }
```

### iOS KMP framework export

If the `framework { export(...) }` block is missing in the shared module's `build.gradle.kts`:
```kotlin
framework {
    export("com.arkivanov.decompose:decompose:<version>")
    export("com.arkivanov.essenty:lifecycle:<essenty-version>")
    // Optional — for state preservation across process death:
    // export("com.arkivanov.essenty:state-keeper:<essenty-version>")
}
```

### Desktop

If `LifecycleRegistry`, `LifecycleController`, or `runOnUiThread` are missing in `main.kt`, add the full setup:
```kotlin
fun main() {
    val lifecycle = LifecycleRegistry()
    val stateKeeper = StateKeeperDispatcher(
        File("saved_state.dat").takeIf { it.exists() }?.readSerializableContainer()
    )

    val root = runOnUiThread {
        DefaultRootComponent(
            componentContext = DefaultComponentContext(lifecycle = lifecycle, stateKeeper = stateKeeper)
        )
    }

    application {
        val windowState = rememberWindowState()
        LifecycleController(lifecycle, windowState)  // required

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

If a ProGuard/R8 rules file exists but is missing this rule, add it:
```
-keep class com.arkivanov.decompose.extensions.compose.mainthread.SwingMainThreadChecker
```

### Web (Kotlin/JS or Wasm)

If `LifecycleRegistry` + `attachToDocument` are missing in the main entry file:
```kotlin
@OptIn(ExperimentalDecomposeApi::class)
fun main() {
    val lifecycle = LifecycleRegistry()

    val root = DefaultRootComponent(
        componentContext = DefaultComponentContext(lifecycle = lifecycle)
    )

    lifecycle.attachToDocument()

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

---

## Step 4 — Report

Output a concise summary:
- ✅ Items that were already correctly set up
- 🔧 Items that were fixed (list file paths changed)
- ⚠️ Items requiring manual action (e.g. adding files to Xcode project, enabling framework exports in Xcode build settings)
