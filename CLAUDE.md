# Decompose — Claude Code Guide

This project uses **[Decompose](https://github.com/arkivanov/Decompose)** for multiplatform navigation and component lifecycle management.

## Key Rules

- **Component structure:** Always use `interface XxxComponent` + `class DefaultXxxComponent(...) : XxxComponent, ComponentContext by componentContext`. Never extend library base classes.
- **Delegation:** `ComponentContext by componentContext` — delegation, not inheritance.
- **Root component:** Must be created on the **UI/Main thread**, **outside** any `@Composable` function (before `setContent`/`application { }`).
- **`defaultComponentContext()`** — call only **once** per Activity/Fragment lifetime, in `onCreate`.
- **Configurations** (nav state): Must be `@Serializable data class/object`, immutable, small (~<500 KB total on Android).
- **Navigate on Main thread only** — background navigation causes crashes.
- **State observation in Compose:** Use `subscribeAsState()` to convert `Value<T>` → Compose `State<T>`.
- **Tab switching:** Use `bringToFront()`, not `push()`.
- **Dialogs/modals:** Use `ChildSlot` with `SlotNavigation`.
- **Instance retaining (like ViewModel):** Use `retainedInstance { }` — never pass `Activity`/`Context` into retained objects.
- **State preservation (process death):** Use `saveable(serializer = ...)` or `stateKeeper`.

## Navigation Model Cheatsheet

| Use case | Model |
|----------|-------|
| Screen stack (push/pop) | `ChildStack` + `StackNavigation` |
| Dialog / modal / bottom sheet | `ChildSlot` + `SlotNavigation` |
| Horizontal pager / swipeable tabs | `ChildPages` + `PagesNavigation` |
| Master-detail responsive layout | `ChildPanels` + `PanelsNavigation` |
| Lazy list of live components | `ChildItems` |

## Available Skills

Use these slash commands for detailed guidance and code patterns:

- `/decompose:decompose-component` — component structure, lifecycle, state/instance keeping, back handling
- `/decompose:decompose-navigation` — choosing and implementing navigation models with full examples
- `/decompose:decompose-compose` — Compose UI integration (subscribeAsState, Children, animations, previews)
- `/decompose:decompose-platform` — platform setup (Android, iOS SwiftUI/Compose, Desktop, Web)
