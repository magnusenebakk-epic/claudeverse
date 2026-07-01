# Gotchas — the mistakes that break Verse code

Quick-lookup list of where Verse departs from imperative habits. These are the things an LLM gets wrong by writing "what feels right." If you're about to write something that *looks* obvious, check here first.

## Calls & failure

- **`()` vs `[]`.** Call total functions and access tuples with `()`; call `<decides>` functions and index arrays/maps/casts with `[]`. Using `()` on a `<decides>` function (or `[]` on a total one) is a compile error.
- **No booleans for control.** Expressions **succeed or fail**. `if (X > 0):` works because `X > 0` *fails* when false. `if (SomeLogic)` is wrong unless it's failable — use `if (SomeLogic?)`.
- **Comparisons return their left operand**, not a bool: `X := (0 < 10)` makes `X = 0`. They're failable. Relational ops (`< <= > >=`) sit at **precedence level 6 and are right-associative**; `A <= B <= C` is special chain syntax checking both and returning `A`.
- **`<decides>` must be paired with a heap effect (current build).** Write `<decides><computes>` (pure) or `<decides><transacts>` (mutating). A bare `<decides>` is uncallable from failure/rollback contexts. Same for any helper called from a `<decides>`/`<transacts>` body — annotate it `<computes>`/`<transacts>` even if it can't fail.
- **`<no_rollback>` contexts.** Scene Graph lifecycle methods, event `Signal`, and `spawn`/`branch` bodies don't roll back — a method that signals an event **can't be `<decides>`**; return `logic` instead.
- **Failable expr needs a failure context.** `Arr[0]`, `Map[K]`, `A/B`, `Type[V]`, `Opt?`, comparisons must sit in `if(...)`, a `for`/`first` filter, a `<decides>` body, `option{}`, `logic{}`, `not`, or `or`. Bare `Arr[0]` in a `void` function won't compile.
- **Don't launder failure early.** Prefer `<decides>` helpers that propagate over `?t`/`logic`/`or Default` returns — converting forces callers to re-test and severs rollback. Consume with `if` at the real decision point; convert only where a value is forced (stored field, `@editable`, before `<suspends>`, the `<no_rollback>` frontier).
- **`<suspends>` + `<decides>` is forbidden.** Hide failure (`option{Failable[]}`) before suspending.
- **Constructors are never `<suspends>`.** Do async setup in `OnBegin`/`OnSimulate`, not construction.

## Numbers

- **`int / int → rational`**, and `/` is `<decides>` (fails on 0). Get back to `int` with **`Floor(R)`** — *parens* (total on rational). `Floor[F]` with *brackets* is the float overload (`<decides>`). `Int[Floor(R)]` is always wrong (`Floor(R)` is already `int`; `Int[]` only takes `float`).
- **`/=` is float-only** (integer division is failable). Use `set X = Floor(X / N)` for ints.
- **No implicit int→float.** Use `1.0 * N`. There is **no `Float()`/`ToFloat()`**.
- **`0 = 0.0` fails** (cross-type). Promote first. `"5" = 5` fails too.
- **NaN:** `NaN = NaN` is *true* in Verse; there's one NaN; `0.0` has no sign. Arithmetic that would produce NaN (`0.0/0.0`, `Inf - Inf`) is a **runtime error**, but math functions (`Sqrt(-1.0)`, `Ln(-1.0)`) **return** NaN. Float `/0.0` → `Inf` (no failure); int `/0` fails.
- **`Round` is banker's rounding** (ties to even): `Round[0.5]=0`, `Round[2.5]=2`, `Round[-1.5]=-2`.
- **Don't shadow intrinsics** (`Max`, `Min`, `Floor`, `Clamp`, `Abs`, `Mod`, …) — causes "ambiguous identifier". And intrinsics aren't first-class (can't store `Abs` in a var).

## Control flow

- **`for` is an expression returning an array.** No `break`/`continue`/`return` inside it. Filter in the domain (`for (X : Arr, Pred[X])`); for early-exit-on-first-match use **`first`**; for accumulate-until use `loop` + a `var`.
- **`loop` is the only construct with `break`** (which takes no args and has bottom type). Every statement in a `loop:` body must succeed — wrap `<decides>` ops in `if(...)`. `loop` itself returns the top type `true`.
- **`first`** returns the first body value reaching it, fails if none (`<decides>`); `first{ E : Gen }` pulls one item from a generator without building an array.
- **Flatten multi-clause `for`/`if`; use `for`'s array result.** Prefer the block forms `for:`/`do:` and `if:`/`then:`/`else:` over cramped inline `for (A, B:=…, filter):` / `if (A, B:=…):` — put each clause on its own line. Nested `for`s collapse into one multi-source `for` (flat-map): `for: A:Xs; B:F(A) do: B`. Since `for` returns the array of body values, write `for(…): X` instead of `var Acc; for(…): set Acc += array{X}` — keep the accumulator only for non-1:1 bodies (conditional appends, side mutations). Single-condition `if (X):` one-liners are fine as-is.
- **No iterative `sync`/`race`/`rush`.** `race(E : Gen){...}` doesn't exist. `rush`/`branch` can't even be used directly inside `loop`/`for` bodies (wrap in a function). For a dynamic fan-out, recurse with binary split (see `patterns.md`).
- **Ranges are ascending, `for`/`first`-only, bound with `:=`.** `5..1` is empty; reverse via `for (I := 0..N-1): Use(N-1-I)`. Collection iteration uses `:` (`for (X : Arr)`), range uses `:=` (`for (I := 0..N-1)`).
- **`case` doesn't support `float`, objects, or tuples** — only `int`/`logic`/`char`/`string`/enums/refinements. Open enums always need a `_` wildcard or `<decides>` context.
- **`defer` does NOT run on failure-rollback** (it's as if it never registered); it can't `<suspends>`/`<decides>`/`return`/`break`; runs LIFO and on cancellation.

## Mutability & bindings

- **`set L = R` requires R to succeed.** Don't put a failable projection on the RHS (`set Arr[I] = Arr[I] - V` is wrong because `Arr[I]` is failable). Use compound assignment: `set Arr[I] -= V`.
- **`set` returns the RHS value** (chainable: `set Y = set X = 5`). Compound assignment evaluates the LHS **once** (unlike the expanded form).
- **No variable shadowing anywhere** — not even nested blocks. Re-binding a name is an error; use `set` for reassignment.
- **Reading a `var` returns an immutable snapshot.** `Y := X` copies; later `set X = ...` doesn't change `Y`. Classes are the exception (reference/shared).
- **Struct-in-array = independent copy; class-in-array = shared reference.** `set A[0].Field = v` changes all class elements but only one struct element.
- **Set array element can't grow it** (`set A[i]=v` fails if `i>=Length`); append with `set A += array{v}`. Map `set M[k]=v` adds-or-updates (the insert itself is infallible but **still needs a failure context** — `if (set M[K]=V):` / `option{set M[K]=V}`); `set M[k]+=v` fails if `k` absent, so update-or-create is `if (set M[K]+=V): else if (set M[K]=V):`. Maps have **no delete** — rebuild via `for`.
- **Bindings in `and`/`or`/`not` don't escape** to the `then`-branch; but `if (X := Arr[0]):` does bind `X` in `then`.
- **`var` needs an explicit type** (`var X := 0` is an error); module-scope mutable state must be a `weak_map`.

## Types & containers

- **Tuple access `T(0)` is parens, compile-time, never fails. No destructuring binding** `(A,B) := T` — bind then index `T(0)`, `T(1)`. *(But function **parameters** can destructure: `F(A:int, (B:int, C:int))`.)*
- **`?T` not `T?`** for the type. Empty option is `false` (not null). `option{e}` wraps `<decides>`→`?t`; `o?` unwraps; `o? or Default`; `o?.Field` short-circuits.
- **`;` vs `,`** differ by context: in a tuple/`option{}`/`logic{}`, `,` builds a tuple and `;` sequences (last value). `array{}` accepts either but **can't mix** them. In `for`, both separate filters.
- **`string` length is bytes** (`"José".Length = 5`); `string = []char` of UTF-8 code units. `char` (0o.. literal, 1 byte) vs `char32` (0u.. literal) — no implicit conversion.
- **Classes aren't comparable unless `<unique>`** (then by identity). Structs compare field-by-field if all fields comparable.
- **`weak_map` is a *supertype* of `map`** — assigning a `map` to a `weak_map` loses `.Length`/iteration. Module-scoped `weak_map` only supports per-key read/write.
- **Generators (`Find*` results) aren't arrays.** No indexing, single iteration. Take one with `first{...}`; materialize with `for (X : Gen){X}`.

## Scene Graph / engine specifics

- **Two SpatialMath namespaces.** `/Verse.org/SpatialMath` `vector3` is **Forward/Left/Up**; `/UnrealEngine.com/Temporary/SpatialMath` `vector3` is **X/Y/Z** (used by `fort_character.GetTransform/GetViewRotation`). They're different types — convert with `FromTransform`/`FromRotation`/`FromVector3`. `vector2` only exists in the Temporary namespace.
- **`GetComponent[...]` is `<decides>`** — use it in a failure context, and the caller needs a heap effect.
- **One component per subclass-group per entity.** Need two lights → two entities.
- **Subclass components with `<final_super>`**; prefabs subclass `entity`. Keep logic in components, not the entity subclass.
- **Always call `(super:)OnBeginSimulation()` etc.** when overriding lifecycle methods, and cancel subscriptions in the matching teardown method.
- **No polling tick loops.** `loop: Check(); Sleep(0.1)` is the imperative habit leaking back in — be event-reactive: `Await` events, express gameplay over time with `race`/`sync`/sequencing. `Sleep`-cadence only when the effect itself is periodic; frame-coupled work uses `tick_events`.
- **Can't attach entities/components to an `agent`** (currently) — hanging components/child entities off a player needs the state-entity proxy workaround. Plain per-player *data* doesn't: a module `weak_map(session, [player]t)`, a manager-component map field, or a persistent `weak_map(player, persistable)` are all normal homes (`patterns.md` §4).
- **`agent.GetFortCharacter[]` is `<transacts><decides>`** and the character **changes on respawn** — re-fetch and re-subscribe; don't cache a `fort_character` across deaths.
- **Health clamps to `[1.0, MaxHealth]`** via `healthful.SetHealth`; you can't `SetHealth(0)` — eliminate through `damageable.Damage`.

## Modules & evolution

- **Module bodies hold definitions only** (no executable statements) and require explicit type annotations (no `:=` at module scope).
- **`using` is a statement** at file/module top level; each file needs its own imports (same-folder files see each other without imports). Import parents before nested children.
- **Publishing is a permanent contract.** Once public, you can't remove a definition, change its kind, narrow access, or widen effects/param-narrow/return-widen on final members; non-final instance members must keep exact signatures. `<castable>`/`<final_super>` are irreversible. Persistable schemas can only add defaulted fields / open-enum values. Over-specify effects (`<reads>` you might later drop to `<computes>`).
