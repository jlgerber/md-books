# Chapter 10 — Async and background work

A UI thread that blocks is a UI that freezes. GPUI gives you `spawn`,
executors, timers, and `Task`s to do work off the critical path and then fold
results back into your views. This chapter also shows the very common pattern
of bridging a background OS thread (e.g. a socket/db worker) into GPUI.

## `cx.spawn`: an async task tied to a view

`cx.spawn` starts an async task. Its closure receives a **weak** handle to your
view and an async context:

```rust
use gpui::Context;

impl MyView {
    fn load(&mut self, cx: &mut Context<Self>) {
        cx.spawn(async move |this, cx| {
            // `this`: WeakEntity<MyView>   `cx`: &mut AsyncApp
            let data = fetch_data().await;            // your async work

            // Re-enter the view to apply the result. `update` returns Err if
            // the view was dropped while we were awaiting — hence WeakEntity.
            let _ = this.update(cx, |this, cx| {
                this.data = data;
                cx.notify();
            });
        })
        .detach();          // let it run independently; or keep the Task to cancel
    }
}
```

Key points:

- The handle is **weak** so a long task can't keep a closed view alive. After
  the `await`, `this.update(cx, …)` returns `Err` if it's gone — ignore it.
- `cx.spawn` returns a **`Task<T>`**. Dropping the `Task` cancels the work. To
  fire-and-forget, call `.detach()`. To be able to cancel, store the `Task`.
- You can't hold `&mut App`/`&mut Context` across `.await`; that's why you
  re-acquire via `this.update`.

## Timers and debouncing

For delays, use the background executor's timer:

```rust
use std::time::Duration;

let timer = cx.background_executor().timer(Duration::from_millis(300));
cx.spawn(async move |this, cx| {
    timer.await;
    let _ = this.update(cx, |this, cx| { /* … */ cx.notify(); });
}).detach();
```

A real-world **debounce**: coalesce rapid events (e.g. arrowing through a list
that triggers a fetch per row) so only the last one fires. Use a generation
counter:

```rust
struct Detail {
    fetch_gen: u64,
    // …
}

impl Detail {
    fn selection_changed(&mut self, cx: &mut Context<Self>) {
        self.fetch_gen = self.fetch_gen.wrapping_add(1);
        let generation = self.fetch_gen;
        let timer = cx.background_executor().timer(Duration::from_millis(300));
        cx.spawn(async move |this, cx| {
            timer.await;
            let _ = this.update(cx, |this, cx| {
                // Only fire if no newer selection happened during the wait:
                if this.fetch_gen == generation {
                    this.fetch_now(cx);
                }
            });
        })
        .detach();
    }
}
```

Each change bumps `fetch_gen` and schedules a fetch 300ms out; only the wait
whose captured `generation` still matches actually fetches.

## Executors

- `cx.background_executor()` — a thread pool for CPU/IO work and timers. Work
  here is **not** on the UI thread.
- `cx.foreground_executor()` — runs on the UI thread; use for tiny continuations
  that must touch the UI without an `update` hop.

`cx.background_executor().spawn(async { … })` runs a future on the pool and
returns a `Task` you can `.await` from another task. Use it to parallelize
heavy work, then `update` the view with results.

## Bridging an OS thread (worker) into GPUI

GPUI's async is cooperative; truly blocking work (a synchronous socket, a
blocking DB driver, FFI) belongs on its own `std::thread`. The clean pattern:
**channels in, channels out, and a pump task that forwards results into GPUI.**

```rust
use std::sync::mpsc;

// 1. Spawn the worker thread. It owns its blocking runtime/driver and
//    communicates only via channels.
fn spawn_worker(cmd_rx: mpsc::Receiver<Cmd>, on_event: impl Fn(WorkerEvent) + Send + 'static) {
    std::thread::spawn(move || {
        for cmd in cmd_rx {                       // blocks on the worker thread, fine
            let result = run_blocking(cmd);       // synchronous work
            on_event(result);
        }
    });
}

// 2. At startup: create channels, start the worker, and pump its events
//    into GPUI on the foreground side.
fn wire_up(state: gpui::Entity<AppState>, cx: &mut gpui::App) {
    let (cmd_tx, cmd_rx) = mpsc::channel::<Cmd>();
    let (evt_tx, evt_rx) = mpsc::channel::<WorkerEvent>();

    spawn_worker(cmd_rx, move |ev| { let _ = evt_tx.send(ev); });

    // The pump: an async task that drains evt_rx and applies events to state.
    cx.spawn(async move |cx| {
        while let Ok(ev) = evt_rx.recv() {       // (use an async channel for true non-blocking)
            let _ = state.update(cx, |s, cx| { s.apply(ev); cx.notify(); });
        }
    }).detach();

    // Store cmd_tx on your state so views can send commands:
    // state.update(cx, |s, _| s.cmd_tx = cmd_tx);
    let _ = cmd_tx;
}
```

In a production app you'd use an async channel (e.g. `tokio::sync::mpsc` or
`smol`/`futures` channels) so the pump's `recv().await` doesn't block the
executor, and the worker thread can host its own runtime. The shape is the
same: **views send `Cmd`s down a channel; the worker does blocking work; a
pump task lifts `Event`s back into GPUI via `entity.update + notify`.** This is
exactly how the asset browser talks to its database daemon over a Unix socket.

### Correlating async results to current state

Async results can arrive *after* the user moved on. Tag requests with the
context they were for and **drop stale results**:

```rust
// When a fetch result arrives, ignore it unless it matches what's selected now:
fn on_result(&mut self, for_key: &str, rows: Vec<Row>, cx: &mut Context<Self>) {
    if self.current_key.as_deref() == Some(for_key) {
        self.rows = rows;
        cx.notify();
    }
    // else: a result for a row we've since left — discard it.
}
```

Combine this with the debounce above and rapid navigation stays correct and
cheap.

## Cheatsheet

| Need | Use |
|------|-----|
| Async work that updates a view | `cx.spawn(async move |this, cx| { … this.update(cx, …) })` |
| A delay | `cx.background_executor().timer(dur).await` |
| Heavy CPU/IO off the UI thread | `cx.background_executor().spawn(fut)` |
| Cancelable work | keep the returned `Task`; drop it to cancel |
| Fire-and-forget | `.detach()` |
| Blocking driver / FFI | dedicated `std::thread` + channels + a pump task |
| Avoid stale UI | weak handle (`update` returns `Err` if gone) + result correlation |

Next: app-wide singletons with [global state](11-global-state.md).
