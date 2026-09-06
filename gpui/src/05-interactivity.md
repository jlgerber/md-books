# Chapter 5 — Interactivity: events, listeners, and stateful elements

So far our UIs draw; now let's make them respond. GPUI splits interactivity
into two traits an element can offer:

- **`InteractiveElement`** — hover styling, mouse-down/up/move handlers, scroll,
  tooltips, context menus.
- **`StatefulInteractiveElement`** — *clicks* and anything requiring identity
  across frames. This requires the element to have an **`id`**.

Bring them into scope:

```rust
use gpui::{InteractiveElement as _, StatefulInteractiveElement as _};
```

## Why clicks need an `.id()`

A click is two events (down then up) that must be correlated to the *same*
element across frames. Because elements are rebuilt every frame, GPUI needs a
stable identity to know "the thing you pressed is the thing you released."
That identity is the `id`:

```rust
div()
    .id("save-button")          // now it's stateful → .on_click is available
    .on_click(cx.listener(|this, _ev, _window, cx| { /* … */ }))
    .child("Save");
```

Without `.id()`, calling `.on_click` won't compile (the method lives on
`StatefulInteractiveElement`). For lists, make the id unique per item with a
tuple: `.id(("row", index))`.

> **`.id()` must be unique among siblings within a render.** Reusing the same
> id for two visible elements confuses hit-testing and state. For dynamic
> lists, `("note-row", ix)` style ids are the norm.

## Mouse handlers without an id

Down/up/move don't need identity, so they're on plain `InteractiveElement`:

```rust
div()
    .on_mouse_down(gpui::MouseButton::Left, |event, window, cx| {
        // event: &MouseDownEvent  (position, modifiers, click_count, …)
    })
    .on_mouse_up(gpui::MouseButton::Left, |_event, _window, _cx| { /* … */ })
    .on_mouse_move(|_event, _window, _cx| { /* … */ });
```

A common pattern: a clear (✕) button built as a styled div with
`on_mouse_down` rather than `on_click`, because it doesn't need click
correlation:

```rust
let input = self.search.clone();
div()
    .id("clear")
    .cursor_pointer()
    .child("✕")
    .on_mouse_down(gpui::MouseButton::Left, move |_ev, window, cx| {
        input.update(cx, |state, cx| state.set_value("", window, cx));
    });
```

## Hover

`.hover(closure)` returns a styled variant applied while the pointer is over
the element. It's pure styling — no callback:

```rust
div()
    .rounded_md()
    .hover(|style| style.bg(rgb(0x313244)))   // background only while hovered
    .child("hover me");
```

For active/pressed styling there's `.active(|s| …)` (applies while the mouse
button is held on a stateful element) — handy for buttons:

```rust
div()
    .id("btn")
    .bg(rgb(0x89b4fa))
    .hover(|s| s.bg(rgb(0x74c7ec)))
    .active(|s| s.bg(rgb(0x6c9fe0)))
    .on_click(cx.listener(|_this, _ev, _w, _cx| { /* … */ }));
```

## The `ClickEvent`

`on_click`'s event carries useful info — modifiers, click count (for
double-click), the down/up positions:

```rust
.on_click(cx.listener(|this, event: &gpui::ClickEvent, _window, cx| {
    let multi = event.modifiers().secondary();   // Cmd on macOS, Ctrl elsewhere
    let shift = event.modifiers().shift;
    // e.g. ctrl-click toggles, shift-click range-selects:
    this.select(multi, shift, cx);
}))
```

## Tooltips and context menus

Tooltips are a closure that builds the tooltip element on demand (so they cost
nothing until shown). Context menus and tooltips from `gpui-component` are the
common path:

```rust
// Tooltip (gpui-component helper):
div()
    .id("info")
    .tooltip(move |window, cx| {
        gpui_component::tooltip::Tooltip::new("More info").build(window, cx)
    })
    .child("ⓘ");

// Right-click context menu (gpui-component's ContextMenuExt):
use gpui_component::menu::ContextMenuExt as _;
div()
    .child("right-click me")
    .context_menu(move |menu, _window, _cx| {
        menu.menu("Copy value", Box::new(CopyValue { /* an Action, see Ch. 7 */ }))
    });
```

## A complete interactive list

Tying it together — a selectable list where clicking a row updates state:

```rust
use gpui::{
    div, px, rgb, Context, InteractiveElement as _, IntoElement, ParentElement,
    Render, StatefulInteractiveElement as _, Styled, Window,
};
use gpui::prelude::FluentBuilder as _;

struct FileList {
    files: Vec<String>,
    selected: Option<usize>,
}

impl Render for FileList {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let mut list = div().flex().flex_col().gap_1().p_2();
        for (ix, name) in self.files.iter().enumerate() {
            let is_selected = self.selected == Some(ix);
            list = list.child(
                div()
                    .id(("file", ix))                 // unique per row → clickable
                    .px_2().py_1().rounded_md()
                    .cursor_pointer()
                    .when(is_selected, |d| d.bg(rgb(0x45475a)))
                    .hover(|s| s.bg(rgb(0x313244)))
                    .on_click(cx.listener(move |this, _ev, _window, cx| {
                        this.selected = Some(ix);     // `ix` captured by the closure
                        cx.notify();
                    }))
                    .child(name.clone()),
            );
        }
        list
    }
}
```

Note the `move` on the click closure: each row's listener captures its own
`ix`. The state lives in `FileList.selected`; clicking mutates it and
notifies; the next render restyles the selected row. No event bus, no
callbacks-up-the-tree — just local state and `notify`.

When interaction needs to reach *another* view or component, you graduate to
**events** and **actions** — the next two chapters.
