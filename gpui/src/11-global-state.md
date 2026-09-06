# Chapter 11 — Global state

Most state should live in an entity you pass around (Chapter 3) — it's
testable and explicit. But some things are genuinely application-wide
singletons: the active theme, app settings, a registry, feature flags. GPUI
models these as **globals**.

## Declaring a global

Implement the `Global` marker trait on a type:

```rust
use gpui::Global;

#[derive(Default)]
struct Settings {
    show_line_numbers: bool,
    font_size: f32,
}

impl Global for Settings {}
```

## Setting and reading

```rust
// Install (or replace) the global:
cx.set_global(Settings { show_line_numbers: true, font_size: 14.0 });

// Read it (panics if not set):
let size = cx.global::<Settings>().font_size;

// Mutate it in place:
cx.update_global::<Settings, _>(|settings, _cx| {
    settings.show_line_numbers = !settings.show_line_numbers;
});

// Get-or-insert-default — never panics, creates it if missing:
let settings = cx.default_global::<Settings>();   // &mut Settings
```

`cx.global::<T>()` **panics** if the global isn't set, so either set it during
startup or use `cx.default_global::<T>()` (requires `T: Default`) which inserts
a default on first access. These methods are on `App`, so they're available
from any `&mut App` or (via `Deref`) any `&mut Context<V>`.

## A real use: a registry of open dialog windows

Globals shine for cross-cutting bookkeeping. Here's the actual pattern this app
uses to ensure only one instance of each dialog window opens — pressing the
shortcut twice re-focuses the existing window instead of stacking duplicates:

```rust
use std::collections::HashMap;
use gpui::{AnyWindowHandle, App, Global, SharedString};

#[derive(Default)]
struct OpenDialogs(HashMap<SharedString, AnyWindowHandle>);
impl Global for OpenDialogs {}

fn open_unique(title: SharedString, cx: &mut App /* …open args… */) {
    // Is one already open under this title?
    let existing = cx.default_global::<OpenDialogs>().0.get(&title).cloned();
    if let Some(handle) = existing {
        // Re-focus it if it's still alive; `update` errors if it was closed.
        if handle.update(cx, |_, window, _| window.activate_window()).is_ok() {
            return;
        }
        cx.default_global::<OpenDialogs>().0.remove(&title);   // stale → forget it
    }
    // … open the window, then record its handle:
    // let handle = cx.open_window(…)?;
    // cx.default_global::<OpenDialogs>().0.insert(title, handle.into());
}
```

Note the **self-healing** trick: a closed window leaves a stale handle in the
map; the next open attempt tries to `update` it, gets `Err` (window gone),
removes it, and proceeds to open fresh. No explicit cleanup-on-close needed.

## Globals and re-rendering

Setting a global does **not** automatically repaint views. If a view's
appearance depends on a global, you must trigger a render after changing it.
Two approaches:

1. After mutating the global, call `cx.refresh_windows()` to repaint
   everything (blunt but simple — fine for a theme change).
2. Mirror the relevant bit into an entity you observe, and notify that.

```rust
// e.g. after applying a theme stored as a global:
cx.refresh_windows();   // push the new palette to every open window
```

This app applies its theme by writing into `gpui-component`'s theme global and
then calling `cx.refresh_windows()` so the main window repaints with the new
colors immediately (otherwise it would only update on the next natural
notify).

## When to use a global vs an entity

| Use a **global** when… | Use an **entity** when… |
|------------------------|--------------------------|
| It's a true app-wide singleton (theme, settings, a registry). | It's owned by some part of the UI, even if shared by a few views. |
| Many unrelated places read it and you don't want to thread a handle everywhere. | You want observation/events (`observe`, `subscribe`) and `notify`-driven re-render. |
| It's framework-level plumbing (window registry, font registry). | It's your application/domain state. |

Default to entities (explicit, observable, testable). Reach for globals for the
handful of things that are legitimately ambient.

The final chapter assembles everything into a small but complete app:
[Putting it together](12-a-complete-app.md).
