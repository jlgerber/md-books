# Chapter 9 — Windows and dialogs

You opened a window in Chapter 1. Here we cover window options, multiple
windows, lifecycle, resizing, and the two ways to do "dialogs" — separate OS
windows vs. in-app modal overlays — including a real platform gotcha.

## Opening a window

```rust
use gpui::{Bounds, WindowBounds, WindowOptions, px, size};

let bounds = Bounds::centered(None, size(px(900.), px(640.)), cx);
let handle = cx.open_window(
    WindowOptions {
        window_bounds: Some(WindowBounds::Windowed(bounds)),
        ..Default::default()
    },
    |window, cx| {
        // Build and return the root view entity:
        cx.new(|cx| Workspace::new(window, cx))
    },
).unwrap();
```

`open_window` returns a `WindowHandle<V>` you can keep to address the window
later (activate it, update through it). `Bounds::centered(None, size, cx)`
centers on the primary display; pass a `DisplayId` to target another monitor.

### Common `WindowOptions`

```rust
WindowOptions {
    window_bounds: Some(WindowBounds::Windowed(bounds)),
    titlebar: Some(gpui::TitlebarOptions { /* title, traffic-light pos, … */ ..Default::default() }),
    // On Linux you often want client-side decorations so you can draw your own titlebar:
    #[cfg(target_os = "linux")]
    window_decorations: Some(gpui::WindowDecorations::Client),
    ..Default::default()
}
```

## Window lifecycle

Hook the close button — for a single-window app, quit when it closes so stray
child windows don't keep the process alive:

```rust
cx.open_window(opts, |window, cx| {
    window.on_window_should_close(cx, |_window, cx| {
        cx.quit();
        true        // true = allow the close
    });
    cx.new(|cx| Workspace::new(window, cx))
});
```

## Resizing and the absence of a "move" API

You can resize a window after it's open:

```rust
window.resize(size(px(480.), px(640.)));
```

But note: **`resize` keeps the top-left origin** — there is no public API to
*reposition* a window after creation. If you resize a centered window it will
appear shifted. The practical consequence: choose the right size at
`open_window` time (so `Bounds::centered` lands correctly) rather than opening
small and growing later. This app sizes per-content at open time for exactly
this reason.

## Two flavors of "dialog"

### 1. A separate OS window

Treat a dialog as just another window. Factor a helper:

```rust
fn open_dialog_window<F>(title: impl Into<gpui::SharedString>, w: f32, h: f32, build: F, cx: &mut gpui::App)
where
    F: FnOnce(&mut gpui::Window, &mut gpui::App) -> gpui::AnyView + 'static,
{
    let bounds = gpui::Bounds::centered(None, gpui::size(px(w), px(h)), cx);
    // Defer one tick so this works even when called from inside an action handler:
    cx.spawn(async move |cx| {
        let _ = cx.update(|cx| {
            let _ = cx.open_window(
                gpui::WindowOptions { window_bounds: Some(gpui::WindowBounds::Windowed(bounds)), ..Default::default() },
                move |window, cx| {
                    let view = build(window, cx);
                    // Wrap in gpui-component's Root so overlays/dialogs work inside:
                    cx.new(|cx| gpui_component::Root::new(view, window, cx))
                },
            );
        });
    }).detach();
}
```

This is great for genuinely separate surfaces (preferences, an editor).
**But beware Wayland:** Wayland compositors do *not* let a client position its
own top-level windows. `Bounds::centered` is ignored there, so your separate
window appears wherever the compositor decides (often offset, not centered).
On X11/macOS/Windows it centers fine.

### 2. An in-app modal overlay (recommended for forms)

Because of the Wayland positioning limitation, an **in-app modal** rendered
*inside* the existing window is often better — you control its position
absolutely, and it centers everywhere. `gpui-component` provides this via
`window.open_dialog`:

```rust
use gpui_component::{WindowExt as _, dialog::{DialogFooter, DialogClose, DialogAction}};
use gpui_component::button::{Button, ButtonVariants as _};

// From inside a view's click/listener (cx: &mut Context<V> coerces to &mut App):
window.open_dialog(cx, move |dialog, _window, _cx| {
    dialog
        .title("New Item")
        .child(/* your form element tree, e.g. an Input */ gpui::div())
        .footer(
            DialogFooter::new()
                .child(DialogClose::new().child(Button::new("cancel").label("Cancel").ghost()))
                .child(DialogAction::new().child(Button::new("ok").label("Create").primary())),
        )
        .on_ok(|_ev, _window, cx| {
            // return true to close, false to keep open (e.g. failed validation)
            true
        })
});
```

Things worth knowing about `gpui-component` dialogs:

- The `build` closure runs **every render**, so any stateful form widgets
  (inputs, selects) must be entities you create *before* opening and **capture**
  in the closure — they persist; the closure just re-renders them.
- The dialog renders **no footer by default** — supply `.footer(…)` if you
  want Cancel/OK buttons. `on_ok` returning `false` keeps it open (good for
  validation); `Enter` triggers `on_ok`, the ✕/overlay triggers cancel.
- It requires your window's root to be a `gpui_component::Root` (which is what
  `gpui_component::init(cx)` + wrapping your view in `Root::new` sets up).

## Choosing

- **Separate window** — large, independent surfaces; multi-window workflows.
  Accept that positioning is at the compositor's mercy on Wayland.
- **In-app modal** — forms, confirmations, pickers. Centers reliably
  everywhere; lighter weight; stays within the main window.

Next: doing work without freezing the UI —
[async and background work](10-async-and-background-work.md).
