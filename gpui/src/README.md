# Developing Applications with GPUI

GPUI is the GPU-accelerated, Rust-native UI framework that powers the
[Zed](https://zed.dev) editor. It is fast, immediate-ish in feel but
retained in structure, and unapologetically Rust: state lives in your own
types, the borrow checker is your friend, and there is no virtual DOM, no
macro DSL for layout, and no separate template language.

The catch: **GPUI has almost no published documentation.** This book fills
that gap. It is written for a developer who is comfortable with Rust but has
never touched GPUI, and it covers the framework's big ideas with working,
idiomatic examples. Every example is distilled from real, compiling code in
a production GPUI application (the YAAMS TK2 asset browser in this repo), so
the patterns are the ones you will actually use.

> **Which GPUI?** This book targets the `gpui` crate from the
> [zed-industries/zed](https://github.com/zed-industries/zed) repository (the
> git dependency, not an old crates.io release — the API differs). Every
> example is verified against two pinned revisions:
>
> - **gpui / gpui_platform** rev `62c01be72eb72b4a87f13d1fcdb8217a077950e9`
> - **gpui-component** rev `6b8d9773392a627eed4e9b93d3b840d43e7fdf1f`
>
> Two consequences of targeting the git revision rather than crates.io, both
> covered in [Chapter 1](01-introduction.md):
>
> 1. **The platform is a separate crate.** This revision has no
>    `gpui::Application::new()`; you boot the app through
>    `gpui_platform::application()`. This is the same on macOS, Windows, and
>    Linux — only the Cargo features differ (Linux needs `wayland`/`x11`).
> 2. **Pin the rev.** An unpinned `git` dependency floats to `main` and
>    examples can stop compiling overnight.
>
> Method names are otherwise stable across nearby revisions; if something
> doesn't compile, check the signature in `crates/gpui/src/` of your checkout.
> Where a widget (text input, tables, dialogs) comes from the companion
> [`gpui-component`](https://github.com/longbridge/gpui-component) library
> rather than core GPUI, the text says so.

## How to read this book

Chapters build on each other. If you read 1–7 in order you will understand
the whole mental model; 8–12 are feature deep-dives you can take as needed.

## Table of contents

1. [Introduction: the mental model and your first window](01-introduction.md)
2. [Elements and styling](02-elements-and-styling.md)
3. [Entities and views](03-entities-and-views.md)
4. [Contexts: `App`, `Context`, `Window`, `AsyncApp`](04-contexts.md)
5. [Interactivity: events, listeners, and stateful elements](05-interactivity.md)
6. [State, `notify`, events, and observation](06-state-events-observation.md)
7. [Actions and key bindings](07-actions-and-keybindings.md)
8. [Focus and text input](08-focus-and-input.md)
9. [Windows and dialogs](09-windows-and-dialogs.md)
10. [Async and background work](10-async-and-background-work.md)
11. [Global state](11-global-state.md)
12. [Putting it together: a complete app](12-a-complete-app.md)
13. [Common components (`gpui-component`): buttons, inputs, selects, switches, sliders…](13-common-components.md)
14. [Theming and icons](14-theming-and-icons.md)

## The five-minute summary

If you remember nothing else:

- **Elements** are cheap, throwaway descriptions of what to draw. You build
  them every frame from your state. `div()` is the workhorse; you style it
  with chained, Tailwind-flavored methods (`.flex().gap_2().bg(...)`).
- **Entities** (`Entity<T>`) are long-lived, reference-counted handles to
  *your* state, owned by GPUI. You create one with `cx.new(|cx| MyState {…})`.
- A **view** is just an entity whose type implements `Render`. `render`
  returns an element tree.
- **Contexts** (`App`, `Context<V>`, `Window`, `AsyncApp`) are your handle to
  the framework. You almost never store them; they're passed into every
  callback.
- You change the UI by mutating your state and calling **`cx.notify()`**.
  GPUI re-runs `render` for you. There is no `setState`, no diffing you write.
- **Events** flow between entities (`EventEmitter` + `cx.subscribe`), and
  **actions** flow up the focus tree (`actions!` + `bind_keys` +
  `on_action`).

Turn to [Chapter 1](01-introduction.md) to see it in motion.
