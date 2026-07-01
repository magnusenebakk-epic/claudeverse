# Verse Language Reference

Core language: types, literals, variables, operators, functions, classes, modules, advanced types. For effects/failure/concurrency see `effects-failure-concurrency.md`. For the Scene Graph object model see `scene-graph.md`.

## Block syntax (3 interchangeable forms)

```verse
if (X). DoThing()          # dot-space (single expression)
if (X): DoThing()          # colon + indented block
if (X) { DoThing() }       # braces
```

Works everywhere blocks appear (`if`, `for`, `case`, function/class bodies, `sync`/`race` arms…). `=` can be followed by an indented block:

```verse
F(X:int):int =
    Y := X * 2
    Y + 1                  # last expression is the return value
```

**Line continuation:** a line continues only if it ends with a binary operator, comma, or open delimiter. **Infix operators must *trail* the previous line, not lead the next** (opposite of Python/Swift):

```verse
X := A +        # OK
    B
if (P and       # OK
    Q): ...
Y := Arr        # WRONG: `.Length` on next line parses as a new expression
.Length
```

## Comments

```verse
# line comment
<# block comment — may span
   multiple lines #>            # (block comments nest)
<#> everything from here to the end of this indented region is a comment
    still commented (indented under the <#>)
```

Three forms: `#` line, `<# … #>` block (nestable), and `<#>` **indented block** (comments the whole indented region that follows). **Comments are stripped inside string literals too** — `"a<#x#>b"` evaluates to `"ab"` (surprising; escape `<` as `\<` if you need a literal `<#`).

## Reserved words

Cannot be used as identifiers: `break catch const continue do else if in is not return set then until var where with yield alias live mutable ref`, plus the type/container built-ins `type comparable subtype array map tuple option weak_map`. Some (`alias const mutable yield continue with catch until`) are **reserved but not yet implemented** in BetaVerse — the compiler rejects them as names but they do nothing. This is why `continue`/`const` "don't work."

## Types & literals

Primitives: `int` (64-bit; VVM is arbitrary-precision bigint, BPVM errors on overflow), `float` (IEEE-754; `1.0` needs the decimal; `Inf`, `-Inf`, `NaN`; note `NaN = NaN` is **true** in Verse), `rational` (no literal — produced by `int/int`; no arithmetic operators yet — convert via `Floor`/`Ceil`), `logic` (`true`/`false`, lowercase only), `char`/`char32` (`'a'`, `'😀'`), `string` (`= []char`; `"value: {X}"` interpolation calls `ToString`; length is **bytes**), `void`, `any`, `true`, `false`.

```
42  -17  +42  0xDeAd   # int — lowercase 0x only, hex DIGITS may be mixed case; 1_000 invalid
0123                   # == 123, NOT octal (no octal literals); - -11 (spaces ok) is valid
3.14  1e10  2.5E-3     # float — e/E, optional exponent sign; 1. and .5 invalid; f16/f32 suffix unsupported
"hi {Name}"            # string interpolation; "{0u1F600}" interpolates a code point
'a'   '\n'   0o41      # char (1 byte): 0oXX = exactly 2 hex digits, lowercase 0o only
'😀'  0u1F600          # char32: 0uXXXXXX = up to 6 hex digits, lowercase 0u only
true  false            # logic
```

- **Escapes** (in strings/chars): `\t \n \r \" \' \\ \{ \} \< \> \& \# \~`.
- **char literals reject Unicode surrogates** (U+D800–U+DFFF) at parse time; non-characters U+FFFE/U+FFFF are allowed.
- **`any` is special:** the top type has **no equality**, **cannot be a map key**, and needs an **explicit wrap** — `X:any = any(42)`, not `X:any = 42`.
- **No `typeof` operator.** A value belongs to *every* type that characterizes it (`3` is an `int`, a `nat`, a `positive`, …); types just describe values, and equality is by value (extensional).

**Type hierarchy:** `any` (top) ⊃ primitives/classes/interfaces/structs/enums/containers ⊃ `false` (bottom, uninhabited). `comparable` = supertype of all `=`/`<>`-comparable types; `persistable` = serializable types. Both `comparable`/`persistable` are usable as **parameter types** directly (`F(A:comparable)`). Numeric subtyping: `nat` <: `int` <: `rational` (`rational` is arbitrary-precision). `true` is the unit type whose only value is `false`. There's also a `path` type for `/…`-style path literals.

**Lenient evaluation:** Verse is **neither lazy nor strict** — every subexpression runs as soon as its inputs are ready, so definitions can be used before they textually appear (`X + 2, X := 3` is valid, and a field initializer may reference a later field `[VVM]`). This is *not* laziness: an unused binding still evaluates (`X := loop<>; 3` does not skip the loop). Ordering is observable only through **effects** — effectful ops (`set`, signals) keep their written order, while pure `<computes>` work may be reordered or run early.

**Containers:**

| Type | Syntax | Construct | Access |
|---|---|---|---|
| Array | `[]t` | `array{1,2,3}` | `A[i]` (failable), `A.Length`, `A + B`, `A.Slice[i,j]` |
| Map | `[k]v` | `map{"a"=>1}` | `M[k]` (failable), `for (K->V : M)`, `set M[k]=v` |
| Tuple | `tuple(t1,t2)` | `(a, b)` | `T(0)` **parens, compile-time, never fails**; **no destructuring** |
| Option | `?t` | `option{v}` or `false` | `O?` (failable unwrap), `O? or Default`, `O?.Field` |
| Generator | `generator(t)` | (from `Find*` etc.) | lazy, single-iteration, `for`-only; take one with `first{...}` |

- **Array methods return a *new* array**; the failable ones (`<decides>`, call with `[]`): `Find[Elem]:int` (index), `Remove[Index]`, `RemoveFirstElement[Elem]`, `ReplaceElement[Index,New]`, `Insert[Index,Elem]` (valid `0<=Index<=Length`); the total ones: `RemoveAllElements[Elem]`, `ReplaceAllElements`, `ReplaceAll[Pattern,Repl]`. **`.Slice[I,J]` end index J is exclusive.** Mixed element types infer to the common supertype (unrelated → `[]any`); jagged arrays allowed; a `tuple(int,int)` coerces to `[]int`.
- **Map**: `k` must be `comparable` (`logic`/`int`/`float`/`char`/`string`/enums/`<unique>` classes/comparable structs & tuples; **not** arrays-except-`string`, maps, `:void`, `:true`, `type`, functions, plain classes, interfaces). **Iteration is insertion order.** `for (V : M)` (single binder) iterates **values**. `ConcatenateMaps(M1,M2)` — later map wins on key collision. Map **equality includes insertion order** (`map{0=>"a",1=>"b"} <> map{1=>"b",0=>"a"}`); in a literal with duplicate keys, position comes from the first, value from the last. Float keys: `-0.0 = +0.0`; `NaN` works as a key.
- **Variance:** arrays & generators covariant; **map keys contravariant, values covariant**; **function params contravariant, returns covariant, effects invariant**. A `var` field is invariant.
- `weak_map(k,v)` — restricted map for **module-scoped persistent global state**: not iterable, no `.Length`, observe only by known key. Value type must be `<persistable>`; key type must carry `<module_scoped_var_weak_map_key>` (e.g. `player`, `session`). **Hard limit of 4** persistent weak_maps per island — can't work around it with a plain `var` or an `<allocates>` constructor. Partial update: `set PlayerData[P].Score = N`.
- **Tuple**: `T(0)` index must be a **literal** (compile-time). `()` is the empty tuple; `(X)` is *grouping*, not a 1-tuple — there is **no 1-tuple literal** (use `T:tuple(int) = array{5}`). Auto-expands into call arguments.
- **Option**: `option{1;2}?` = 2 (sequenced, last), `option{1,2}?` = a tuple. **Nested options do NOT collapse** — `option{option{X?}} <> option{X?}` and `??t` is a valid distinct type.

**Gotchas:** cross-type comparison fails (`0 = 0.0` fails); no implicit numeric conversion (use `1.0 * N` to go int→float; there is **no `Float()`/`ToFloat()`**); `false = () = array{} = map{}` (empty values equal `false`).

## Variables & mutability

```verse
X := 42            # immutable binding (type inferred)
Y:int = 42         # immutable, explicit type
var Z:int = 0      # mutable (explicit type required; must have initial value)
set Z = 10         # mutation (the `set` keyword is required)
set Z += 1         # compound assignment
```

- **Immutable by default.** `var` declares a mutable cell; `set` mutates it.
- **Reading a `var` returns an immutable snapshot** ("freeze"); `Y := X` copies. **Setting deep-copies** ("melt"). Exception: class references copy the *pointer* (shared mutability).
- **Deep mutability:** `var X:[]int` makes every nested level mutable.
- `set L = R`: **R must succeed** (no failable projection on the RHS). For self-referential updates use compound assignment: `set Arr[I] -= V` (not `set Arr[I] = Arr[I] - V`).
- Module-scope: type annotations required (no `:=`); mutable module state must be a `weak_map`.
- **`var` is an expression** returning its value: `X := (var Y:int = 42)` binds `X = 42`. `X := E` desugars to an existential binding + unification (`exists X. X = E`).
- **Explicit pointers** underlie `var`/`ref`: `New(t)(Init)` `<allocates>` a cell → `^t`; `Ptr^` reads it (`<reads>`); `set Ptr^ = V` writes (`<transacts>`). `var`/`ref` are implicitly-dereferenced pointers over this layer.
- **Avoid side effects inside an index expression** (`A[set A[1]=13; 1]`) — L-expression evaluation order differs between `[BPVM]` and `[VVM]`.
- `[VVM]` adds `ref` (alias without copy), `var ref` (rebindable alias), and reactive **live variables** (`var live`, `when`, `upon`, `await`, `batch`) — see `effects-failure-concurrency.md`.

## Operators (by precedence, high→low)

`. [] () {} ?` → prefix `- + not` → `* /` → binary `+ -` → `< <= > >=` (chainable, right-assoc) → `= <>` → `and` → `or` → `..` → `:= =`.

- **Arithmetic:** `+ - *` keep type; **`int / int → rational` and `/` is `<decides>`** (fails on 0 divisor). `+` concatenates strings/arrays. `Mod[a,b]` and `Quotient[a,b]` are `<decides>` functions (call with `[]`).
- **Comparison:** failable; **return the left operand on success** (`(1 < 2) = 1`); chainable (`0 <= x <= 100`). Ordering (`< > <= >=`) only for `int`/`float`/`string`. Cross-type ordering fails; cross-type `int`/`float` equality fails too (promote: `1.0 * N`).
- **Logical:** `P and Q` (both succeed → Q's value), `P or Q` (first success), `not P` (succeeds iff P fails, **rolls back P's effects**). Short-circuit.
- **`?`** unwraps `?t`→value (fails if empty) or `logic`→`false` (succeeds if `true`). Only for `?t`/`logic`; it is **not** a generic "force".
- **`..`** range (`1..5` inclusive both ends; `5..1` is **empty**, not reverse; **`for`/`first` domains only**).
- **`[]` vs `()`:** `()` = total (compiler proves domain); `[]` = failable (domain test decidable). Call `<decides>` functions and index arrays/maps with `[]`; access tuples with `()`.
- **`of` / `at`:** `F of {Arg}` = `F(Arg)` (total); `F at {Arg}` = `F[Arg]` (`<decides>`). `?.` is the optional-chaining postfix.
- **`not not e`** tests `e` without committing its effects: yields `false` if `e` succeeds (effects rolled back), fails if `e` fails.
- **Integer overflow:** `[BPVM]` 64-bit add/sub/mul/div/negation that overflows (incl. `-(-2^63)`, `Floor(-2^63 / -1)`) is a **runtime error**; `[VVM]` uses bigint, so no overflow.
- `{}` construction takes **no trailing comma**.
- Custom infix: `operator'name'(A:t, B:t):t = ...` (precedence of `*`). Overloadable: `+ - * /` and custom; **not** comparisons or compound-assign.

### `Floor`/`Ceil`/`Round`/`Int` brackets (frequent error)

`Floor`/`Ceil` have **two overloads with different effects**, so different brackets:

| Call | Arg type | Result | Effects |
|---|---|---|---|
| `Floor(R)` / `Ceil(R)` | `rational` | `int` | `<computes>` → **parens** |
| `Floor[F]` / `Ceil[F]` | `float` | `int` | `<decides>` (fails on NaN/Inf) → **brackets** |
| `Round[F]` / `Int[F]` | `float` | `int` | `<decides>` → **brackets** |

So `Mid := Floor((Low + High) / 2)` (the `/` made a `rational`). Never `Int[Floor(R)]` — `Floor(R)` is already `int` and `Int[]` only takes `float`. int→float is `1.0 * N`. (The float `Floor`/`Ceil` overload is `<reads><decides>`.)

**Rounding direction differs:** `Floor` rounds toward −∞ (`Floor(-1.5) = -2`), `Int` truncates toward zero (`Int[-1.5] = -1`), `Round` ties to even (`Round[0.5] = 0`).

## Functions

```verse
Add(X:int, Y:int):int = X + Y
Create(Name:string, ?Health:int = 100, ?Level:int = 1):entity   # named/optional params (prefix ?)
Create("Bob", ?Level := 5)                                       # call with named arg
```

- After the first named param, all must be named. Defaults evaluate in the defining scope.
- **Tuple auto-flatten:** `F(X:int, Y:int)` callable as `F((3,5))`; `F(P:tuple(int,int))` callable as `F(3,5)`.
- **Effects** go after `()` and before `:` → `F(X:int)<decides><computes>:int`. Definitions always use `()`; **calls use `[]` when the function is `<decides>`**.
- **Extension methods:** `(V:int).Double():int = V*2` then `5.Double()`. Class methods take precedence. These are how the engine adds methods to `entity`/`agent`/etc. (e.g. `Agent.GetFortCharacter[]`).
- **Generics:** `Id(X:t where t:type):t = X`; constrain with `t:subtype(comparable)`. `[VVM]` supports higher-order/complex parametric; `[BPVM]` errors on some.
- **Lambdas:** `A=>B` is a value, `a->b` is a **type**. In BetaVerse, `=>` lambdas are only implemented in map literals (`map{"a"=>1}`); use **nested functions** for closures (`[VVM]` only; `[BPVM]` has no nested functions). Nested functions capture `var`s by reference.
- Intrinsics (`Abs`, `Min`, `Max`, `Clamp`, `Mod`, `Sqrt`, `Pow`, `Exp`, `Log`, `Log10`, `Sin`/`Cos`/`Tan`/`ArcTan2`, `PiFloat`, `Print`, `Err`, `ToString`, `Concatenate`, …) aren't first-class — wrap to store.
- **Overloading is by *parameter types only*, never by effects.** Types that all accept `false` are **indistinct** for overload resolution and collide: `logic`, `?t`, `[]t`, `[k]v`, `true`, `void`; so does interface-vs-implementing-class and any subtype relation. Refined types make valid overloads only when the refinements are **disjoint** (overlapping = ambiguous). You **cannot reference an overloaded name, extension method, or intrinsic without calling it** (not first-class).
- **Argument evaluation order:** positional (left→right), then named (in written order), then defaults (in parameter order). No duplicate parameter names; no attributes on parameters; refined (`where`) types aren't allowed in destructured tuple params; an empty-tuple param needs `()` at the call site (`F(5, ())`).

## Classes, interfaces, structs, enums

```verse
my_class<public> := class(base_class, some_interface):
    Name:string                          # immutable field (required at construction if no default)
    var Health:int = 100                 # mutable field
    TakeDamage(A:int):void = set Health -= A
    block:                               # construction-time init code
        Log("created {Name}")
```

Construct: `P := my_class{Name := "Bob"}` (inline) or block form `P := my_class:` then indented `Name := "Bob"`.

**Specifiers** (on the *identifier*): `<abstract>` (can't instantiate), `<final>` (can't inherit/override), `<unique>` (identity equality — required to compare class instances), `<concrete>` (all fields have defaults — instantiable as `T{}`), `<castable>` (enables runtime downcast `T[Value]`), `<persistable>` (serializable; needs `<final>`, no `var`, no superclass), `<final_super>` (Scene Graph: must derive directly from `component`).

**`<override>` is required** when overriding an inherited member, and it attaches to the **identifier**, not the parens. Effects go after the parens:

```verse
Area<override>()<computes>:float = Width * Height   # RIGHT
Area()<override><computes>:float = ...              # WRONG (that slot is for effects)
```

Override rules: can narrow return type, can **reduce** (not add) effects, can't override `<final>`, can't narrow read-accessibility.

- **`Self`** = current instance. **`(super:)Method()`** / `(super:)Field` = superclass. `(super:)` only for side-effect-free defaults.
- **Constructors:** `MakeX<constructor>(N:string)<transacts> := my_class:` then field bindings. Implicitly `<transacts>`; **never `<suspends>`**; **can be `<decides>`** (call with `[]`). Used heavily for UI builders (see `patterns.md`).
- **Downcast:** `if (D := derived_type[Value])` (failable `[]`, needs `<castable>`); upcast `Base := base_type(Value)` (total). Type-test an `entity`'s components / interface impls this way.
- **Acyclicity:** no cyclic immutable data — break cycles with `var` or `?`.
- **`interface`**: abstract methods + optional default impls (`M():void = {}`); multiple inheritance OK. Implement on classes/components to enable interface-based queries (e.g. `GetDescendantComponentsWithInterface(some_iface)`).
- **`struct`**: value type (deep copy, comparable if fields are); **cannot** inherit, have methods, or have `var` fields; cannot contain itself. Fields default to `<internal>` (like everything) and may be `<public>`/`<internal>`/`<scoped>` — **not** `<private>`/`<protected>`.
- **Class `block:` cannot use `<suspends>` or `<decides>`.** Do failable/async setup in a lifecycle method instead.
- **Construction / block order differs by VM:** `[VVM]` runs subclass blocks first (bottom-up, lenient — a field initializer may reference a field declared *later*); `[BPVM]` runs superclass first (top-down, strict — needs dependency order). **Don't rely on cross-class block order** for portable code. In global-scope class instances, `[BPVM]` doesn't run `block:` at all while `[VVM]` does (and flags `<allocates>` there).
- **Diamond interface inheritance:** duplicated fields are merged; conflicting methods still need `<override>`.
- **Specifier evolution asymmetries** (post-publish): `<final>`/`<unique>`/`<final_super>` are **add-only**, `<abstract>` is **remove-only** (you may make a class concrete but never abstract), `<castable>` is irreversible. `GetCastableFinalSuperClass()` returns the most-specific `<final_super>` for versioning.
- **`enum`**: `color := enum: Red; Green; Blue` → value `color.Red`. `<closed>` (default, exhaustive `case`) vs `<open>` (needs wildcard/`<decides>`). `[BPVM]` caps enums at **256 enumerators**; `[VVM]` has no limit. A reserved-word enumerator uses `(keyword_enum:)public`; attribute scopes are `@attribscope_enum` (the type) vs `@attribscope_enumerator` (values).

**`case`** pattern matching (on `int`/`logic`/`char`/`string`/enums/refinements — **not** float/objects/tuples):

```verse
case (Mode):
    effect_stacking_mode.Stack   => ...
    effect_stacking_mode.Refresh => ...
    _ => Default
```

## Control flow

```verse
if (Cond) then A else B            # expression form
if (C): Then else: Else            # block form; Cond must be FAILABLE
for (X : Arr): F(X)                # iterate; returns []result
for (I -> V : Arr): ...            # index->value (arrays) / key->value (maps)
for (I := 0..N-1): ...             # range domain uses :=  (collections use :)
for (X : Arr, X > 0): F(X)         # filter clause (failable)
first (X : Arr, Pred[X]) { X }     # first success; <decides>; fails if none
loop: ...; if (Done). break        # only loop has break; every stmt must succeed
```

- **`for` is an expression returning an array** of body results. **No `break`/`continue`/`return`.** Setup failure → no iterations; filter failure → skip; body failure → result not added.
- **Range domains bind with `:=`**, collection iteration pulls with `:`. `for (I := 0..N-1)` not `for (I : 0..N-1)`.
- **`loop`** is the only construct with `break` (`break` has bottom type, takes no args — use a `var` to carry a result out). **Every statement in a `loop:` body must succeed** — wrap any `<decides>` op in `if(...)`. No `<decides>` directly in a loop body.
- **No reverse ranges:** `for (I := 0..N-1): Process(N-1-I)`.
- **`case` scrutinee must NOT be failable**, and a **closed enum is exhaustiveness-checked** (omitting a case is an error; a redundant `_` may be flagged). Open enums need `_`.
- **`for` with multiple range domains is a cartesian product:** `for (X := 1..3, Y := 1..3). (X,Y)` yields 9 tuples. Iteration order is guaranteed: arrays sequential, **maps in insertion order**, strings char-by-char.
- **`loop` must contain a non-`break` statement** (a loop whose only statement is `break` is invalid). `loop` retracts the *surrounding* failure context, so any failable op inside (indexing, comparison, integer `/`, failable let-RHS) must be guarded with `if`. Evaluation is always lexical (left→right, top→bottom). `if (logic{X?})` is rejected — `logic{}` removes failability, and an `if` condition must be failable.
- **`block: …`** groups statements and returns the last value — the tool for multi-statement `race`/`sync`/`rush` arms. **`profile("Label"). Code`** measures and logs wall-clock time. `yield`/`continue` are not supported.
- **`defer: Cleanup()`** runs on scope exit (LIFO); can't contain `<suspends>`/`<decides>`/`return`/`break` but **can** contain `branch`/`spawn`; runs on cancellation but **not** on failure-rollback (and it's skipped in the failure-consuming contexts `not`/`or`/`option{}`/`logic{}`).
- **`return`** is implicit (last expression). Explicit `return` is **tail-only** in failure contexts.
- **Prefer the multi-line block form for multi-clause `for`/`if`** (idiomatic Verse). Put each source/binding/filter (or condition) on its own line under `for:` / `if:`, with the body under `do:` / `then:`(/`else:`):
  ```verse
  for:
      Drop : Drops            # source
      Item : Spawn(Drop)      # nested source → flattened (flat-map)
      Keep[Item]              # filter (failable)
  do:
      Item
  ```
- **`for` evaluates to the array of its body values**, so drop the manual accumulator when the body is a clean 1:1 map: write `for(...): X` (or `for: … do: … X`) and let the for-expression *be* the result (often the function's last expression) instead of `var Acc:[]t = array{}; for(...): set Acc += array{X}`. Nested `for`s flatten into one multi-source `for` (a flat-map). Keep an accumulator only when the body appends conditionally or mutates other state.

## Modules, `using`, paths

- A **folder is a module**; all `.verse` files in it share the module scope. Module bodies hold definitions only (no executable statements). Nest modules to mirror folders.
- **`using { /Verse.org/SceneGraph }`** (absolute), `using { ../UI/MainMenu }` (relative), `using { Gameplay.Stats }` (by module name). Statement-level; parent before nested; not transitive.
- **Access:** `<public>`, `<internal>` (default — module-only), `<protected>`/`<private>` (class-only), `<scoped{M1,M2}>`, `<epic_internal>`. Read/write split: `var<internal> Health<public>:int = 100` — write accessibility **can** be narrowed on override, read accessibility **cannot**. Access specifiers **can't** apply to local variables. `<scoped{}>` is **module-level only** (not inside class defs) and its code can reach out but outside code can't reach in; `<epic_internal>` ≈ `<scoped{/Fortnite.com,/UnrealEngine.com,/Verse.org}>` (so `/User@…/` content can't touch it).
- **Constructor accessibility:** an uninitialized (no-default) field must be **at least as accessible as the constructor**. The factory idiom `my_type := class<internal>` gives a public type with an internal constructor. A `<public>` definition **cannot expose an `<internal>` type** in its signature.
- **Paths & qualifiers:** definitions live at paths like `/User@Fortnite.com/MyProject/...`. Disambiguate with `(module:)F()`, `(super:)M()`, `(local:)X`, or a full path `(/MyProject/Utils:)Helper()`. A common idiom is a **module alias** used as a qualifier, e.g. `(LUF:)vector3{...}` to pin the Forward/Left/Up SpatialMath `vector3`.
- **Directory → module compilation:** `*.verse` appended as-is; `*.value` → `Name := (contents)`; `Self.verse` inlined at module root; `*.txt` → `string`, `*.bin` → `[]byte`; subdirectories → nested modules. **Ignored:** dotfiles (`.*`), `*.ignore[.*]`, and any file with no `.` in its name. Files are processed in lexicographic (UTF-8) order.
- **`import(...)` alias** (usable as a qualifier): `LUF := import(/Verse.org/SpatialMath)`. **Modules are not first-class** — you can't assign, pass, or store one.
- **Local-scope `using{E}`** brings an entity/object's members into a block: inside `using{E}`, `Print(Name)` means `E.Name` and `set Volume = 0.8` means `set E.Volume`. Block-scoped; errors on a duplicate identifier or a module path.

## Persistence serialization

`Persistence.ToJson[Data]` / `Persistence.FromJson[Json, type]` map Verse↔JSON: metadata `$package_name`/`$class_name`; field names get an `x_` prefix; option `None`→`false`, `Some(v)`→`{"":v}`; tuple→array; map→`[{"k","v"}]`; enum→`"package::enum::Value"`; missing fields fall back to defaults. Persistable initializers must be effect-free; a `<persistable>` struct **cannot be modified after publication**; removing an enum value is always a breaking change (open enums are forward-compatible for *added* values, closed enums can't change).

## Persistence

`weak_map(player|session, persistable)` for global state. Persistable: `int`/`float`/`logic`/`string`/`char`/arrays/maps/tuples/options of persistable, `<final><persistable>` classes, `<persistable>` structs/enums. **Not** persistable: `any`, `comparable`, `rational`, functions, `weak_map`, interfaces, non-final classes. `weak_map` keys: `session_key` (round) / `persistent_key` (cross-session).

## Advanced types

- **Refinement:** `percent := type{_X:float where 0.0 <= _X, _X <= 1.0}`; `positive := type{_X:int where _X > 0}`. Operators are `<= < >= >` **with literals only** (no `=`/`<>`, no variables/functions). `-0.0` is not `< 0.0`; `Inf`/`-Inf` allowed, `NaN` disallowed. `[VVM]` refinements also catch overflow (`i64[MaxInt+1]` fails). Cast: `if (V := percent[Input])`.
- **Parametric:** `container(t:type) := class: Value:t`. Same type arg = same type (memoized). Cannot be `<castable>`/`<persistable>`. **No polymorphic recursion.** Recursive OK (`list(t) := class{ Value:t; Next:?list(t) = false }`). Aliases: `string_map(t:type) := [string]t`.
- **Metatypes:** `subtype(T)` (runtime type value; covariant; can't be the value itself, a map key, or carry attributes), `concrete_subtype(t)` (instantiate `T{}` — needs `<concrete>`), `castable_subtype(t)` (dynamic cast `T[Value]` — needs `<castable>`; classes/interfaces only, no nesting, doesn't compose with `subtype`). With a `subtype(Base)` constraint you can access `Base`'s members inside the generic body. **`classifiable_subset(t)`** / `classifiable_subset_var(t)` = a hierarchy-aware type set (`Contains[]`, `ContainsAll[]`, `ContainsAny[]`, `Add()`, `Remove[]`, `FilterByType()`, `+` union) respecting inheritance. Used pervasively in Scene Graph as `@editable var Prefab:?concrete_subtype(entity)` and `GetComponent[castable_subtype(component)]`.
- `<castable>` cannot apply to structs/modules/functions/parametric types; is inherited by subclasses; a castable class may inherit from a parametric one but not vice-versa.
- **Attributes:** apply with `@attr; Def` or `Def @attr`. `@editable` (expose in UEFN editor), `@editable_number(int){MinValue := option{1}}`, `@editable_text_box{MultiLine := true}`, `@category`, `@display_name`, `@tool_tip`, `@available{MinUploadedAtFNVersion := 4000}` (version gate), `@experimental`, `@deprecated`, `@localizes`, `@import_as` (bind to C++/Blueprint), `@replicated`, `@field`. Define custom: `my_attr := class(attribute){}` with `@attribscope_*`.
- **Localization:** `Msg<localizes>:message = "Text {Param}"` (module scope only). Formatting: `int` params are comma-grouped, `float` standard, `string` as-is; `{0u004d}` for a code point; escape braces `\{Name\}`. **Markup** in messages: `<b;Hello>`, `<i;x>`, `<link{url=x};Click>`, nested `<b,i;…>`, `<Tag:>content</>`.
