---
name: claudeverse
description: >-
  Expert knowledge of the Verse programming language and the UEFN Scene Graph
  (entity/component) system, plus the Verse/UnrealEngine/Fortnite APIs. Use this
  whenever reading or writing Verse (.verse) code, designing Scene Graph entities
  and components, debugging Verse compile errors or effect-specifier problems, or
  working with UEFN/Fortnite gameplay APIs (fort_character, damage/health,
  inventories, devices, UI widgets). Verse looks imperative but is a
  functional-logic language — habits from C/Python/Rust/JS systematically fail.
---

# ClaudeVerse — Verse & Scene Graph

Verse is a **functional-logic** language with OOP, used for Unreal Editor for Fortnite (UEFN). It *looks* like Python/Rust but the semantics are different, and habits from imperative languages **systematically fail to compile**. The single biggest source of mistakes is writing what "feels right" instead of following Verse's rules. Slow down; read the relevant reference before writing.

> **Primary reader is an LLM.** This skill is written densely for that. When a task touches a topic, read the matching `reference/*.md` file end-to-end before writing code — the cost of reading is small; the cost of a wrong assumption is a compile error or wrong behavior.

## The mental-model shifts that matter most

| You expect (imperative) | Verse reality |
|---|---|
| Statements; functions "return" via `return` | **Everything is an expression.** The last expression is the value. `return` is tail-only inside `<decides>`/failure contexts. |
| `if`/comparisons produce booleans | **No booleans as control.** Expressions **succeed (produce a value) or fail**. `x > 0` returns `x` on success or *fails*. `(1 < 2) = 1`. |
| Exceptions / `Result<T,E>` for errors | **Failure is control flow.** `<decides>` functions succeed-or-fail; mutations roll back on failure (STM). |
| `int / int → int` | **`int / int → rational`** (and `/` is `<decides>`). Get back to `int` with `Floor(R)` — **parens**, because `Floor` is total on `rational`. |
| `x / 2.0`, `n + 0.5` mix freely | **Arithmetic is same-type-only.** No `(int, float)` overloads exist and no implicit promotion — `Count / 10.0` is a compile error. Promote first: `(1.0 * Count) / 10.0`. |
| `arr[i]` just indexes | `Arr[I]` is **failable** (`<decides>`); needs a failure context. Tuples use `T(0)` (parens, never fails). |
| `f(x)` always | `()` = compiler must *prove* arg in domain (total); `[]` = domain test is *decidable* (failable). **Call `<decides>` functions with `[]`.** |
| Mutable by default | Immutable by default. `var` to declare mutable, `set` to mutate. Reading a `var` returns an immutable snapshot. |
| `for`/`while` with `break`/`continue`/`return` | `for` is an **expression returning an array**; no `break`/`continue`/`return` in it. Use `loop`+`break`, `first`, or filter clauses. |

## Effects — the current reality (READ THIS)

Every function signature has an **effect set** written as `<specifier>` after the parameter list and before `:returntype`:
`F(X:int)<decides><transacts>:int = ...`

**Current toolchain default is `<no_rollback>` with no heap effects.** You must opt in:
- `<computes>` — pure: no heap reads/writes/allocations, cannot fail. (Most restrictive.)
- `<transacts>` — may read **and** write **and** allocate the heap, **and supports rollback**.
- `<reads>` — read-only heap access (a sub-effect of `<transacts>`).
- `<decides>` — **failable** (the cardinality "0 or 1 values"). Call sites use `[]`.
- `<suspends>` — async (can yield). Call sites need an async context; **cannot combine with `<decides>`**.

**Mandatory pairing rule (current build):** a function is `<no_rollback>` by default, and `<no_rollback>` code **cannot be called from a rollback context**. Therefore:

- **Every `<decides>` function must ALSO carry `<computes>` or `<transacts>` explicitly.** A bare `<decides>` is uncallable from `if`/`for`-filters/other `<decides>` bodies.
- **Every helper called from a `<decides>`/`<transacts>` body must also be `<computes>` or `<transacts>`** — even if the helper never fails. A plain `Helper():void = ...` carries `<no_rollback>` and breaks its caller.

```verse
GetFirst(Arr:[]int)<decides><computes>:int = Arr[0]   # decides + heap effect: OK
IsPositive(X:int)<decides><computes>:void = X > 0      # predicate: decides + computes
Spend(Amt:int)<decides><transacts>:void = set Gold -= Amt; Gold >= 0  # mutates → transacts
```

*(This will flip to `<transacts>`-as-default in a future build; until then, be explicit.)*

### `<no_rollback>` contexts you cannot rely on rollback inside

These run as `<no_rollback>`, so **failure does not roll back** and you generally **cannot place `<decides>` work that needs rollback** there:

- **Scene Graph lifecycle methods**: `OnAddedToScene`, `OnBeginSimulation`, `OnSimulate` (`<suspends>`), `OnEndSimulation`, `OnRemovingFromScene`, `OnReceive`.
- **Event signalling**: `Event.Signal(...)` / `subscribable_event.Signal`.
- **`spawn{...}` and `branch{...}` bodies.**

Practical consequence (seen in real code): a method that signals an event **cannot be `<decides>`** — return a `logic` instead. Example: `TrySpend(Amount:float):logic` (not `<decides>`) because it calls `Recompute()` which fires a change event.

**Mirror rule:** `if` conditions and `for` domains are rollback contexts, so a no_rollback function **cannot even be called there** — `if (Wallet.Add(N)?)` is error 3512; hoist (`Added := Wallet.Add(N)` … `if (not Added?)`). Any helper a `for` iterates (`for (X : Helper())`) needs `<transacts>`/`<reads>`.

## Failure & the `[]` vs `()` rule (quick)

Failable expressions (need a failure context: `if(...)`, `for(...; filter)`, `<decides>` body, `option{}`, `logic{}`, `not`, `or`):
`Arr[I]`, `Map[K]`, `A / B`, `Type[Value]` (downcast), comparisons (`<`, `=`, …), `Opt?`, `<decides>` calls.

```verse
if (P := Players[Name]):   # test + bind in one step; P valid in the then-branch
    Use(P)
```

**Propagate failure as far as possible — don't launder it.** Keep helpers `<decides>` so failure composes up the chain, and consume it with `if` at the *actual* decision point. Converting early (`option{...}`, `logic{...}`, `X[] or Default`) forces every caller to re-test and severs the rollback link. Convert only at a boundary that forces a value: a stored field / `@editable`, before `<suspends>`, or the `<no_rollback>` frontier (e.g. a signalling method returning `logic`).

## Concurrency (quick)

- `sync{A; B}` (all, returns tuple), `race{A; B}` (first wins, cancels losers), `rush{A; B}` (first wins, losers keep running — but only in-scope, see below), `branch{...}` (fire-and-forget child, dies with its scope), `spawn{ F() }` (returns a holdable **`task(t)`** you can `Await()` later; unstructured — OUTLIVES its scope, *not* cancelled with the spawner; `F` must be `<suspends>`).
- **Structured-concurrency guarantee:** `sync`/`race`/`rush`/`branch` tasks are bounded by the scope that *contains* the expression — leave that scope (return, fail, or cancel) and every task it manages is cancelled and cleaned up. A `rush` "loser" therefore survives only while the enclosing scope keeps running *after* the `rush`; if the `rush` is a function's tail, losers die immediately (so `rush` is rarely worth it — prefer a `race` loser one scope up, or `branch`).
- **There is NO iterative form** of `sync`/`race`/`rush`. You cannot write `race(E : Gen){ E.Task() }`. Spawn per-item tasks into a `race` arms list manually, or restructure.
- `sync`/`race`/`rush`/`<suspends>` **conflict with `<decides>`** (no rollback after yielding).
- Timing: `Sleep(Seconds)<suspends>` — `0.0` = yield until next tick; `Inf` = wait forever (only resumes on cancel); `< 0.0` = complete immediately without yielding.

## `first` — pull one value (great with generators)

`first` is like `for` but evaluates to the **first** value reaching its body, and **fails (`<decides>`) if none**:

```verse
first (E : MyGen, SomeFilter[E]) { E }   # first matching element
Item := first{ E : MyGen }               # shorthand: first element of a generator, no array conversion
Idx  := first (I -> V : Arr; V = Target) { I }  # first index whose value matches
```

This matters because Scene Graph `Find*` queries return **`generator(t)`**, not arrays — `first{...}` is the idiomatic "take one".

## Scene Graph in one screen

UEFN's modern model. Read `reference/scene-graph.md` before writing components/prefabs.

- **`entity`** = the object in the world; hierarchical (parent/children); carries **tags**. A `class(entity)` is a **prefab**.
- **`component`** = re-usable logic/data attached to an entity. Subclass with `class<final_super>(component)`. **One component per subclass-group per entity.**
- **Lifecycle** (override these; always call `(super:)`): `OnAddedToScene` → `OnBeginSimulation` → `OnSimulate()<suspends>` (the component's behavior over time) → `OnEndSimulation` → `OnRemovingFromScene`. All are `<no_rollback>`.
- **Be event-reactive, not tick-driven.** Express behavior as awaiting events and structured concurrency playing out over time (`Await`, `race`, `sync`, sequencing) — never poll with `loop: Check(); Sleep(0.1)`. A `Sleep`-cadence loop is only right when the effect itself is periodic; frame-coupled work uses `tick_events`.
- **Access from a component**: `Entity` (the parent entity). `Entity.GetComponent[some_component]` (`<decides>`), `Entity.AddComponents(array{...})`, `Entity.AddEntities(array{...})`, `Entity.GetParent[]`, `Entity.RemoveFromParent()`.
- **Hierarchy queries return generators**: `FindDescendantEntities(entity_type)`, `FindDescendantComponents(component_type)`, `FindAncestorComponents(...)`, `FindDescendantEntitiesWithTag(tag_type)`, etc. Take one with `first{...}`.
- **Transforms**: `entity.GetGlobalTransform()` / `SetGlobalTransform()` / `GetLocalTransform()` use `/Verse.org/SpatialMath` (`vector3` with **Forward/Left/Up** axes). But `fort_character.GetTransform()`/`GetViewRotation()` return the **deprecated** `/UnrealEngine.com/Temporary/SpatialMath` (`vector3` with X/Y/Z). Convert with `FromTransform`/`FromRotation`/`FromVector3`. **These are two different `vector3`/`rotation`/`transform` types — don't mix them.**
- **Events**: built-in `event(t)` (`Await`/`Signal`); components expose `listenable(t)` events you `.Subscribe(callback)` (returns `cancelable`) or `.Await()`.

## Reference index — read the one(s) your code touches

| File | Read when your code involves… |
|---|---|
| `reference/language.md` | types, literals, `var`/`set`/`ref`, operators, functions/generics/extension methods, classes/interfaces/structs/enums, modules/`using`/paths, refinement/parametric types, attributes (`@editable`, …) |
| `reference/effects-failure-concurrency.md` | any effect specifier, `<decides>`/failure, STM rollback, `sync`/`race`/`rush`/`spawn`/`branch`, `event`/`listenable`, `first`, live variables/`when`/`upon`/`await` |
| `reference/scene-graph.md` | entities, components, lifecycle, hierarchy queries, scene events, `tick_events`, collision/overlap/sweep, tags, transforms, the two SpatialMath namespaces, built-in components (mesh/light/camera/sound/particle/interactable/stackable/keyframed-movement) |
| `reference/apis.md` | `agent`/`player`/`session`/`team`, `fort_character`, damage/health/shield, playspaces, teams, inventories/items (Itemization), `creative_prop`/`creative_device`, weapons (Armory), Marketplace, UI widgets, Input, `debug_draw`, colors, math, containers |
| `reference/gotchas.md` | quick lookup of the mistakes that most often break Verse code (and the right idiom) |
| `reference/patterns.md` | idiomatic plumbing patterns (event-reactive component skeleton, cancelable subscriptions, manager lookup, per-player data association incl. the state-entity proxy workaround, typed results, declarative UI, input, recursive sync/race, utils helpers) |

## Naming conventions

Types `snake_case` (`player_manager`, `door_component`). Functions/vars/params/fields `PascalCase` (`GetCurrent`, `TargetEntity`). Avoid shadowing built-ins (`Max`, `Min`, `Floor`, `Clamp`, `NaN`, …) — causes "ambiguous identifier".

## Flavor tags

`[BetaVerse]` = shippable (BPVM + VerseVM, what you write). `[VVM]`/`[BPVM]` = VM-specific behavior. `[MaxVerse]` = theoretical, not shippable. When two VMs differ (integer overflow, lenient vs strict class init, nested-function support, advanced STM), prefer the cross-VM-safe form; details in the references.

## External resources

- **The Verse book** — <https://verselang.github.io/book/> — the most complete public language reference (chapters: expressions, primitives, containers, operators, mutability, functions, control, failure, structs/enums, classes/interfaces, types, access, effects, concurrency, live variables, modules, persistable, evolution). Authoritative for language semantics. **Caveat:** it marks `first`, nested functions, and live variables as "unreleased" (some, like `first`, do work on current builds), and states the default effect is `<transacts>` (the current toolchain default is effectively `<no_rollback>` — see `reference/effects-failure-concurrency.md`).
- **Project API digests** — the generated `*.digest.verse` files for your build are the source of truth for exact API signatures; `grep` them. Regenerate when a symbol seems missing.
