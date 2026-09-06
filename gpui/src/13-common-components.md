# Chapter 13 — Common components (`gpui-component`)

Core GPUI gives you primitives — `div`, styling, events, focus. It does **not**
ship buttons, inputs, dropdowns, switches, or sliders. Those come from the
companion library [`gpui-component`](https://github.com/longbridge/gpui-component),
a batteries-included widget set built on the primitives from the previous
chapters. This chapter is a practical tour of the ones you'll reach for daily.

```toml
# Pin the rev (see Chapter 1) — this book is verified against this one:
gpui-component = { git = "https://github.com/longbridge/gpui-component", rev = "6b8d9773392a627eed4e9b93d3b840d43e7fdf1f" }
```

Call `gpui_component::init(cx)` once at startup, and wrap your window's root in
`gpui_component::Root` (see Chapter 9) so themed widgets, overlays, and
notifications work.

## The one distinction that explains everything: stateless vs stateful

`gpui-component` widgets come in two shapes, and knowing which is which tells
you how to use any of them:

- **Stateless widgets** — `Button`, `Checkbox`, `Switch`, `Radio`. *You* own
  the value (a `bool` in your view). The widget renders from a value you pass
  and calls a handler with the *new* value; you store it and `notify`. Built
  fresh each render.

- **Stateful widgets** — `Input`, `Select`, `Combobox`, `Slider`. These own
  internal state (text + caret, selection, drag position) too complex to
  rebuild each frame, so they're backed by an **entity** you create once
  (`cx.new(|cx| XState::new(...))`) and store. You render a lightweight handle
  (`Input::new(&state)`) that points at it.

Two shared traits cut across all of them:

- **`ButtonVariants`** (buttons) — `.primary()`, `.secondary()`, `.danger()`,
  `.warning()`, `.success()`, `.ghost()`, `.link()`, `.outline()`.
- **`Sizable`** — `.xsmall()`, `.small()`, `.large()`, or `.with_size(Size::…)`.
- **`Disableable`** — `.disabled(bool)`.

Bring the trait into scope to use its methods (e.g.
`use gpui_component::button::ButtonVariants as _;`).

## Button

```rust
use gpui_component::button::{Button, ButtonVariants as _};
use gpui_component::{Sizable as _, IconName};

Button::new("save")                 // needs a stable id (it's clickable)
    .label("Save")
    .primary()                      // variant
    .small()                        // size
    .disabled(!self.can_save)
    .on_click(cx.listener(|this, _ev, _window, cx| {
        this.save(cx);
    }));

// Icon button / icon + label:
Button::new("add").icon(IconName::Plus).label("New").primary();
Button::new("close").icon(IconName::Close).ghost();   // icon-only
```

`on_click` works with `cx.listener` so you get `&mut Self` back. For a button
that should trigger an action (so keyboard and mouse share a path), dispatch
it: `.on_click(cx.listener(|_,_,w,cx| w.dispatch_action(Box::new(Save), cx)))`.

## Text input

Stateful — back it with an `InputState` entity (see also Chapter 8):

```rust
use gpui_component::input::{Input, InputState, InputEvent};

// In your view's constructor (needs &mut Window):
let name = cx.new(|cx| InputState::new(window, cx).placeholder("Name"));
let body = cx.new(|cx| {
    InputState::new(window, cx).multi_line(true).auto_grow(3, 8).placeholder("Notes…")
});

// Render a handle that points at the state:
Input::new(&self.name);                // single line
Input::new(&self.body);                // multi-line, grows 3–8 rows

// Read / write:
let text = self.name.read(cx).value().to_string();
self.name.update(cx, |st, cx| st.set_value("", window, cx));   // clear

// React live (e.g. enable a button as the user types):
let sub = cx.subscribe(&self.name, |_this, _e, ev: &InputEvent, cx| {
    if matches!(ev, InputEvent::Change) { cx.notify(); }
});
```

Useful builders: `.prefix(el)` / `.suffix(el)` (icons inside the field),
`.bordered(false)`, `.rounded_full()`, `.cleanable(true)`. `gpui-component`
also has `NumberInput` and `OtpInput` variants.

## Select (dropdown / picker)

A `Select` is a non-editable dropdown. Stateful — backed by `SelectState`. The
state is generic over a *delegate* that supplies items; the simplest delegate
is a list of `SharedString`.

```rust
use gpui_component::select::{Select, SelectState};
use gpui::{IndexPath, SharedString};

let items: Vec<SharedString> = vec!["Low".into(), "Medium".into(), "High".into()];
let priority = cx.new(|cx| {
    SelectState::new(items, Some(IndexPath::default().row(1)) /* default = Medium */, window, cx)
});

// Render:
Select::new(&self.priority).placeholder("Priority…").cleanable(false);

// Read the choice:
let value = self.priority.read(cx).selected_value().map(|s| s.to_string());
// or by index: self.priority.read(cx).selected_index(cx)

// Replace the items at runtime (e.g. after fetching options):
self.priority.update(cx, |st, cx| st.set_items(new_items, window, cx));
self.priority.update(cx, |st, cx| st.set_selected_index(None, window, cx));
```

### Making a Select searchable — and a real gotcha

`SelectState` has `.searchable(true)`, which shows a filter box. **But the
plain `Vec<SharedString>` delegate doesn't actually filter** — typing shows the
box yet results don't narrow. To get working search, back the state with
`SearchableVec<SharedString>` (which implements the filtering), and read the
selection via `selected_value()` (the selected *index* is relative to the
*filtered* view once a search is active, so don't index into a parallel list):

```rust
use gpui_component::searchable_list::SearchableVec;

let sel = cx.new(|cx| {
    SelectState::new(SearchableVec::new(Vec::<SharedString>::new()), None, window, cx)
        .searchable(true)
});
// later, when options arrive:
sel.update(cx, |st, cx| st.set_items(SearchableVec::new(items), window, cx));
// read:
let chosen = sel.read(cx).selected_value().map(|s| s.to_string());
```

This is the exact pattern (and pitfall) behind the asset browser's foreign-key
comboboxes.

## Combobox

A `Combobox` is a Select with a free-text entry that filters as you type (the
classic editable combobox). Backed by `ComboboxState`:

```rust
use gpui_component::combobox::{Combobox, ComboboxState};

let cb = cx.new(|cx| ComboboxState::new(items_delegate, vec![/*selected*/], window, cx));
Combobox::new(&self.cb).placeholder("Search…");
```

Reach for `Combobox` when the option set is large and the user benefits from
typing to filter *and* possibly entering a custom value; reach for `Select`
when it's a fixed, smallish list.

## Checkbox

Stateless — you own the `bool`:

```rust
use gpui_component::checkbox::Checkbox;

Checkbox::new("notify")
    .label("Email me")
    .checked(self.notify)
    .on_click(cx.listener(|this, checked: &bool, _window, cx| {
        this.notify = *checked;
        cx.notify();
    }));
```

The handler receives the *new* checked value (`&bool`). Note this isn't a
`cx.listener`-only API — `Checkbox::on_click` is
`Fn(&bool, &mut Window, &mut App)`; wrapping it in `cx.listener` (as above)
gives you `&mut Self`, which is what you usually want.

## Switch (toggle)

A switch is a styled boolean toggle — same shape as Checkbox:

```rust
use gpui_component::switch::Switch;

Switch::new("dark-mode")
    .label("Dark mode")
    .checked(self.dark)
    .on_click(cx.listener(|this, checked: &bool, _window, cx| {
        this.dark = *checked;
        cx.notify();
    }));
```

(`.color(hsla)` overrides the "on" color; it defaults to `theme.primary`.)

## Radio and RadioGroup

A single `Radio` is like a checkbox; a `RadioGroup` manages mutually-exclusive
choices and reports the selected index:

```rust
use gpui_component::radio::RadioGroup;

RadioGroup::new("size")
    .children(["Small", "Medium", "Large"])
    .selected_index(Some(self.size_ix))
    .on_click(cx.listener(|this, ix: &usize, _window, cx| {
        this.size_ix = *ix;
        cx.notify();
    }));
```

`RadioGroup` lays the radios out (horizontal by default) and gives you the
chosen index in the handler.

## Slider

Stateful — backed by a `SliderState` entity that emits `SliderEvent`:

```rust
use gpui_component::slider::{Slider, SliderState, SliderEvent};

let volume = cx.new(|_cx| {
    SliderState::new().min(0.).max(100.).step(1.).default_value(50.)
});
// React to changes:
let _sub = cx.subscribe(&volume, |this, _state, ev: &SliderEvent, cx| {
    match ev {
        SliderEvent::Change(v)  => { this.volume = *v; cx.notify(); }  // live drag
        SliderEvent::Release(v) => { /* commit on release */ let _ = v; }
    }
});

// Render:
Slider::new(&self.volume);
// Read current value any time:
let v = self.volume.read(cx).value();
```

`SliderEvent::Change` fires continuously during a drag; `Release` fires once
when the user lets go — handy if `Change` is expensive (e.g. triggers a query).

## Badges, tags, labels, tooltips, spinners

Smaller bits you'll sprinkle around:

```rust
use gpui_component::{Sizable as _, IconName, Icon};
use gpui_component::badge::Badge;        // a small count/dot overlay
use gpui_component::tag::Tag;            // a labeled pill ("status" chips)
use gpui_component::label::Label;        // a styled text label
use gpui_component::spinner::Spinner;    // animated loading indicator
use gpui_component::tooltip::Tooltip;

Tag::new().child("draft");               // a pill
Label::new("Field name");                // styled text
Spinner::new().with_size(Size::Small);   // spinning loader

// Tooltip on any interactive element (the closure builds it lazily):
div().id("info").tooltip(|window, cx| Tooltip::new("Details…").build(window, cx));
```

## A small settings form

```rust
use gpui::{div, Context, IntoElement, ParentElement, Render, Styled, Window};
use gpui_component::{
    Sizable as _,
    button::{Button, ButtonVariants as _},
    checkbox::Checkbox,
    switch::Switch,
    input::{Input, InputState},
};

struct Settings {
    display_name: gpui::Entity<InputState>,
    dark: bool,
    notify: bool,
}

impl Render for Settings {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .flex().flex_col().gap_3().p_4()
            .child(Input::new(&self.display_name))
            .child(Switch::new("dark").label("Dark mode").checked(self.dark)
                .on_click(cx.listener(|this, v: &bool, _w, cx| { this.dark = *v; cx.notify(); })))
            .child(Checkbox::new("notify").label("Notifications").checked(self.notify)
                .on_click(cx.listener(|this, v: &bool, _w, cx| { this.notify = *v; cx.notify(); })))
            .child(
                div().flex().flex_row().justify_end().gap_2()
                    .child(Button::new("cancel").label("Cancel").ghost())
                    .child(Button::new("save").label("Save").primary().small()),
            )
    }
}
```

## Component picker cheat-sheet

| Need | Component | Stateful? |
|------|-----------|-----------|
| Action / command | `Button` (`ButtonVariants`, `Sizable`) | no |
| Free text | `Input` / `InputState` | **yes** |
| Number / OTP | `NumberInput` / `OtpInput` | **yes** |
| Pick one from a fixed list | `Select` / `SelectState` | **yes** |
| Pick one, type to filter (large/open) | `Combobox` / `ComboboxState` | **yes** |
| On/off (form-y) | `Checkbox` | no |
| On/off (settings toggle) | `Switch` | no |
| One of N options, all visible | `RadioGroup` | no |
| A numeric range | `Slider` / `SliderState` | **yes** |
| Status pill / count | `Tag` / `Badge` | no |
| Loading | `Spinner` | no |
| Hover help | `.tooltip(…)` (`Tooltip`) | no |

There's much more in `gpui-component` — tables (`Table`), a dock/panel system,
date/time pickers, charts, a sidebar, resizable splits, steppers, lists. When
you need a standard widget, browse its `crates/ui/src/` before building from
scratch.

Next: making it all look right — [theming and icons](14-theming-and-icons.md).
