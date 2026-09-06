# Chapter 6 — State, `notify`, events, and observation

Local state plus `cx.notify()` (Chapters 3 & 5) handles a single view. Real
apps have *many* views that must react to *each other*. GPUI gives you three
mechanisms, in increasing decoupling:

1. **`cx.notify()`** — "I changed; re-render me (and anyone observing me)."
2. **`cx.observe`** — "re-run my callback whenever that entity notifies."
3. **`EventEmitter` + `cx.subscribe`** — "that entity emits typed events;
   handle them."

## `notify` and the shared-state pattern

When multiple views share one `Entity<AppState>` (Chapter 3), a mutation in
one place must repaint the others. The mechanism is observation: a view that
**reads** another entity during `render` is automatically re-rendered when
that entity notifies — *if* it has registered to observe it. The cleanest way
to wire this is to subscribe/observe at construction.

But the simplest case needs nothing extra: a view that holds
`Entity<AppState>` and mutates it via `update` will trigger that entity's
observers. To make *this* view one of those observers, use `cx.observe`.

## `cx.observe`: react to another entity changing

```rust
struct Footer {
    state: Entity<AppState>,
}

impl Footer {
    fn new(state: Entity<AppState>, cx: &mut Context<Self>) -> Self {
        // Re-render this Footer whenever AppState notifies.
        cx.observe(&state, |_this, _observed_state, cx| {
            cx.notify();
        })
        .detach();                     // keep the subscription alive for the view's life
        Self { state }
    }
}
```

`cx.observe(&entity, handler)` returns a `Subscription`. **If you drop the
`Subscription`, the observation stops.** Two idioms:

- `.detach()` — fire-and-forget; lives until the *observing* view is dropped.
- Store it in a `Vec<Subscription>` field on the view (commonly named
  `_subs`) — same lifetime, but explicit and lets you hold several.

```rust
struct DetailView {
    state: Entity<AppState>,
    _subs: Vec<Subscription>,
}
impl DetailView {
    fn new(state: Entity<AppState>, cx: &mut Context<Self>) -> Self {
        let sub = cx.observe(&state, |_this, _state, cx| cx.notify());
        Self { state, _subs: vec![sub] }
    }
}
```

## `EventEmitter` + `cx.subscribe`: typed messages

`observe` only says "something changed." Often you want to say *what* happened
— "the user requested a refresh," "a row was activated," "a create finished."
That's an **event**: a typed value an entity emits, and other entities
subscribe to.

Step 1 — declare that your entity emits an event type:

```rust
use gpui::EventEmitter;

#[derive(Clone, Debug)]
enum AppEvent {
    FocusQuery,
    CreateFinished { ok: bool, message: String },
}

impl EventEmitter<AppEvent> for AppState {}   // marker impl
```

Step 2 — emit from anywhere you have a `Context<AppState>`:

```rust
impl AppState {
    fn request_focus(&mut self, cx: &mut Context<Self>) {
        cx.emit(AppEvent::FocusQuery);          // delivered to subscribers
    }
}
```

Step 3 — subscribe in another view:

```rust
struct Toolbar { state: Entity<AppState> }

impl Toolbar {
    fn new(state: Entity<AppState>, window: &mut Window, cx: &mut Context<Self>) -> Self {
        // subscribe_in: handler also gets a &mut Window (here we move focus).
        let _sub = cx.subscribe_in(
            &state,
            window,
            |this: &mut Self, _emitter, event: &AppEvent, window, cx| {
                if let AppEvent::FocusQuery = event {
                    // e.g. window.focus(&this.query_focus, cx);
                    cx.notify();
                }
            },
        );
        Self { state }   // (store _sub in a field to keep it alive!)
    }
}
```

The handler signature for `subscribe` is
`Fn(&mut Self, Entity<Emitter>, &Event, &mut Context<Self>)`; for
`subscribe_in` it inserts a `&mut Window` before the context. As with
observe, **keep the returned `Subscription`** (field or `.detach()`).

### Events vs actions

Events flow from a *specific emitter entity* to its *subscribers*, regardless
of focus or screen position. They're for app-internal, model-to-view (or
view-to-view) messaging. **Actions** ([Chapter 7](07-actions-and-keybindings.md))
flow up the *focus tree* and are how keyboard shortcuts and menus dispatch.
Rule of thumb: keystroke/menu → action; "this model changed / this thing
happened" → event.

## A complete example: toolbar drives a results view via shared state + events

```rust
use gpui::{Context, Entity, EventEmitter, Window};

#[derive(Clone)]
enum AppEvent { QueryChanged(String) }

#[derive(Default)]
struct AppState { query: String, results: Vec<String> }
impl EventEmitter<AppEvent> for AppState {}

impl AppState {
    fn set_query(&mut self, q: String, cx: &mut Context<Self>) {
        self.query = q.clone();
        cx.emit(AppEvent::QueryChanged(q));   // tell listeners
        cx.notify();                          // and repaint anything reading us
    }
}

struct Results {
    state: Entity<AppState>,
    _subs: Vec<gpui::Subscription>,
}

impl Results {
    fn new(state: Entity<AppState>, cx: &mut Context<Self>) -> Self {
        let sub = cx.subscribe(&state, |this: &mut Self, state, event: &AppEvent, cx| {
            if let AppEvent::QueryChanged(q) = event {
                // run the search, then store results back on the model:
                let hits = vec![format!("match for {q}")];
                state.update(cx, |s, cx| { s.results = hits; cx.notify(); });
            }
        });
        Self { state, _subs: vec![sub] }
    }
}
```

Here the toolbar (not shown) calls `state.update(cx, |s, cx| s.set_query(…))`;
`Results` hears the `QueryChanged` event, computes hits, writes them back to
the shared state, and notifies. Every view reading `state.results` repaints.

## Choosing the right tool

| You want… | Use |
|-----------|-----|
| To repaint *this* view after mutating *its own* fields | `cx.notify()` |
| To repaint when *another* entity changes (don't care why) | `cx.observe` |
| To react to *specific, typed* things happening | `EventEmitter` + `cx.subscribe(_in)` |

Always remember: **a dropped `Subscription` is a dead subscription.** Detach
it or store it.

Next, the keyboard story: [actions and key bindings](07-actions-and-keybindings.md).
