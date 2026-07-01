# Effects, Failure & Concurrency

The three tightly-coupled systems that trip up imperative habits the most. **Read this before writing any function signature.**

---

## 1. The effect system

Effects describe what a function may do beyond returning a value. They are written `<specifier>` between the parameter list and the return colon, and they **propagate up the call chain** unless a construct hides them.

### Specifiers you write

| Specifier | Meaning | Category |
|---|---|---|
| `<computes>` | Pure: no heap read/write/alloc, cannot fail | exclusive (heap) |
| `<reads>` | Read-only heap access | exclusive (heap) |
| `<transacts>` | Read + write + allocate heap, **rollback-capable** | exclusive (heap) |
| `<decides>` | **Failable** — "0 or 1 values" | additive (cardinality) |
| `<suspends>` | Async — can yield/await | additive (suspension) |
| `<converges>` | Guaranteed termination — **native-only** (can't apply to your own code) | — |
| `<writes>` / `<allocates>` | Separately-writable heap sub-effects sitting between `<reads>` and `<transacts>` | exclusive (heap) |

Heap effect lattice: `<computes>` <: `<reads>` <: `<transacts>`. Fewer effects = subtype (a `<computes>` function is usable where `<transacts>` is expected).

- **Specifier order is irrelevant** and each affects only its own family; **redundant specifiers produce a warning, not an error**.
- **`<varies>` is deprecated** — remove it if you see it in legacy code.
- **`<no_rollback>` cannot be written** as a specifier — it's only the current *default*.
- `[MaxVerse]` also has prediction effects (`<dictates>` server-authoritative default vs `<predicts>` client-speculative) and a cardinality lattice (`<fails>`/`<succeeds>`/`<ambiguates>`/`<iterates>`/`<abstracts>`/`<contradicts>`) — theoretical, not shippable.

### ⚠️ Current-build default is `<no_rollback>` — the pairing rules

**Right now the default effect set is `<no_rollback>` with no heap effects.** You must opt into heap access with `<transacts>`/`<reads>`, and opt into failure with `<decides>`. `<no_rollback>` code **cannot be called from a context that requires transactional rollback** (i.e. from `<decides>`/`<transacts>` bodies, `if`-conditions, `for`-filters).

Therefore, two rules you must follow on *every* function:

1. **A `<decides>` function MUST also carry `<computes>` or `<transacts>`.** A bare `<decides>` keeps the default `<no_rollback>` and is uncallable from any rollback/failure context.
2. **Any helper called from a `<decides>`/`<transacts>` body MUST also be `<computes>` or `<transacts>`** — even if it never fails and does no I/O. A plain `Helper(...):T = ...` carries `<no_rollback>` and breaks its caller.

```verse
# WRONG — bare <decides>, and a no_rollback helper called from a transacts body
Check(X:int)<decides>:void = X > 0
SkipWS():void = loop { ... }
Parse()<decides><transacts>:int = SkipWS(); ...        # both fail to compile

# RIGHT
Check(X:int)<decides><computes>:void = X > 0
SkipWS()<transacts>:void = loop { ... }
Parse()<decides><transacts>:int = SkipWS(); ...
```

Pick the heap partner by what the body does: pure → `<computes>`; reads state → `<reads>` (or `<transacts>`); mutates → `<transacts>`.

*(This is a temporary state of the toolchain; the default will become `<transacts>` later. Until then, write the heap effect explicitly — every time.)*

> **Reconciling sources:** the published *Verse book* says the default is `<transacts>`, and the compiler model lists the function default as `transacts + no_rollback`. The practically important truth on the current build is: **rely on nothing implicitly.** Annotate every `<decides>` function — and every helper reachable from a rollback/failure context — with `<computes>` or `<transacts>`. That rule is correct under either description, and it's what makes code actually compile today.

### ⚠️ `<no_rollback>` contexts (no rollback to rely on)

These execute as `<no_rollback>`. Failure inside them does **not** roll back, and you generally cannot run rollback-dependent `<decides>` work there:

- **Scene Graph lifecycle:** `OnAddedToScene`, `OnBeginSimulation`, `OnSimulate`, `OnEndSimulation`, `OnRemovingFromScene`, `OnReceive` (scene events).
- **Event signalling:** `Event.Signal(...)`, `subscribable_event.Signal(...)`, `listenable` signal paths.
- **`spawn{...}` and `branch{...}`** bodies.

**Real consequence:** a method that signals an event **must not be `<decides>`** — return a `logic`:

```verse
# Recompute() fires a StatChangedEvent (no_rollback), so this can't be <decides>.
TrySpend(Amount:float):logic =
    if (Current >= Amount):
        set Base = Base - Amount
        Recompute()         # signals an event
        true
    else:
        false
```

### Hiding effects

| Construct | Hides | Result |
|---|---|---|
| `if (Failable):` | `<decides>` | branches on success/failure |
| `option{ e }` | `<decides>` | `?t` (filled or `false`) |
| `logic{ e }` | `<decides>` | `logic` (`true`/`false`) |
| `spawn{ e }` | `<suspends>` | returns `task(t)` immediately |

### Forbidden / notable combinations

- **`<suspends>` + `<decides>` is forbidden.** After yielding, rollback is impossible. To fail then suspend, hide the failure first: `option{Failable[]}` before `Sleep`.
- Constructors are implicitly `<transacts>`, may be `<decides>`, **never `<suspends>`**.
- `<writes>` implies divergence (a write can trigger reactive re-evaluation), so it isn't `<computes>`.

---

## 2. Failure as control flow (`<decides>`)

Verse has **no exceptions and no boolean control**. Expressions **succeed** (produce a value) or **fail** (produce nothing). `<decides>` marks functions that can fail.

**Failable expressions** (require a failure context):

| Expression | Fails when |
|---|---|
| `Arr[I]` | index out of bounds |
| `Map[K]` | key absent |
| `A / B` | integer divisor 0 |
| `Type[Value]` | downcast doesn't match |
| `X > 0`, `X = Y`, … | comparison false (returns left operand on success) |
| `Opt?` | optional is empty |
| `F[...]` | `F` is `<decides>` and fails |

**Failure contexts** and how they treat failure:

- **Handle** (consume): `if(e0){e1}else{e2}` (e0), `for(e0){e1}` (e0 filter), `first(e0){e1}` (e0 filter), `option{e}`, `logic{e}`, `not e`, `e0 or e1` (e0).
- **Pass through** (propagate): `<decides>` function body, `e0 and e1`, `e0; e1`, `if` branches, `for`/`first` bodies, `case` arms, `block{e}`, `return e`.
- **Disallow:** `loop{e}` (retracts the failure context — wrap with `if`), `defer{e}`, non-`<decides>` function bodies, `spawn{e}`, `case` scrutinee, `sync`/`race`/`rush` arms.

**Idioms:**

```verse
if (P := Players[Name]):        # combine test + bind; P valid only in then-branch
    Use(P)
Result := A[] or B[] or C[]     # first success (all-failable chain still propagates)
Valid  := for (X : Items, Pred[X]) { X }   # filter
First  := first{ E : Gen }      # first of a generator (<decides>)
Opt    := option{ Arr[0] }      # decides → ?t     (boundary conversion — see below)
Flag   := logic{ Cond[] }       # decides → logic  (boundary conversion — see below)
not Map[K]                      # succeeds iff K absent
```

**Style: propagate failure as far as possible.** Failure *is* Verse's composition mechanism — a `<decides>` chain reads top-to-bottom, needs no re-testing, and keeps every speculative effect in one rollback scope. Converting early into data (`option{...}`, `logic{...}`, `X[] or Default` sentinels) is failure with extra steps: each caller must unwrap/re-test, and the rollback link is severed. So keep helpers `<decides>` (returning the useful value, not a `logic`), consume failure with `if` at the genuine decision point, and reserve the converters for boundaries that force a value: storing into a field, an `@editable` default, hiding `<decides>` before `<suspends>`, or the `<no_rollback>` frontier (a signalling `TryX(...):logic` wrapper around a failable transactional core).

**More idioms:** `Type[Value]`/`Type(Value)` casts apply to **classes/interfaces only** (not primitives). Wrap a failable *mutation* in `logic{ set Map[K] = V }` to run it and discard failure. `false?` yields `:logic` — if a branch needs another type, follow with `Err("…")` (yields `:false`, type-checks anywhere) or use a real failable check.

**Anti-patterns:** no `return` inside a `for`/`if` body in a failure context (the body *is* the value — use `first` or collect+index). Don't redundantly guard before an implicitly-failable op.

**Error vs failure:** `<decides>` = expected, recoverable, rolled back. `Err(Message:string)` = unrecoverable programmer error, aborts, not rolled back, yields `:false`.

**Typed errors with `result`:** model recoverable, *inspectable* failures with `result(success_type, error_type)` (`MakeSuccess(...)`, `MakeError(...)`, `.GetSuccess[]`, `.GetError[]`). Common for `TryFire`/`Craft`/inventory ops:

```verse
TryFire(...)<suspends>:result(fire_success, fire_error) =
    if (OnCooldown[]). return MakeError(on_cooldown_error{})
    ...
    MakeSuccess(fire_success{})
```

---

## 3. STM rollback

In `<transacts>` (+ `<decides>`) contexts, mutations are **speculative** and roll back on failure:

```verse
Purchase(Cost:int)<decides><transacts>:void =
    set Gold -= Cost     # happens speculatively
    Gold >= 0            # if this fails, Gold is restored automatically
```

`not e` also rolls back `e`'s effects. Works for primitives, containers, and class fields. **Nested transactions** merge into the parent on commit and roll back independently on abort (an inner failure doesn't disturb the outer unless it propagates). `[VVM]` has the most complete STM — it can roll back not just memory but map-key changes, unification/placeholder state, and spawned tasks (`spawn`/`when`/`upon`/`set live` inside a failing transaction are undone); `[BPVM]` disallows `spawn` in `<transacts><decides>`. **But remember**: lifecycle methods, signalling, and `spawn`/`branch` are `<no_rollback>` — mutations there are not rolled back.

---

## 4. Concurrency

### Structured primitives

| Expr | Completes | Cancellation | Returns |
|---|---|---|---|
| `sync` | all arms | — | tuple of results (static arm order) |
| `race` | first arm | cancels losers | winner's result |
| `rush` | first arm | losers keep running *until this scope exits* | first result |
| `branch` | immediately | ends with parent | void (fire-and-forget child) |

```verse
race:
    block:
        Sleep(5.0)
        "timeout"
    block:
        E.Await()
        "done"
```

> **Structured-concurrency guarantee.** Every `sync`/`race`/`rush`/`branch` task is bounded by the scope that *contains* the expression. When that scope exits — returns, fails, or is itself cancelled — **all tasks it manages are cancelled and cleaned up.** You never leak a task by leaving its scope; conversely a task lives only as long as its scope keeps running.

- **≥2 arms** for `sync`/`race`/`rush`. All `race` arms must be `<suspends>`.
- **No iterative form.** `race(E : Gen){ E.Task() }` does **not** exist (neither do `sync(...)`/`rush(...)` iterated). To race over a dynamic set, spawn each as a task and coordinate manually, or restructure with `event`s. (The compiler has internal `*Iterated` node types, but they are not shippable Verse.)
- `sync`/`race`/`rush` arms **cannot be `<decides>`** (conflict with `<suspends>`).
- `[BPVM]` has known cancellation bugs (branches/rush arms may outlive their context); `[VVM]` cancels correctly. Don't rely on defers in cancelled arms; don't signal an event that wakes a `branch` in the same coroutine from a `sync`/`rush` arm.
- **`rush` is rarely the right tool.** Its "losers keep running" lasts only while the **enclosing scope keeps running after the `rush`** — so if the `rush` is a function's tail expression, the losers are cancelled the instant it returns (behaving like `race`, minus the point). To run work concurrently and stop it when something else finishes, use a **`race` loser one scope up**, or a `branch`.
- **`race` same-tick tie-break:** if two arms complete on the same tick, the **earlier (static-order) arm wins**; `race` may skip spawning later arms once one completes.
- **`sync` returns its tuple in static arm order**, not completion order.
- **Don't `return`/`break` out of a concurrency arm.** Returning from a `sync` arm returns from the enclosing function (`[VVM]` keeps other arms running, `[BPVM]` cancels them); returning from a *winning* `race`/`rush` arm **crashes on `[VVM]` (known bug)** and hijacks the return on `[BPVM]`; a losing arm's return never executes. `break`-ing a loop that lives inside a `branch` is also disallowed.

### `spawn` (unstructured)

```verse
T:task(int) = spawn{ AsyncFunc() }   # T.Await() for the result
spawn{ FireAndForget() }             # no handle
spawn. F()                           # dot/colon forms also valid
```

- The spawned function **must be `<suspends>`**. Single call only (`spawn{F(); G()}` is an error). Spawned tasks are **independent** (not children) and survive their spawner's cancellation. The `spawn` body is `<no_rollback>`.

### `task(t)` & cancellation

`task(t)` is `class<abstract><final>(awaitable(t))` — a **holdable handle** to a running async op, and `spawn` is the only construct that returns one (`T := spawn{ F() }`, then `T.Await()` anywhere later, even after the spawning scope is gone). `Await()<suspends>:t` (public; callable multiple times; **sticky** — a finished task returns its cached result immediately). A task moves through **Active → Completed/Canceled → Settled** (terminal); **double-cancellation is a no-op**. Cancellation is **cooperative** — a task runs to its next suspension point; insert `Sleep(0.0)` in long loops. Defers run during cancel (LIFO). Cancelling a parent cancels children recursively (LIFO); spawned tasks are not children. **`task.Cancel()` is `<epic_internal>`** — not public API right now (pending a `<suspends>` self-cancel redesign), so you can't cancel a held task directly; end its managing scope instead.

### `first` (take one; pairs with generators)

`first` is like `for` but returns the **first** value reaching its body and **fails (`<decides>`)** if none:

```verse
first (E : MyGen, SomeFilter[E]) { E }    # first matching
Item := first{ E : MyGen }                # shorthand — first item of a generator, no array conversion
Idx  := first (I -> V : Arr; V = Target) { I }
```

Use it to pull one result from Scene Graph `Find*` queries (which return `generator(t)`).

### Events

```verse
E := event(int){}            # parametric event
E.Signal(42)                 # producer (no_rollback)
V := E.Await()               # consumer (FIFO), <suspends>
```

- `event(t)` is **non-sticky** (a late `Await` after `Signal` waits forever). Subscribers/awaiters resume **FIFO**, a `Signal` runs the awaiting code **immediately** (up to its next suspend), multiple signals before the next tick are **separate triggers**, and signalling is **reentrant-safe** (you may signal from within an await handler). `sticky_event(t)` remembers the last signal (returns it to repeated awaits until `ClearSignal()`). `subscribable_event(t)` (observer pattern, `Subscribe(handler)→cancelable`) is **transactional** — rolled back on failure.
- Interfaces: `awaitable(t)` (`.Await()`), `signalable(t)` (`.Signal(v)`), `subscribable(t)` (`.Subscribe(cb)→cancelable`), `listenable(t)` = awaitable+subscribable (most engine events are `listenable`), `cancelable` (`.Cancel()`), `cancelable_container` (`.AddCancelables(array{...})`, then `Cancel()` all).
- Device events come typed as `device_event_void` / `device_event_agent` / `device_event_optional_agent`.
- Pattern: components expose `SomeEvent():listenable(payload)`; you `Subscribe` in `OnBeginSimulation`/`OnAddedToScene`, store the `cancelable`, and `Cancel()` it in `OnEndSimulation`/`OnRemovingFromScene`.

### Timing

`Sleep(Seconds)<suspends>`: `0.0` = yield until next tick (use to avoid hogging the frame in a loop); `Inf` = never completes except on cancel (handy as a non-winning `race` arm); `< 0.0` = completes immediately without yielding. `NextTick()` guarantees a tick boundary. `GetSecondsSinceEpoch()` / `GetSimulationElapsedTime()` are constant within a transaction (replay determinism).

---

## 5. Reactive / live variables `[VVM]`

VerseVM adds reactive bindings (not in BPVM):

- `var live X:int = Expr` (mirrors changes), `live Y:int = Expr` (read-only reactive). Also an input→output form: `var Base->Health : Clamp.Evaluate = 50` (independent access control via `var In<private>->Out<public>`), and functions-as-types for liveness (`var X:Multiply` where `Multiply` has `<reads>`). Live vars need an explicit type and **can't be module-scope or class fields**.
- `when (Guard): Body` returns a cancellable `task(void)` (runs every time Guard changes), `upon (Guard): Body` returns `task(int)` (fires once), `await{ Expr }` suspends until Expr changes — the `await{ X = Y }` form waits until the condition holds and works on array/map/struct elements `[VVM]`. You **cannot `break`/`return`** from these bodies; a guard's condition can't use `<suspends>`/`<writes>`/`<allocates>`. A plain `set` on a live var **cancels** its reactive binding; `await` respects transactions (rolled-back changes don't trigger).
- Dependency tracking is **runtime, not static**. Cycles re-evaluate to a fixed point (stable when comparable), and because a write may re-trigger, **`<writes>` implies `<diverges>`**.
- `batch: ...` groups mutations so reactive triggers fire once: notifications fire in trigger-registration order, a variable that changed multiple times delivers only the first, nested batches act as one, a failed batch triggers nothing, and `batch` can't contain `<suspends>`. Guards must be side-effect-free (`<reads><computes><decides>`, no `<writes>`/`<allocates>`).

A project may keep `.versefuture` files as experimental live-variable rewrites — treat them as not-yet-active.
