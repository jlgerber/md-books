# Chapter 14 — Theming and icons

This chapter covers two things that make an app look finished: the
`gpui-component` **theme** (a palette of semantic colors that every widget
reads) and **icons**.

## Two layers of styling

Recall from [Chapter 2](02-elements-and-styling.md) that core GPUI styling is
literal: `.bg(rgb(0x…))`, `.text_color(…)`. That's fine for one-off elements,
but if you hard-code hex everywhere, you can't theme the app. The fix is to
pull colors from a **theme** object instead of literals.

`gpui-component` provides a global `Theme` with ~100 semantic color "slots"
(background, foreground, primary, secondary, accent, border, danger, …) plus a
corner `radius`. Every built-in widget reads these slots, and your own views
should too.

## Reading the theme: `cx.theme()`

The `ActiveTheme` trait adds `.theme()` to contexts. Bring it into scope and
read slots while building elements:

```rust
use gpui_component::ActiveTheme as _;

impl Render for Panel {
    fn render(&mut self, _w: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let theme = cx.theme().clone();   // cheap; clone so the borrow doesn't linger
        gpui::div()
            .bg(theme.background)
            .text_color(theme.foreground)
            .border_1().border_color(theme.border)
            .rounded(theme.radius)
            .child(
                gpui::div()
                    .text_color(theme.muted_foreground)   // de-emphasized text
                    .child("subtitle"),
            )
    }
}
```

Cloning `theme` up front is the common idiom — `cx.theme()` borrows `cx`, and
you'll want `cx` free for `cx.listener` etc. while building the tree. `Theme`
is cheap to clone (it's a bag of `Hsla`s and a few scalars).

### The slots you'll use most

| Slot | Use |
|------|-----|
| `background` / `foreground` | base surface + text |
| `secondary` / `secondary_hover` / `secondary_active` | panels, subtle surfaces, hover/press |
| `primary` / `primary_hover` / `primary_active` | accent / selection |
| `button_primary*` | **what `.primary()` buttons actually read** — distinct from `primary` |
| `accent` / `accent_foreground` | highlights, selected rows |
| `muted_foreground` | dimmed/secondary text |
| `border` / `ring` | separators / focus ring |
| `danger` / `warning` / `success` / `info` | status colors |
| `radius` | global corner radius |

> **Gotcha:** `.primary()` buttons (and other primary widgets) read the
> `button_primary*` slots, **not** `primary`. If your theme sets `primary`
> but not `button_primary`, primary buttons fall back to the base theme's
> near-white default and look wrong. Set both. (This exact bug bit the asset
> browser: themed primary buttons rendered white until `button_primary*` was
> mapped.)

### Deriving shades

For hover/press states, derive from a base slot by nudging lightness rather
than hard-coding another hex — it tracks the theme automatically:

```rust
fn raised(theme: &gpui_component::Theme) -> gpui::Hsla {
    let mut c = theme.secondary;        // a panel surface
    c.l = (c.l + 0.04).clamp(0.0, 1.0); // a touch lighter → reads as "raised"
    c
}
// card background that doesn't look "selected":
gpui::div().bg(raised(cx.theme()));
```

The browser uses this for task/note cards (a lifted `secondary`) and for the
create button's hover/active (accent ±6% lightness), so the whole UI shifts
correctly when the theme changes.

## Switching and customizing the theme

Switch between the built-in light/dark base:

```rust
use gpui_component::{Theme, ThemeMode};

Theme::change(ThemeMode::Dark, None, cx);   // or ThemeMode::Light
```

Override specific slots by mutating the global theme (e.g. to apply your own
brand palette). Then push the change to open windows so they repaint
immediately:

```rust
use gpui_component::{Theme, ThemeMode};

fn apply_brand(cx: &mut gpui::App) {
    Theme::change(ThemeMode::Dark, None, cx);   // start from a base
    let theme = Theme::global_mut(cx);          // &mut Theme

    theme.background = hsla_from_hex("#1e1e2e");
    theme.foreground = hsla_from_hex("#cdd6f4");
    theme.primary    = hsla_from_hex("#89b4fa");
    // Don't forget the button slots that .primary() actually reads:
    theme.button_primary            = theme.primary;
    theme.button_primary_foreground = hsla_from_hex("#11111b");
    // … set the rest of the slots you care about; unset ones inherit the base …

    // Repaint every open window with the new palette right now:
    cx.refresh_windows();
}
```

`Theme::global_mut(cx)` is the escape hatch for full control — set as many or
as few slots as you want; anything you don't touch keeps the base value. After
mutating, **`cx.refresh_windows()`** is what makes the change visible
immediately (otherwise the new colors only appear on the next natural render).
This "map my tokens onto `gpui-component`'s slots, then refresh" pattern is how
the asset browser applies its `*.theme` files.

### A token-bridge pattern

If your app has its own palette format (named tokens loaded from a file), write
one function that maps your tokens onto the `Theme` slots:

```rust
struct Palette { bg: String, text: String, accent: String, /* … */ }

fn apply(p: &Palette, cx: &mut gpui::App) {
    use gpui_component::{Theme, ThemeMode};
    Theme::change(ThemeMode::Dark, None, cx);
    let t = Theme::global_mut(cx);
    t.background = hsla_from_hex(&p.bg);
    t.foreground = hsla_from_hex(&p.text);
    t.primary = hsla_from_hex(&p.accent);
    t.button_primary = t.primary;
    // … one assignment per slot you map …
    cx.refresh_windows();
}
```

Keep this function the single source of truth for "palette → theme," call it at
startup and whenever the user picks a new palette.

## Icons

`gpui-component` bundles a large icon set (the Lucide icons) as the `IconName`
enum, rendered via the `Icon` element.

```rust
use gpui_component::{Icon, IconName, Sizable as _, Size};
use gpui_component::ActiveTheme as _;

// Basic icon:
Icon::new(IconName::Search);

// Sized + colored (icons take text color):
Icon::new(IconName::Plus)
    .with_size(Size::Small)               // or .small() via Sizable
    .text_color(cx.theme().muted_foreground);

// Inside a button:
use gpui_component::button::{Button, ButtonVariants as _};
Button::new("refresh").icon(IconName::RefreshCw).ghost();
```

Icons inherit their color from `text_color` (set it on the icon or a parent),
and size via `Sizable` (`Size::XSmall/Small/Medium/Large`, or `with_size(px(…))`).

Common names you'll use: `Search`, `Plus`, `Close` / `CircleX`, `Check` /
`CircleCheck`, `ChevronDown` / `ChevronRight` / `ChevronUp` / `ChevronLeft`,
`Menu`, `ArrowLeft` / `ArrowRight`, `Star`, `Info`, `TriangleAlert`, `Loader`
(used by `Spinner`), `PanelLeftOpen` / `PanelLeftClose`, `RefreshCw`. There are
hundreds — your editor's autocomplete on `IconName::` is the fastest way to
browse. (Browse `crates/ui/src/icon.rs` for the full list.)

### Icon buttons

A round/square icon-only button is just a `Button` with an icon and no label,
or a styled `div` with an `Icon` child when you want full control over shape:

```rust
// As a Button:
Button::new("close").icon(IconName::Close).ghost().small();

// As a custom round button (full control over size/shape/colors):
use gpui::{div, px, InteractiveElement as _, StatefulInteractiveElement as _, Styled as _, ParentElement as _};
div()
    .id("add").size(px(22.)).rounded_full()
    .flex().items_center().justify_center()
    .bg(cx.theme().button_primary)
    .text_color(cx.theme().button_primary_foreground)
    .hover(|s| s.bg(cx.theme().button_primary_hover))
    .cursor_pointer()
    .child(Icon::new(IconName::Plus).with_size(Size::Small));
```

## Rules of thumb

- **Never hard-code hex in views** beyond throwaway prototypes — read
  `cx.theme()` slots so theming and dark/light just work.
- **Clone the theme** at the top of `render` to keep `cx` free.
- **`.primary()` widgets read `button_primary*`**, not `primary` — set both.
- **Derive hover/press shades** from base slots (lightness nudges) so they
  track the theme.
- **`cx.refresh_windows()`** after changing the global theme to repaint now.
- **Icons take `text_color` and `Sizable`** like text; `Button::icon(…)` is the
  quick path for icon buttons.

That completes the tour. Back to the [table of contents](README.md).
