# Chapter 1 — Introduction: the mental model and your first window

## What GPUI is

GPUI is a GPU-rendered UI framework. Instead of laying out widgets through a
retained widget tree you mutate, you describe the UI as a tree of
**elements** that GPUI rebuilds and re-renders whenever your state changes.
Think React's "UI is a function of state," but the function is plain Rust,
the elements are values you construct with builder methods, and there is no
diff you have to reason about — GPUI re-runs your render function and paints
the result on the GPU.

Three nouns carry the whole framework:

| Noun | Lifetime | What it is |
|------|----------|------------|
| **Element** | one frame | A throwaway description of something to draw (`div()`, text, an image). |
| **Entity** `Entity<T>` | long-lived | A handle to a piece of *your* state that GPUI owns and reference-counts. |
| **View** | long-lived | An entity whose `T: Render` — it knows how to turn itself into elements. |

And one verb: **`cx.notify()`** — "my state changed; re-render me."

## The borrow-checker-friendly design

The thing that surprises people coming from other UI frameworks: GPUI never
hands you a `&mut self` *and* a context that can reach back into `self`. Your
state lives behind an `Entity<T>`. To touch it you go through a context
(`entity.update(cx, |state, cx| { … })`), and inside that closure you have
exclusive access. This is why GPUI apps rarely fight the borrow checker:
mutation is always scoped, and the framework owns the storage.

## Setting up

GPUI is a git dependency. **Pin an explicit `rev`** — the API moves, and an
unpinned `git` line will silently float to whatever `main` is the day you run
`cargo update`, which is how examples stop compiling. Every example in this
book is verified against the two revs below:

```toml
# Cargo.toml
[dependencies]
# The gpui crate itself, pinned to a known-good revision.
gpui = { git = "https://github.com/zed-industries/zed", rev = "62c01be72eb72b4a87f13d1fcdb8217a077950e9" }

# The companion widget library (buttons, inputs, tables, theming — Chapters 13–14).
# Not needed for this chapter's bare-gpui example, but you'll want it soon.
gpui-component = { git = "https://github.com/longbridge/gpui-component", rev = "6b8d9773392a627eed4e9b93d3b840d43e7fdf1f" }

# gpui_platform is the OS event loop, and it's a SEPARATE crate in this revision
# (gpui itself no longer ships one). The Linux backends live behind the
# `wayland`/`x11` features and pull in a Linux-only crate, so they must be
# Linux-gated — enabling them on macOS/Windows is a build error. macOS and
# Windows need no extra features; the backend is chosen automatically by cfg.
[target.'cfg(target_os = "linux")'.dependencies]
gpui_platform = { git = "https://github.com/zed-industries/zed", rev = "62c01be72eb72b4a87f13d1fcdb8217a077950e9", features = ["wayland", "x11"] }

[target.'cfg(not(target_os = "linux"))'.dependencies]
gpui_platform = { git = "https://github.com/zed-industries/zed", rev = "62c01be72eb72b4a87f13d1fcdb8217a077950e9" }
```

> **The split that bites everyone first.** Older crates.io `gpui` (≤ 0.2.x on
> crates.io) had `gpui::Application::new()`. The git revision this book targets
> **moved the platform out into the `gpui_platform` crate and removed
> `Application::new`** — you now construct the app with
> `gpui_platform::application()` (see the program below). This is true on every
> OS: if you copy an `Application::new()` example from elsewhere it will not
> compile against this rev, regardless of platform. That's the API split, not a
> typo. The good news: `gpui_platform::application()` picks the right backend
> (macOS / Windows / Linux) for you, so the boot code below is **identical on
> all three.**

Some platforms also need system libraries: on Linux you'll want
Vulkan/X11/Wayland dev packages; macOS and Windows build out of the box with a
recent toolchain. Check the Zed repo's build docs for the current list.

## Your first window

Here is a complete, runnable GPUI program. It opens a window that displays a
greeting.

```rust
use gpui::{
    App, AppContext, Bounds, Context, IntoElement, ParentElement, Render,
    Styled, Window, WindowBounds, WindowOptions, div, px, rgb, size,
};
// `AppContext` is the trait that provides `cx.new(...)` and `cx.open_window(...)`;
// without it in scope those calls won't resolve. (Alternatively, `use
// gpui::prelude::*;` pulls in `AppContext` plus the element-builder traits —
// `Styled`, `ParentElement`, `IntoElement`, `Render` — in one line. This book
// lists them explicitly so you can see where each method comes from.)

/// Our root view. In GPUI, a "view" is any struct that implements `Render`.
struct Hello {
    name: String,
}

impl Render for Hello {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .flex()
            .size_full()
            .items_center()
            .justify_center()
            .bg(rgb(0x1e1e2e))
            .text_color(rgb(0xcdd6f4))
            .text_xl()
            .child(format!("Hello, {}!", self.name))
    }
}

fn main() {
    // `gpui_platform::application()` builds the app with the right OS backend
    // (macOS / Windows / Linux). In this revision there is no
    // `gpui::Application::new()` — see the note above.
    gpui_platform::application().run(|cx: &mut App| {
        let bounds = Bounds::centered(None, size(px(480.), px(320.)), cx);
        cx.open_window(
            WindowOptions {
                window_bounds: Some(WindowBounds::Windowed(bounds)),
                ..Default::default()
            },
            // The build closure returns the window's *root view* entity.
            |_window, cx| cx.new(|_cx| Hello { name: "GPUI".into() }),
        )
        .unwrap();

        cx.activate(true);
    });
}
```

Walk through it:

1. **`gpui_platform::application().run(|cx: &mut App| { … })`** boots the
   platform event loop and hands you the application context `cx: &mut App`.
   Everything you do at startup (open windows, register fonts, install key
   bindings, set up global state) happens here. (`gpui_platform::application()`
   returns a `gpui::Application` already wired to this OS's backend; you call
   `.run(…)` on it.)
2. **`cx.open_window(options, build)`** creates an OS window. The `build`
   closure receives `(&mut Window, &mut App)` (here written
   `|window, cx|`) and must return the window's **root view** — an
   `Entity<V: Render>`.
3. **`cx.new(|cx| Hello { … })`** allocates our state inside GPUI and returns
   an `Entity<Hello>`. We'll cover `cx.new` properly in Chapter 3.
4. **`render`** is called by GPUI to produce the element tree. We build a
   `div`, style it (flexbox centering, colors, text size), and give it a text
   child. `render` takes `&mut self`, so you can read (and even tweak) your
   state while building the tree.
5. **`cx.activate(true)`** brings the app to the foreground.

Notice what's *absent*: no template file, no component registration, no
manual layout math. Layout is flexbox; styling is method chaining; the view
is a function from state to elements.

## The render cycle

```
            your state mutates
                  │
            cx.notify()
                  │
                  ▼
   GPUI calls view.render(window, cx)  ──►  element tree  ──►  GPU paint
```

You never call `render` yourself. You mutate state and call `cx.notify()`;
GPUI schedules a re-render. If nothing notifies, nothing repaints — renders
are cheap, but they don't happen for free on a timer.

## What's next

You now have a window. [Chapter 2](02-elements-and-styling.md) goes deep on
the element you'll use most — `div` — and the styling system. Then
[Chapter 3](03-entities-and-views.md) explains entities and views properly,
which is where GPUI's design really clicks.
