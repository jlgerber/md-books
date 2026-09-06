# Chapter 12 — Putting it together: a complete app

Let's build a small but real **todo app** that exercises everything: an entity
holding state, a root view, child components, interactivity, a text input
(via `gpui-component`), an action with a key binding, and the `notify` loop.

It's intentionally compact, but the structure scales: swap `AppState` for your
domain model, add views, and you have an application.

## Cargo setup

```toml
[dependencies]
gpui = { git = "https://github.com/zed-industries/zed" }
gpui-component = { git = "https://github.com/longbridge/gpui-component" }
```

(`gpui-component` gives us a ready-made text `Input`; everything else is core
GPUI. Pin both to a `rev`.)

## The whole program

```rust
use gpui::{
    actions, div, px, rgb, App, AppContext, Bounds, Context, Entity,
    InteractiveElement as _, IntoElement, KeyBinding, ParentElement, Render,
    StatefulInteractiveElement as _, Styled, Window, WindowBounds, WindowOptions,
    prelude::FluentBuilder as _, size,
};
use gpui_component::{
    Root,
    button::{Button, ButtonVariants as _},
    input::{Input, InputEvent, InputState},
};

actions!(todo, [AddTodo]);

// ---------------------------------------------------------------------------
// State
// ---------------------------------------------------------------------------

struct Todo {
    text: String,
    done: bool,
}

struct TodoApp {
    todos: Vec<Todo>,
    input: Entity<InputState>,
    _subs: Vec<gpui::Subscription>,
}

impl TodoApp {
    fn new(window: &mut Window, cx: &mut Context<Self>) -> Self {
        let input = cx.new(|cx| {
            InputState::new(window, cx).placeholder("What needs doing?")
        });
        // Re-render as the user types (so the Add button can enable/disable).
        let sub = cx.subscribe(&input, |_this, _e, ev: &InputEvent, cx| {
            if matches!(ev, InputEvent::Change) {
                cx.notify();
            }
        });
        Self { todos: Vec::new(), input, _subs: vec![sub] }
    }

    // Add the current input text as a todo and clear the field.
    fn add(&mut self, _: &AddTodo, window: &mut Window, cx: &mut Context<Self>) {
        let text = self.input.read(cx).value().trim().to_string();
        if text.is_empty() {
            return;
        }
        self.todos.push(Todo { text, done: false });
        self.input.update(cx, |st, cx| st.set_value("", window, cx));
        cx.notify();
    }

    fn toggle(&mut self, ix: usize, cx: &mut Context<Self>) {
        if let Some(t) = self.todos.get_mut(ix) {
            t.done = !t.done;
            cx.notify();
        }
    }

    fn remove(&mut self, ix: usize, cx: &mut Context<Self>) {
        if ix < self.todos.len() {
            self.todos.remove(ix);
            cx.notify();
        }
    }
}

// ---------------------------------------------------------------------------
// View
// ---------------------------------------------------------------------------

impl Render for TodoApp {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let has_text = !self.input.read(cx).value().trim().is_empty();
        let remaining = self.todos.iter().filter(|t| !t.done).count();

        div()
            .flex().flex_col().size_full().gap_3().p_6()
            .bg(rgb(0x1e1e2e)).text_color(rgb(0xcdd6f4))
            // Handle the AddTodo action when this view (or a child) is focused:
            .on_action(cx.listener(Self::add))
            // ---- header ----
            .child(div().text_xl().child("Todos"))
            // ---- add row: input + button ----
            .child(
                div()
                    .flex().flex_row().gap_2().items_center()
                    .child(div().flex_1().child(Input::new(&self.input)))
                    .child(
                        Button::new("add")
                            .label("Add")
                            .primary()
                            .disabled(!has_text)
                            // Dispatch the action so the keyboard path and the
                            // button path share one code path:
                            .on_click(cx.listener(|_this, _ev, window, cx| {
                                window.dispatch_action(Box::new(AddTodo), cx);
                            })),
                    ),
            )
            // ---- the list ----
            .child(self.render_list(cx))
            // ---- footer ----
            .child(
                div()
                    .text_sm().text_color(rgb(0x9399b2))
                    .child(format!("{remaining} remaining")),
            )
    }
}

impl TodoApp {
    fn render_list(&self, cx: &Context<Self>) -> impl IntoElement {
        if self.todos.is_empty() {
            return div().text_color(rgb(0x6c7086)).child("Nothing yet — add one above.")
                .into_any_element();
        }
        let mut list = div().flex().flex_col().gap_1();
        for (ix, todo) in self.todos.iter().enumerate() {
            list = list.child(
                div()
                    .flex().flex_row().items_center().gap_2()
                    .px_2().py_1().rounded_md()
                    .hover(|s| s.bg(rgb(0x313244)))
                    // Click the row text to toggle done:
                    .child(
                        div()
                            .id(("toggle", ix))
                            .flex_1().cursor_pointer()
                            .when(todo.done, |d| d.text_color(rgb(0x6c7086)))   // dim when done
                            .child(if todo.done { format!("✓ {}", todo.text) } else { todo.text.clone() })
                            .on_click(cx.listener(move |this, _ev, _window, cx| this.toggle(ix, cx))),
                    )
                    // ✕ to delete:
                    .child(
                        div()
                            .id(("del", ix))
                            .cursor_pointer()
                            .text_color(rgb(0x6c7086))
                            .hover(|s| s.text_color(rgb(0xf38ba8)))
                            .child("✕")
                            .on_click(cx.listener(move |this, _ev, _window, cx| this.remove(ix, cx))),
                    ),
            );
        }
        list.into_any_element()
    }
}

// ---------------------------------------------------------------------------
// Bootstrap
// ---------------------------------------------------------------------------

fn main() {
    gpui_platform::application().run(|cx: &mut App| {
        gpui_component::init(cx);     // sets up themes, inputs, dialogs, etc.

        // Enter adds a todo. (Bound globally here; scope with a key context
        // if Enter should mean something else elsewhere.)
        cx.bind_keys([KeyBinding::new("enter", AddTodo, None)]);

        let bounds = Bounds::centered(None, size(px(420.), px(520.)), cx);
        cx.open_window(
            WindowOptions {
                window_bounds: Some(WindowBounds::Windowed(bounds)),
                ..Default::default()
            },
            |window, cx| {
                let app = cx.new(|cx| TodoApp::new(window, cx));
                // Wrap in Root so gpui-component inputs/overlays work:
                cx.new(|cx| Root::new(app.into(), window, cx))
            },
        )
        .unwrap();

        cx.activate(true);
    });
}
```

## What each part demonstrates

- **Entity + view** — `TodoApp` is the state; implementing `Render` makes it a
  view. `cx.new(|cx| TodoApp::new(window, cx))` allocates it (Ch. 3).
- **Text input** — `InputState`/`Input` from `gpui-component`, created with a
  window, read with `.read(cx).value()`, cleared with `.set_value` (Ch. 8).
- **Live re-render on type** — subscribing to `InputEvent::Change` and calling
  `cx.notify()` so the Add button's `disabled` state tracks the field (Ch. 6).
- **Interactivity** — stateful rows with unique ids (`("toggle", ix)`),
  `on_click` listeners that capture `ix`, `hover` styling, `when` for the
  done/undone style (Ch. 2, 5).
- **Actions + keybinding** — `AddTodo` declared with `actions!`, bound to
  `enter`, handled via `.on_action(cx.listener(Self::add))`. The Add *button*
  dispatches the same action so keyboard and mouse share one path (Ch. 7).
- **The notify loop** — every mutation (`add`, `toggle`, `remove`) ends in
  `cx.notify()`; GPUI re-runs `render`; the new tree paints (Ch. 1, 3).
- **Bootstrap** — `gpui_platform::application().run(…)`, `gpui_component::init`,
  `bind_keys`, `open_window` with a `Root` wrapper, `activate` (Ch. 1, 9).

## Where to go from here

Scaling this up is mechanical:

- **Split into views.** Pull the add-row and the list into their own
  `Entity<…>` views holding `Entity<TodoStore>`; have them `observe` the store
  and `update` it. That's the shared-`AppState` pattern (Ch. 3, 6) the asset
  browser uses for its toolbar/table/detail/footer.
- **Persist + load async.** Save todos on a background thread; load with
  `cx.spawn` and fold the result back with `this.update + notify` (Ch. 10).
- **Add dialogs.** An "edit todo" modal via `window.open_dialog` (Ch. 9).
- **Reach for `gpui-component`** for tables, selects, checkboxes, markdown
  (`TextView`), tooltips, notifications — all built on the primitives here.

That's GPUI: state in your own types, UI as a function of that state,
mutation scoped through contexts, and `notify` closing the loop. No DSL, no
hidden runtime — just Rust drawing pixels on the GPU.

---

*Back to the [table of contents](README.md).*
