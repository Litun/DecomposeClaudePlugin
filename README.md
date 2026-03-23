# Decompose Claude Code Plugin

A Claude Code plugin that provides production best practices for [Decompose](https://github.com/arkivanov/Decompose) — the Kotlin Multiplatform library for navigation and component lifecycle management.

## What You Get

- **4 slash commands** with detailed code patterns for components, navigation, Compose UI, and platform setup
- **4 auto-invoked skills** that activate automatically when Claude detects you're working with Decompose
- **A `CLAUDE.md` template** to paste into any Decompose project for always-on guidance

## Installation

### Step 1 — Add this repo as a marketplace

In a Claude Code session, run:

```
/plugin marketplace add Litun/DecomposeClaudePlugin
```

### Step 2 — Install the plugin

```
/plugin install decompose@Litun-DecomposeClaudePlugin
```

Or browse via `/plugin` → **Discover** and install from there.

### Step 3 — Add CLAUDE.md to your project (optional but recommended)

Copy the contents of `CLAUDE.md` into your project's `CLAUDE.md`. This gives Claude baseline Decompose context in every conversation — no skill invocation needed.

## Usage

### Slash commands (explicit)

Invoke any command when you need deep guidance on a specific topic:

```
/decompose:decompose-component
/decompose:decompose-navigation
/decompose:decompose-compose
/decompose:decompose-platform
```

### Auto-invoked skills (automatic)

Claude detects Decompose context and loads the right skill automatically. For example:

- You write `class DefaultXxxComponent(componentContext: ComponentContext)` → `decompose-component` activates
- You ask "should I use ChildStack or ChildSlot for this dialog?" → `decompose-navigation` activates
- You ask "how do I observe state in Compose?" → `decompose-compose` activates
- You ask "how do I set this up on iOS?" → `decompose-platform` activates

## Plugin Structure

```
DecomposeClaudePlugin/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest
├── commands/                    # Slash commands (user-invoked)
│   ├── decompose-component.md
│   ├── decompose-navigation.md
│   ├── decompose-compose.md
│   └── decompose-platform.md
├── skills/                      # Auto-invoked skills
│   ├── decompose-component/SKILL.md
│   ├── decompose-navigation/SKILL.md
│   ├── decompose-compose/SKILL.md
│   └── decompose-platform/SKILL.md
├── CLAUDE.md                    # Template for your project's CLAUDE.md
└── README.md
```

## Skill Coverage

### `decompose-component`
- `interface + DefaultXxxComponent` pattern
- `ComponentContext by componentContext` delegation
- `Value<T>` / `MutableValue<T>` observable state
- `retainedInstance { }` — ViewModel equivalent
- `saveable(serializer = ...)` — process death survival
- Lifecycle events (`doOnStart`, `doOnStop`, `doOnDestroy`)
- Custom back button handling
- Preview / test double patterns

### `decompose-navigation`
- Decision table: when to use Stack vs Slot vs Pages vs Panels vs Items
- Configuration requirements (`@Serializable`, immutable, size limits)
- Full `ChildStack` example with `push`, `pop`, `bringToFront`, `popTo`, result delivery
- `ChildSlot` for dialogs/modals
- `ChildPages` for pagers
- `ChildPanels` for responsive master-detail
- Multiple navigation models with distinct keys
- Deep linking via `initialStack`

### `decompose-compose`
- `subscribeAsState()` — `Value<T>` → Compose `State<T>`
- `Children()` composable with `stackAnimation(fade() + scale())`
- `predictiveBackAnimation()` for Android 13+
- `ChildPages()` composable
- `ChildPanels()` with `HorizontalChildPanelsLayout`
- Tab bottom bar pattern
- `LifecycleController` for Desktop
- Compose previews with fake component implementations
- Back handling with `BackCallback` (vs Jetpack `BackHandler`)

### `decompose-platform`
- **Android:** `defaultComponentContext()`, Fragment setup, `retainedComponent()`
- **iOS SwiftUI:** `ApplicationLifecycle` vs manual `RootHolder`, `StateValue`, `StackView`
- **iOS Compose MP:** `ComposeUIViewController`, `UIViewControllerRepresentable`
- **Desktop JVM:** `runOnUiThread`, `LifecycleRegistry`, `LifecycleController`, state persistence to file
- **Web/JS:** `LifecycleRegistry` + `visibilitychange`, browser history

## Links

- [Decompose documentation](https://arkivanov.github.io/Decompose/)
- [Decompose GitHub](https://github.com/arkivanov/Decompose)
- [Decompose template repository](https://github.com/arkivanov/decompose-multiplatform-template)
