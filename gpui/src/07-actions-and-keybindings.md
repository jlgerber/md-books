# Chapter 7 — Actions and key bindings

Actions are GPUI's mechanism for **commands**: named, dispatchable units of
behavior that keyboard shortcuts, menu items, and code all trigger the same
way. They flow up the **focus tree**, so the same keystroke can mean different
things depending on what's focused — exactly what you want in a real app.

## Declaring actions

Two ways, depending on whether the action carries data.

**Unit actions** — use the `actions!` macro to declare several at once:

```rust
use gpui::actions;

actions!(
    my_app,                 // namespace (groups them; appears in keymaps)
    [
        Quit,
        Refresh,
        ToggleSidebar,
        FocusSearch,
    ]
);
```

This generates a zero-field struct per name (`Quit`, `Refresh`, …), each
implementing the `Action` trait.

**Data-carrying actions** — derive `Action`:

```rust
use gpui::Action;
use serde::Deserialize;

#[derive(Action, Clone, PartialEq, Eq, Deserialize)]
#[action(namespace = my_app, no_json)]
struct DeleteItem {
    id: u64,
}
```

(`no_json` skips JSON keymap deserialization for actions you only dispatch
from code; omit it if you want to bind them in a JSON keymap with arguments.)

## Binding keys

Register key → action mappings, usually once at startup, via `cx.bind_keys`:

```rust
use gpui::KeyBinding;

cx.bind_keys([
    KeyBinding::new("cmd-q",  Quit,          None),
    KeyBinding::new("ctrl-q", Quit,          None),
    KeyBinding::new("cmd-r",  Refresh,       None),
    KeyBinding::new("ctrl-r", Refresh,       None),
    KeyBinding::new("cmd-b",  ToggleSidebar, None),
]);
```

The third argument is a **key context** (or `None` for global). More on that
below. Keystroke syntax is `modifier-key`: `cmd`, `ctrl`, `alt`, `shift`,
plus the key (`q`, `tab`, `enter`, `escape`, `up`, `pageup`, `/`, …).

## Handling actions

Two places to handle, mirroring contexts:

**Globally**, on `App` (typical for app-wide commands):

```rust
cx.on_action(|_: &Quit, cx: &mut App| cx.quit());

// Capture state in the handler closure:
let state = app_state.clone();
cx.on_action(move |_: &Refresh, cx| {
    state.update(cx, |s, cx| s.refresh(cx));
});
```

**On a view**, via `.on_action` on its root element (the action is only
handled when this view — or a descendant — is in the focus path):

```rust
impl Render for Editor {
    fn render(&mut self, _w: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        gpui::div()
            .on_action(cx.listener(Self::save))          // bind a method
            .on_action(cx.listener(|this, _: &DeleteItem, _w, cx| { /* … */ }))
            .child("editor")
    }
}

impl Editor {
    fn save(&mut self, _: &Save, _window: &mut Window, cx: &mut Context<Self>) {
        // … cx.notify();
    }
}
```

## Key contexts: scoping shortcuts to what's focused

A binding's third argument scopes it to a **key context** — a string attached
to an element subtree. The binding only fires when an element carrying that
context (or a descendant) is focused. Set a context with `.key_context`:

```rust
gpui::div().key_context("Editor").track_focus(&self.focus_handle)
```

Now `KeyBinding::new("cmd-s", Save, Some("Editor"))` only triggers `Save` when
the editor subtree is focused — `cmd-s` elsewhere can mean something else.
Contexts support predicates like `"Editor && !Modal"`.

This is how, for example, a table can bind `tab` to "next column" only while
the table is focused (`KeyBinding::new("tab", SelectNextColumn, Some("Table"))`),
without stealing `tab` from text inputs elsewhere.

## Overriding a binding and falling through with `cx.propagate()`

When two bindings match the same keystroke in the same context (e.g. you add a
`tab` binding in a context that already has one), GPUI dispatches them in
order. If your handler decides it doesn't want the event, call
`cx.propagate()` and the next matching action runs:

```rust
cx.on_action(move |_: &NextDetailTab, cx| {
    let st = state.read(cx);
    if st.detail_open && st.has_subtabs() {
        state.update(cx, |s, cx| s.cycle_tab(cx));   // consume it
    } else {
        cx.propagate();                                // let the table handle Tab
    }
});
```

`App::propagate()` continues dispatch; without it the action is consumed. This
app uses exactly this to make `Tab` cycle detail-pane sub-tabs *only when the
pane is open*, otherwise falling through to the table's column navigation.

## Dispatching from code

You don't need a keypress to fire an action — dispatch it programmatically
(e.g. from a menu button):

```rust
window.dispatch_action(Box::new(Refresh), cx);
```

## Putting it together

```rust
use gpui::{actions, div, App, Context, IntoElement, KeyBinding,
           ParentElement, Render, Styled, Window};

actions!(notes, [AddNote, Quit]);

struct Notes { count: usize }

impl Notes {
    fn add(&mut self, _: &AddNote, _w: &mut Window, cx: &mut Context<Self>) {
        self.count += 1;
        cx.notify();
    }
}

impl Render for Notes {
    fn render(&mut self, _w: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .on_action(cx.listener(Self::add))           // handle AddNote here
            .child(format!("notes: {}", self.count))
    }
}

fn main() {
    gpui_platform::application().run(|cx: &mut App| {
        cx.bind_keys([
            KeyBinding::new("cmd-n", AddNote, None),
            KeyBinding::new("cmd-q", Quit, None),
        ]);
        cx.on_action(|_: &Quit, cx| cx.quit());
        // … open a window whose root view is Notes …
        let _ = cx;
    });
}
```

| Concept | API |
|---------|-----|
| Declare unit actions | `actions!(ns, [A, B])` |
| Declare data action | `#[derive(Action)] #[action(namespace = ns)]` |
| Bind a key | `cx.bind_keys([KeyBinding::new("cmd-s", Save, ctx)])` |
| Handle app-wide | `cx.on_action(|a, cx| …)` |
| Handle in a view | `.on_action(cx.listener(Self::handler))` |
| Scope to focus | `.key_context("X")` + `Some("X")` on the binding |
| Pass to the next handler | `cx.propagate()` |
| Fire from code | `window.dispatch_action(Box::new(A), cx)` |

Actions ride the focus tree — which means you need to understand focus. That's
[Chapter 8](08-focus-and-input.md).
