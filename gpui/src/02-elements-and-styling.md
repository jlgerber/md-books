# Chapter 2 — Elements and styling

Elements are the leaves and branches of what GPUI draws. They are **cheap and
ephemeral**: you construct a fresh element tree on every render and throw it
away. Because they're disposable, you build them directly from your state with
no ceremony.

## `div()` — the universal container

`div()` is GPUI's `<div>`: a flexbox container you style and fill with
children. Almost every layout is divs nesting divs.

```rust
use gpui::{div, px, rgb, IntoElement, ParentElement, Styled};

fn card() -> impl IntoElement {
    div()
        .flex()
        .flex_col()          // vertical stack
        .gap_2()             // space between children
        .p_4()               // padding
        .rounded_md()        // corner radius
        .bg(rgb(0x313244))
        .child("Title")
        .child(
            div()
                .text_sm()
                .text_color(rgb(0xa6adc8))
                .child("A subtitle in a nested div"),
        )
}
```

`.child(x)` appends one child; `.children(iter)` appends many. A child can be
another element, or anything that implements `IntoElement` — including a
`String`, a `&str`, or a `SharedString` (those render as text).

## The `Styled` trait: Tailwind for Rust

Styling methods come from the `Styled` trait, and the vocabulary is borrowed
from Tailwind CSS. You'll reach for these constantly:

```rust
div()
    // ---- flexbox ----
    .flex().flex_row().flex_col()
    .items_center().justify_between().justify_center()
    .gap_1().gap_2().gap_4()
    .flex_1()          // grow to fill
    .flex_shrink_0()   // never shrink

    // ---- sizing ----
    .w_full().h_full().size_full()
    .w(px(240.)).h(px(32.))
    .min_w(px(0.)).max_h(px(260.))

    // ---- spacing ----
    .p_2().px_3().py_1().pl(px(12.))
    .m_2().mt_1().ml_2()

    // ---- color & text ----
    .bg(rgb(0x1e1e2e))
    .text_color(rgb(0xcdd6f4))
    .text_xs().text_sm().text_lg()
    .font_weight(gpui::FontWeight::MEDIUM)

    // ---- borders & radius ----
    .border_1().border_b_1().border_color(rgb(0x45475a))
    .rounded_md().rounded_full()

    // ---- overflow ----
    .overflow_hidden().truncate().text_ellipsis().whitespace_nowrap();
```

Sizes are a mix of fixed (`px(…)`), fractional (`.w_full()`), and spacing-scale
helpers (`p_2` ≈ 0.5rem). Mix freely.

### Conditional styling: `when` / `when_some`

You don't write `if` around your builder chain — you use the `FluentBuilder`
combinators (bring the trait into scope with
`use gpui::prelude::FluentBuilder as _;`):

```rust
use gpui::prelude::FluentBuilder as _;

div()
    .px_3()
    .when(is_active, |d| d.bg(rgb(0x45475a)))          // apply only if true
    .when(!is_active, |d| d.text_color(rgb(0x6c7086)))
    .when_some(label, |d, text| d.child(text))          // apply if Some, passing the value
    .child("row");
```

`when` keeps the chain fluent and avoids intermediate `let mut el = …;`
reassignment. `when_some` is perfect for optional content.

## Color

`rgb(0xRRGGBB)` and `rgba(0xRRGGBBAA)` are the quick path. For computed colors
GPUI uses `Hsla` (hue/saturation/lightness/alpha), which is handy for
deriving shades:

```rust
use gpui::{Hsla, Rgba};

// Parse a hex string into Hsla:
let base: Hsla = Rgba { r: 0.12, g: 0.12, b: 0.18, a: 1.0 }.into();

// Nudge lightness to make a "raised card" shade — a real trick from this app:
let mut raised = base;
raised.l = (base.l + 0.04).clamp(0.0, 1.0);

div().bg(base).child(div().bg(raised).child("a slightly lighter card"));
```

In a themed app you typically pull colors from a theme object rather than
hard-coding hex; see Chapter 9 and the companion `gpui-component`'s
`cx.theme()`.

## Text

The simplest text is a string child. For styled runs, wrap text in a styled
`div`. For **rich text / Markdown**, core GPUI gives you primitives, but the
practical answer in most apps is `gpui-component`'s `TextView`:

```rust
// from gpui-component:
use gpui_component::text::TextView;
TextView::markdown(("note-body", id), "# Heading\n\nBody **bold**.");
```

(That's exactly how this app renders note/task bodies.)

## `IntoElement` and `RenderOnce`: making reusable components

A function returning `impl IntoElement` (like `card()` above) is the lightest
way to factor out UI. When you want a *named, parameterized* component with
its own fields, derive `IntoElement` and implement `RenderOnce`:

```rust
use gpui::{div, px, rgb, App, IntoElement, ParentElement, RenderOnce, SharedString, Styled, Window};

#[derive(IntoElement)]
struct Badge {
    label: SharedString,
    danger: bool,
}

impl Badge {
    fn new(label: impl Into<SharedString>) -> Self {
        Self { label: label.into(), danger: false }
    }
    fn danger(mut self) -> Self { self.danger = true; self }
}

impl RenderOnce for Badge {
    fn render(self, _window: &mut Window, _cx: &mut App) -> impl IntoElement {
        let bg = if self.danger { rgb(0xf38ba8) } else { rgb(0x89b4fa) };
        div()
            .px_2()
            .rounded_full()
            .bg(bg)
            .text_color(rgb(0x11111b))
            .child(self.label)
    }
}

// Usage, anywhere a child is expected:
//   div().child(Badge::new("draft")).child(Badge::new("error").danger())
```

`RenderOnce` is for **stateless** components consumed once per frame — the
`render(self, …)` takes `self` by value. This is the right tool for buttons,
chips, list rows, and any "given these props, draw this" widget. Components
that own *persistent* state are views (entities), covered next chapter.

> **`RenderOnce::render` takes `&mut App`, not `&mut Context<Self>`.** A
> stateless component has no entity, so there's no view-specific context — it
> gets the plain application context. Keep this distinction in mind; it
> determines what you can do inside (e.g. you can't `cx.notify()` a `RenderOnce`,
> because there's nothing to notify).

## Putting it together

```rust
fn toolbar(title: &str, dirty: bool) -> impl IntoElement {
    div()
        .flex().flex_row().items_center().gap_2()
        .w_full().px_3().py_2()
        .bg(rgb(0x181825))
        .border_b_1().border_color(rgb(0x313244))
        .child(div().font_weight(gpui::FontWeight::MEDIUM).child(title.to_string()))
        .when(dirty, |d| d.child(Badge::new("unsaved").danger()))
        .child(div().flex_1())                  // spacer pushes the rest right
        .child(Badge::new("ready"))
}
```

That's the whole styling story: build divs, chain styles, nest children, use
`when`/`when_some` for conditionals, and factor reusable bits into
`RenderOnce` components. Next we make these trees come alive with state —
[Chapter 3: Entities and views](03-entities-and-views.md).
