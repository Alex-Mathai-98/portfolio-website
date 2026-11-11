### 2. Lockset analysis: Eraser

**Paper:** Savage, Burrows, Nelson, Sobalvarro and Anderson, *Eraser: A Dynamic Data Race Detector for Multithreaded Programs*, SOSP 1997 / ACM TOCS 1997. [Read the full post]({% post_url 2026-10-30-eraser-lockset %}).

> Instead of asking whether two accesses happened in an order the program enforced, ask whether **one lock was held for every access** to the variable across threads.

#### A tiny server with two bugs

MiniServer is a toy web cache. A main thread sets a config value, two workers serve requests from a shared table, and a reporter thread prints statistics.

![MiniServer source code, with the two buggy lines highlighted](/assets/img/posts/eraser/01-miniserver-code.png)

| Variable | Intended protection | What happens | Verdict we want |
| --- | --- | --- | --- |
| `config_max` | none needed: written once before other threads exist, then only read | exactly that | **stay quiet** |
| `hits` | `stats_mu` | the reporter uses the wrong lock `log_mu` | **Bug 1**: wrong lock |
| `table` | `table_rw`, write mode must be used for writes | `evict_stale` writes under a read lock | **Bug 2**: wrong mode |

A good detector flags `hits` and `table` and says nothing about `config_max`. Eraser's design is, more or less, the story of getting all three right.


#### The core idea: intersect the locks

Eraser enforces a **locking discipline**: every shared variable is protected by some common lock, held on every access across threads. The program never says which lock protects which variable, so Eraser infers it from the run. For each variable `v` it keeps a **candidate set** C(v):

```text
locks_held(t)@l = locks t holds when it executes line l

C(v) := all locks                        // initially
on each access to v by thread t at line l:
    C(v) := C(v) ∩ locks_held(t)@l
    if C(v) = { } then warn
```

This is all done dynamically: the instrumented program updates `locks_held(t)` on every lock and unlock, and intersects on every load and store.

##### Dry run on MiniServer

**Assumed schedule:** main runs first, then W1 (a cache miss, so it inserts), then W2 (a cache hit), then R, each running to completion. Line numbers refer to the MiniServer code above.

![Dry run of the basic lockset algorithm on MiniServer](/assets/img/posts/eraser/04-dry-run-basic.png)

This naive version gets one of three right. It catches Bug 1, but it raises a false alarm on `config_max` (its first write held no lock) and misses Bug 2 (`table_rw` *is* held at `evict_stale`, just in the wrong mode \[read mode instead of write mode\]).

#### Quieting false alarms: a per-variable state machine

Two common patterns are harmless but violate the naive rule:

- Initializing data without a lock before anyone else can see it.
- Sharing data that is only read after initialization.

Eraser's two fixes:

- **For initialization:** Eraser cannot tell when initialization ends, so it uses a proxy. A variable counts as initialized when a **second thread** first touches it.
- **For read-only sharing:** Eraser reports only once a variable is **written** after being shared.

##### State Machine for the above two fixes

![Eraser's per-variable state machine](/assets/img/posts/eraser/05-state-machine.png)

One detail that trips people up: allocation is not an access. Allocating memory just marks it Virgin. The first real read or write (initialization is treated as a write), by any thread, makes that thread the owner, and the owner's accesses never enter C(v).

##### Dry run with the state machine

**Assumed schedule:** the same as in the first dry run: main, then W1 (cache miss), then W2 (cache hit), then R.

![Dry run with the state machine](/assets/img/posts/eraser/06-dry-run-state-machine.png)

With this, `config_max` goes Exclusive (owned by main), then Shared when a worker reads it. C(v) is empty, but Shared never reports, so the false alarm is gone. Bug 1 is still caught.

##### The price: a schedule-dependent blind spot

###### The modification

For this subsection only, change `main` so that it starts the threads **before** setting `config_max`:

![The modified main(): config_max is set after the threads start](/assets/img/posts/eraser/07-modification.png)

###### What Eraser sees under two schedules

The price is a new blind spot that depends on the schedule. If main started the workers *before* setting `config_max`, a worker might read 0. Eraser catches that only if the worker's read lands first:

![Sharing data before it is initialized, under two schedules](/assets/img/posts/eraser/08-early-sharing.png)

The authors admit this makes Eraser more schedule-dependent than they would like, but doubt it mattered in practice.

#### Reader-writer locks: finding the last bug

A reader-writer lock held in read mode lets other readers in at the same time, so it protects a read but not a write. Eraser's last refinement changes one line: **reads intersect with locks held in any mode, writes intersect only with locks held in write mode.** An ordinary mutex is always exclusive, so it counts in both sets, and Bug 1 behaves as before.

```text
locks_held(t)@l       = locks t holds in ANY mode when it executes line l
write_locks_held(t)@l = locks t holds in WRITE mode when it executes line l

on each access op(v) by thread t at line l:
    if op(v) = read then
        C(v) := C(v) ∩ locks_held(t)@l
    else if op(v) = write then
        C(v) := C(v) ∩ write_locks_held(t)@l     // the one changed line
    if C(v) = { } then warn

e.g. line 26: locks_held(R)@26 = {table_rw}, but write_locks_held(R)@26 = { }
```

Here is the whole run under the final algorithm, with the held-lock sets shown at every access:

![The whole run under the full algorithm, with held-lock sets at every access](/assets/img/posts/eraser/09-full-algorithm-run.png)

At `evict_stale`, the reporter holds `table_rw` only in read mode, so its write-mode set is empty and C(table) empties: Bug 2 is caught. The scorecard is now three for three: quiet on `config_max`, warnings on `hits` and `table`.
