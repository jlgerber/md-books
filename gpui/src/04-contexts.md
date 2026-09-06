# Chapter 4 — Contexts: `App`, `Context`, `Window`, `AsyncApp`

Contexts are your handle to the framework. Every callback GPUI gives you
receives one or more. You rarely store them — you receive them, use them, and
let them go. The confusing part for newcomers is that there are *several*, so
here's a map.

## The four you'll meet

| Type | Where you get it | What it's for |
|------|------------------|---------------|
| `&mut App` | `Application::run`, action handlers, `RenderOnce::render` | The whole application. Open windows, create entities, set globals, dispatch actions. No "current view." |
| `&mut Context<V>` | a view's `render`, `update`, `cx.new`, listeners | `App` *plus* knowledge of the current entity `V`. Adds `notify`, `emit`, `subscribe`, `spawn`, `entity()`. |
| `&mut Window` | `render`, window callbacks, `*_in` variants | The window: focus, key dispatch, sizing, redraw, spawning window-aware tasks. |
| `&mut AsyncApp` / `AsyncWindowContext` | inside `cx.spawn` | The async-side context you use after an `.await`. |

### `App`

The root. You get it in `gpui_platform::application().run(|cx: &mut App| { … })` and in
global `cx.on_action(|action, cx: &mut App| { … })` handlers. With it you can
`open_window`, `cx.new(...)`, `cx.set_global(...)`, `cx.quit()`,
`cx.activate(true)`, `cx.background_executor()`, etc. It has no notion of a
"current view," so it can't `notify` (there's nothing to notify).

### `Context<V>`

This is the one you'll use most. It `Deref`s to `App`, so anything you can do
with `&mut App` you can do with `&mut Context<V>` — *plus* view-aware powers:

```rust
impl Render for MyView {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        cx.notify();                       // re-render this view
        cx.emit(MyEvent::Changed);         // emit an event (needs EventEmitter)
        let me: gpui::Entity<Self> = cx.entity();   // my own handle
        // cx.subscribe(&other, …), cx.observe(&other, …), cx.spawn(…) …
        gpui::div()
    }
}
```

Because `Context<V>` derefs to `App`, you can pass `cx` (a `&mut Context<V>`)
to any function expecting `&mut App` — Rust coerces it. That's why
`window.open_dialog(cx, …)` works from inside a view even though the method
signature wants `&mut App`.

### `Window`

The window context handles everything window-scoped: keyboard focus
(`window.focus(&handle, cx)`), forcing a repaint (`window.refresh()`),
viewport size, dispatching actions (`window.dispatch_action(…)`), and
lifecycle hooks (`window.on_window_should_close(cx, …)`). `render` receives
`&mut Window` alongside `&mut Context<Self>`.

### `AsyncApp` / `AsyncWindowContext`

When you cross an `await` point you can't keep a `&mut App` (the borrow can't
live across suspension). `cx.spawn` instead gives you an owned async context.
You re-acquire view access by calling `.update` on a (weak) entity handle.
See [Chapter 10](10-async-and-background-work.md).

## The `*_in` variants: getting a `Window` in a callback

Many subscription/observation methods come in two flavors:

- `cx.subscribe(&e, |this, emitter, event, cx| { … })` — no window.
- `cx.subscribe_in(&e, window, |this, emitter, event, window, cx| { … })` —
  the handler also receives a `&mut Window`.

Use the `_in` variant when the handler needs to do window things — move focus,
push a notification, set an input's value (text inputs need a window). This
app uses `subscribe_in` precisely so a "create failed" handler can call
`window.push_notification(…)`.

## `cx.listener`: turning a handler into an element callback

Element callbacks (`.on_click`, `.on_mouse_down`, …) are plain closures with
signatures like `Fn(&ClickEvent, &mut Window, &mut App)`. But you usually want
your *view* back so you can mutate it. `cx.listener` bridges the two:

```rust
.on_click(cx.listener(|this: &mut MyView, event: &ClickEvent, window: &mut Window, cx: &mut Context<MyView>| {
    this.selected = true;
    cx.notify();
}))
```

`cx.listener(f)` returns a closure that, when the event fires, re-acquires
`&mut MyView` and a fresh `Context<MyView>` and calls `f`. You can also pass a
method directly: `.on_action(cx.listener(Self::handle_save))`.

## A worked tour

```rust
use gpui::{div, Context, IntoElement, InteractiveElement, ParentElement, Render, Styled, Window};

struct Panel {
    open: bool,
}

impl Panel {
    // A method using Context<Self> — note it can do everything App can, plus notify.
    fn toggle(&mut self, cx: &mut Context<Self>) {
        self.open = !self.open;
        cx.notify();
    }
}

impl Render for Panel {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("toggle")
            .child(if self.open { "▼ open" } else { "▶ closed" })
            // cx.listener gives us &mut self in the click handler:
            .on_click(cx.listener(|this, _ev, _window, cx| this.toggle(cx)))
    }
}
```

## Mental model

- Need to mutate **my** view and re-render? `Context<V>` + `cx.notify()`.
- Need to do something app-wide (open window, global)? `&mut App` (or `cx`
  derefs to it).
- Need to touch focus/keyboard/repaint? `&mut Window`.
- Crossing an `.await`? `AsyncApp`, and re-enter via `entity.update(cx, …)`.

With contexts mapped, the next chapters use them constantly. On to
[interactivity](05-interactivity.md).
