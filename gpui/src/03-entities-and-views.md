# Chapter 3 — Entities and views

This is the chapter where GPUI's design clicks. Elements are throwaway;
**entities** are where your application actually lives.

## What an entity is

An `Entity<T>` is a cheap, cloneable, reference-counted **handle** to a value
of type `T` that *GPUI owns*. Your `T` can be anything — a struct of UI state,
a data model, a controller. GPUI stores it; you hold handles.

```rust
struct Counter {
    count: i32,
}

// Create one. The closure receives a Context<Counter> so the value can,
// e.g., spawn tasks or subscribe during construction.
let counter: Entity<Counter> = cx.new(|_cx| Counter { count: 0 });
```

Key properties:

- **Cloning an `Entity<T>` is cheap** (it bumps a refcount). Clone it freely to
  capture in closures, store in other structs, pass to workers.
- You **cannot** dereference it to `&T` directly. Access goes through a
  context (next section).
- GPUI keeps the value alive as long as a strong handle exists. A
  `WeakEntity<T>` (from `entity.downgrade()`) doesn't keep it alive — use weak
  handles in long-lived async tasks to avoid leaks.

## Reading and updating an entity

You touch an entity's value through a context (`App` or `Context<V>`):

```rust
// Read: borrow &T for the duration of the closure / call.
let n = counter.read(cx).count;

// Update: get exclusive &mut T plus a Context<Counter> for side effects.
counter.update(cx, |counter, cx| {
    counter.count += 1;
    cx.notify();          // ask GPUI to re-render anything showing this entity
});
```

`read` gives `&T`. `update` gives `(&mut T, &mut Context<T>)`. The closure form
is what enforces scoped, non-overlapping mutation — there's never a dangling
`&mut` into GPUI-owned storage.

> Multiple `read`s can coexist (shared borrows of `cx`), but `update` takes
> `&mut cx`. If the borrow checker complains, you're usually trying to hold a
> `read` borrow across an `update`; pull the value out (`.clone()` / copy)
> first, then update.

## A view = an entity that can render

A **view** is simply an entity whose `T` implements `Render`. There's no
separate "View" type; rendering is a capability you add to your state.

```rust
use gpui::{
    div, px, rgb, Context, IntoElement, InteractiveElement, ParentElement,
    Render, Styled, Window,
};

struct Counter {
    count: i32,
}

impl Render for Counter {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        // `render` takes &mut self, so you read your state to build the tree.
        div()
            .flex().flex_col().items_center().gap_4().p_8()
            .child(div().text_2xl().child(format!("{}", self.count)))
            .child(
                div()
                    .id("inc")                 // stateful elements need an id (Ch. 5)
                    .px_4().py_2().rounded_md().bg(rgb(0x89b4fa)).text_color(rgb(0x11111b))
                    .cursor_pointer()
                    // cx.listener wraps a handler so it gets &mut self back:
                    .on_click(cx.listener(|this, _event, _window, cx| {
                        this.count += 1;
                        cx.notify();
                    }))
                    .child("Increment"),
            )
    }
}
```

The flow on click: GPUI calls your listener with `&mut Counter`, you mutate
`count`, you call `cx.notify()`, GPUI re-runs `render`, the new count paints.
That's the entire reactive loop. (We cover `cx.listener` and interactivity
fully in [Chapter 5](05-interactivity.md).)

## Composing views

Views nest by holding `Entity<Child>` and rendering them as children. Build
child entities with `cx.new` and store the handles in the parent:

```rust
struct Sidebar { /* … */ }
impl Render for Sidebar {
    fn render(&mut self, _w: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        div().w(px(220.)).h_full().bg(rgb(0x181825)).child("sidebar")
    }
}

struct Workspace {
    sidebar: Entity<Sidebar>,
    counter: Entity<Counter>,
}

impl Workspace {
    fn new(cx: &mut Context<Self>) -> Self {
        Self {
            sidebar: cx.new(|_| Sidebar { /* … */ }),
            counter: cx.new(|_| Counter { count: 0 }),
        }
    }
}

impl Render for Workspace {
    fn render(&mut self, _w: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .flex().flex_row().size_full()
            // An Entity<V: Render> is itself a valid child — clone the handle.
            .child(self.sidebar.clone())
            .child(div().flex_1().child(self.counter.clone()))
    }
}
```

An `Entity<V: Render>` implements `IntoElement`, so `.child(self.sidebar.clone())`
just works. The child renders itself and manages its own state. This is how you
build large UIs: a tree of views, each owning a slice of state.

### Shared state across views

Often several views need the same state — a selection, a document, app
settings. The idiomatic move is **one state entity, many views holding a
clone of the handle**:

```rust
let app_state = cx.new(|_| AppState::default());

let toolbar = cx.new(|_| Toolbar { state: app_state.clone() });
let table   = cx.new(|_| Table   { state: app_state.clone() });
let footer  = cx.new(|_| Footer  { state: app_state.clone() });
```

Each view reads `self.state.read(cx)` in `render`, and any view can
`self.state.update(cx, |s, cx| { …; cx.notify(); })`. When the shared state
notifies, every view observing it re-renders. (How a view "observes" another
entity is [Chapter 6](06-state-events-observation.md).) This `AppState`
pattern is exactly how the asset browser in this repo wires its toolbar,
results table, detail pane, and footer together.

## `cx.new` vs `cx.entity()`

- `cx.new(|cx| T { … })` — **create** a new entity. The closure's `cx` is a
  `Context<T>` for the new entity, so you can subscribe/spawn during init.
- Inside a view's method, `cx.entity()` returns the **current** view's
  `Entity<Self>` — useful when you need to hand your own handle to a callback
  or a child. `cx.weak_entity()` gives the weak form.

## When to make something an entity

- **Make it an entity/view** if it has persistent state that outlives a single
  frame, needs to receive events, spawns async work, or is shared.
- **Make it a `RenderOnce` component or a plain `impl IntoElement` function**
  (Chapter 2) if it's pure "props in, elements out" with no lifetime.

A typical app has a handful of entities (app state + a few major views) and
many stateless components.

Next: the contexts that thread through all of this —
[Chapter 4](04-contexts.md).
