# Chapter 8 — Focus and text input

Keyboard input and action dispatch both depend on **focus**. This chapter
covers focus handles, making an element focusable, and the practical reality
of text input.

## Focus handles

A `FocusHandle` identifies a focusable spot. You create one (usually once, in
your view's constructor) and store it:

```rust
use gpui::{Context, FocusHandle, Focusable};

struct SearchBar {
    focus_handle: FocusHandle,
    query: String,
}

impl SearchBar {
    fn new(cx: &mut Context<Self>) -> Self {
        Self {
            focus_handle: cx.focus_handle(),
            query: String::new(),
        }
    }
}
```

Attach it to an element with `.track_focus`, which makes that subtree
focusable and ties keyboard events / action context to it:

```rust
impl Render for SearchBar {
    fn render(&mut self, _w: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        gpui::div()
            .track_focus(&self.focus_handle)     // this subtree can hold focus
            .key_context("Search")               // (optional) scope key bindings
            .child(self.query.clone())
    }
}
```

Move focus programmatically with the window:

```rust
self.focus_handle.focus(window, cx);     // give this element focus
let focused: bool = self.focus_handle.is_focused(window);
```

A view can advertise its focus target by implementing `Focusable`:

```rust
impl Focusable for SearchBar {
    fn focus_handle(&self, _cx: &gpui::App) -> FocusHandle {
        self.focus_handle.clone()
    }
}
```

This lets the framework (and parent views) ask "where should focus go for this
view?" — useful when you push a panel and want its input focused.

## Reacting to key events directly

For low-level handling you can attach `on_key_down`/`on_key_up` to a focusable
element. But for app commands, prefer **actions** ([Chapter 7](07-actions-and-keybindings.md)) —
they're declarative, rebindable, and respect context. Reserve raw key handlers
for things actions don't model well.

```rust
gpui::div()
    .track_focus(&self.focus_handle)
    .on_key_down(cx.listener(|this, ev: &gpui::KeyDownEvent, _window, cx| {
        if ev.keystroke.key == "escape" {
            this.query.clear();
            cx.notify();
        }
    }));
```

## Text input: use a widget library

Here's the honest truth: **core GPUI does not ship a ready-made text field.**
A real text input is a lot of work — caret, selection, IME, clipboard,
shaping, scrolling. Building one from `track_focus` + key handlers is a
substantial project.

In practice you use the companion **`gpui-component`** library's `InputState`
+ `Input`:

```rust
use gpui_component::input::{Input, InputState};

struct Form {
    name: gpui::Entity<InputState>,
}

impl Form {
    fn new(window: &mut gpui::Window, cx: &mut gpui::Context<Self>) -> Self {
        Self {
            // InputState owns the text + caret; it's its own entity.
            name: cx.new(|cx| InputState::new(window, cx).placeholder("Your name")),
        }
    }

    fn submit(&mut self, cx: &mut gpui::Context<Self>) {
        let value = self.name.read(cx).value().to_string();
        // … use `value` …
    }
}

impl gpui::Render for Form {
    fn render(&mut self, _w: &mut gpui::Window, _cx: &mut gpui::Context<Self>) -> impl gpui::IntoElement {
        // Render the input by handing it the state entity:
        Input::new(&self.name)
    }
}
```

Notes from real use:

- `InputState::new(window, cx)` needs a `&mut Window`, so create inputs where
  a window is available (a view constructor that receives `window`, or inside
  `cx.new(|cx| …)` within a `render`/window callback).
- Configure it fluently: `.placeholder("…")`, `.multi_line(true)`,
  `.auto_grow(min_rows, max_rows)`.
- Read with `self.input.read(cx).value()`; set with
  `self.input.update(cx, |st, cx| st.set_value("new", window, cx))`.
- To react to typing live, subscribe to its `InputEvent::Change`:

```rust
use gpui_component::input::InputEvent;

let _sub = cx.subscribe(&self.search, |_this, _emitter, ev: &InputEvent, cx| {
    if matches!(ev, InputEvent::Change) {
        cx.notify();          // re-render to apply the new filter text
    }
});
```

`gpui-component` also gives you searchable selects/comboboxes
(`SelectState`/`Select`), checkboxes, buttons, tables, dialogs, and more — all
built on the core primitives in this book. When you need a standard widget,
look there before building from scratch.

## Focus + actions, together

Because actions ride the focus tree, the typical pattern is:

1. Give your input/panel a `FocusHandle` and `.track_focus(&handle)`.
2. Give that subtree a `.key_context("X")`.
3. Bind shortcuts with `Some("X")` so they only fire when focused there.
4. Move focus on relevant events (e.g. an `AppEvent::FocusQuery` handler calls
   `self.query_handle.focus(window, cx)` — which is why such handlers use
   `subscribe_in`, to get a `Window`).

With focus understood, let's open more surfaces:
[windows and dialogs](09-windows-and-dialogs.md).
