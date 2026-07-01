# Idiomatic Patterns (Scene Graph)

Battle-tested plumbing patterns for component-based Scene Graph games. These are engine-generic shapes — how you design *gameplay systems* on top of them is your design space, and this file deliberately stays out of it.

## 1. Component skeleton — event-reactive, not tick-driven

```verse
my_component<public> := class<final_super>(component):
    @editable var Speed:float = 100.0
    var<private> Subs:[]cancelable = array{}

    OnBeginSimulation<override>():void =
        (super:)OnBeginSimulation()
        if (Mgr := Entity.GetMyManager[]):
            set Subs += array{ Mgr.SomeEvent.Subscribe(OnSomething) }   # lifecycle-tied callback

    OnSimulate<override>()<suspends>:void =      # the component's behavior over time
        loop:
            Target := AwaitTargetSpotted()       # sleep until something HAPPENS
            race:
                Pursue(Target)                   # a sequence that plays out over time…
                AwaitTargetLost(Target)          # …interrupted by an event

    OnEndSimulation<override>():void =
        (super:)OnEndSimulation()
        for (C : Subs) { C.Cancel() }
        set Subs = array{}
```

- **Do NOT write tick loops** (`loop: CheckStuff(); Sleep(0.1)`). Polling for change is the imperative habit leaking back in — Verse's model is **event-reactive systems** and **gameplay expressed over time** with structured concurrency (`Await`, `race`, `sync`, plain sequencing). A `Sleep`-cadence loop is only right when the *effect itself* is periodic (a pulse every 2s), never for detecting state changes. Frame-coupled work that must run before/after physics uses `tick_events` — the exception, not the default.
- **Two ways to consume events — pick per use case.** The stored `Subs:[]cancelable` + `Subscribe(callback)` shape fits callbacks whose lifetime is the component's: subscribe in `OnBeginSimulation`/`OnAddedToScene`, cancel in the matching teardown. Events that are part of a *flow* are better `Await`ed inside a coroutine — `OnSimulate`, a `race` arm, a `branch` — where structured concurrency does the teardown for you.
- Lifecycle methods are `<no_rollback>` — no `<decides>` work that needs rollback.

## 2. Cancelable subscriptions: `subscribable_event` + `subscription_task`

The engine's `listenable.Subscribe` returns a `cancelable`; for your own events, wrap `event(t)` so subscribers get a cancelable that stops a per-subscriber loop:

```verse
subscribable_event<public>(t:type) := class(awaitable(t), signalable(t)):
    Event<protected>:event(t) = event(t){}
    Subscribe<public>(Callback:type{_(:t):void}):subscription_task(t) =
        subscription_task(t){ Callback := Callback, Event := Event }.Listen()
    Signal<override>(Payload:t):void = Event.Signal(Payload)
    Await<override>()<suspends>:t = Event.Await()

subscription_task<public>(t:type) := class<internal>:
    Callback:type{_(:t):void}
    Event:event(t)
    CancelEvent<private>:event() = event(){}
    Cancel<public>():void = CancelEvent.Signal()
    Listen<internal>():subscription_task(t) = spawn. CallbackLoop(); Self
    CallbackLoop<private>()<suspends>:void =
        race:
            CancelEvent.Await()
            loop:
                Payload := Event.Await()
                Callback(Payload)
```

Expose component events as `subscribable_event(payload)` fields and `.Signal(...)` them (remember: signalling is `<no_rollback>`).

## 3. Tag-based manager/singleton lookup

Give a manager entity a tag; find it from anywhere via the simulation root:

```verse
my_manager_tag<public> := class(tag){}

(InEntity:entity).GetMyManager()<transacts><decides>:my_manager =
    Sim := InEntity.GetSimulationEntity[]
    MgrEntity := first{ E : Sim.FindDescendantEntitiesWithTag(my_manager_tag) }
    MgrEntity.GetComponent[my_manager]
```

## 4. Associating data with agents

There are **several ways to associate data with a `player`/`agent`** — pick per use case; don't standardize on one shape for all of a game's data:

- **Module-scoped `weak_map(session, [player]data)`** — plain per-round data, owned by the module that uses it:

  ```verse
  orb_data<public> := class: ...
  var OrbData<internal>:weak_map(session, [player]orb_data) = map{}

  (Player:player).GetOrbData()<transacts><decides>:orb_data =
      OrbData[GetSession()][Player]
  ```

  Keep each module's player data in that module — don't funnel every system's state into one shared blob. A single central accessor is fine for a genuinely shared core, not as the home for everything.

- **A map field on a manager component** — `var Data:[player]my_data = map{}` on a component in the scene; ownership and lifetime follow the manager.

- **Persistent `weak_map(player, persistable)`** — cross-session data (hard limit of 4 per island; see persistence in `language.md`).

- **The state-entity proxy — this one is a workaround.** Use it when a player needs *components or child entities* hung off them, because you currently **cannot attach entities/components to an `agent`/`player`**. A manager spawns one state-entity prefab per player on join (subscribe to `fort_playspace.PlayerAddedEvent`/`PlayerRemovedEvent`, round start/end via `entity.GetFortRoundManager[]`), caches it, and fires a `subscribable_event(player)`; systems attach their per-player components to that proxy, and per-player **watcher** objects subscribe to the player's events and tear down on leave/respawn. When agent attachment lands, attach directly and let this retire.

## 5. Typed results & error hierarchies

```verse
my_error<public> := class<computes>:                       # base
specific_error<public> := class<concrete><computes>(my_error){}
my_success<public> := class<concrete>{}

DoThing(...)<transacts>:result(my_success, my_error) =
    if (not Precondition[]). return MakeError(specific_error{})
    MakeSuccess(my_success{})

# consume:
R := DoThing(...)
if (Ok := R.GetSuccess[]) { ... }
else if (E := R.GetError[]):
    if (specific_error[E]) { Print("...") }
```

## 6. Declarative UI via `<constructor>` + `let:`

Build widget trees in a constructor, binding the runtime handles you need afterward:

```verse
MakeMyUI<public><constructor>(P:player, Source:entity)<transacts> := my_ui:
    let:
        Label := text_block:
            DefaultText := MsgReady
            DefaultTextSize := 18.0
        Stack := stack_box:
            Orientation := orientation.Horizontal
            Slots := array:
                stack_box_slot{ Widget := Label, VerticalAlignment := vertical_alignment.Center }
        Root := canvas:
            Slots := array:
                canvas_slot:
                    Widget := Stack
                    Anchors := AnchorAt(1.0, 0.5)
                    Alignment := vector2{X := 1.0, Y := 0.5}
    Player := P
    Widget := Root
    OptSlot := option{ player_ui_slot{ZOrder := 0, InputMode := ui_input_mode.None} }
```

A reusable `ui_component` base (with `Show`/`Hide`/`Initialize`, `GetPlayerUI[Player].AddWidget`, and `Shown`/`Hidden` events) plus a `ui_group` that forwards show/hide to children works well. Small helpers: `S2M(s)<localizes>:message = "{s}"`, `AnchorAt(x,y):anchors`.

## 7. Spawned-entity ownership via `source_component`

When code spawns an entity on behalf of an agent, tag it with the instigator so later logic can resolve who caused it by walking up the hierarchy:

```verse
source_component<public> := class<final_super>(component): var Source:?agent = false

# on spawn:
New := Prefab{}
Sim.AddEntities(array{New})
New.AddComponents(array{ source_component{ Entity := New, Source := option{Activator} } })

# later (extension helper): check self, then ancestors
(Entity:entity).GetSource()<transacts><decides>:agent =
    if (SC := Entity.GetComponent[source_component]):
        SC.Source?
    else:
        AC := first{ C : Entity.FindAncestorComponents(source_component) }
        AC.Source?
```

## 8. Input handling (current workaround)

`subscribable`/`.Await` on input events needs a "primer" subscription:

```verse
if (Input := GetPlayerInput[Owner]):
    Events := Input.GetInputEvents(SomeInputAction)
    Events.BeginDetectEvent.Subscribe(NullCallback)   # NOTE: workaround so .Await() fires
    loop:
        Sleep(-1.0)                                    # complete immediately, don't yield
        Events.BeginDetectEvent.Await()
        Fire()
```

`NullCallback(:any):void = {}`.

## 9. Workaround for the missing iterative `sync`/`race` — recursive binary split

Since there's no `race(E : Gen){...}`, recurse over an array of `()<suspends>:t` thunks, splitting in half and combining with binary `sync:`/`race:`:

```verse
RecursiveSync<public>(Exprs:[](type{_()<suspends>:t}) where t:type)<suspends>:[]t =
    if (Mid := Floor(Exprs.Length / 2), R := Exprs.Slice[0, Mid], L := Exprs.Slice[Mid]):
        Res := sync:
            RecursiveSync(R)
            RecursiveSync(L)
        Res(0) + Res(1)
    else if (E1 := Exprs[0]) { array{E1()} }
    else { array{} }

RecursiveRace<public>(Exprs:[](type{_()<suspends>:t}) where t:type)<suspends>:t =
    if (Mid := Floor(Exprs.Length / 2), R := Exprs.Slice[0, Mid], L := Exprs.Slice[Mid]):
        race:
            RecursiveRace(R)
            RecursiveRace(L)
    else if (E1 := Exprs[0]) { E1() }
    else { Err() }
```

## 10. A utils file of extension helpers

A small `utils.verse` of shared extension methods makes the rest of the code read cleanly. The standard set (note `import(...)` for the SpatialMath aliases, and `p_interface` as the marker base for interface queries):

```verse
LUF := import(/Verse.org/SpatialMath)                       # (LUF:)vector3{...}
XYZ := import(/UnrealEngine.com/Temporary/SpatialMath)      # (XYZ:)vector3{...}

p_interface<public> := interface{}                          # base for castable interface queries
NullCallback<public>(:any):void = {}

(M:[k]v where k:subtype(comparable), v:type).RemoveKey<public>(Key:k):[k]v =
    var New:[k]v = map{}
    for (K -> Val : M, K <> Key) { set New = ConcatenateMaps(New, map{K => Val}) }
    New

(E:entity).GetFirstDescendantComponent(ct:castable_subtype(component))<transacts><decides>:ct =
    first{ C : E.FindDescendantComponents(ct) }
(E:entity).GetDescendantComponentsWithInterface(iface:castable_subtype(p_interface))<transacts>:[]iface =
    for (C : E.FindDescendantComponents(component), IF := iface[C]) { IF }
(Inv:inventory_component).GetOwningPlayer()<transacts><decides>:player =
    player[Inv.Entity] or first (A : Inv.Entity.FindAncestorEntities(entity), P := player[A]) { P }
(Agent:agent).GetInventoryRoot()<transacts><decides>:inventory_component =
    first (Child : Agent.GetEntities(), IC := Child.GetComponent[inventory_component]) { IC }
```

Interfaces you want queryable this way must be declared `interface<castable>(p_interface)`.

## Conventions

- **Folder = module**, mirrored by a `modules.verse` declaring the nested module tree (`Gameplay := module: Stats := module: ...`).
- Types `snake_case`, members `PascalCase`; private fields `var<private> X<private>`.
- Generators are taken with the `first` macro (`first{ X : Gen }`); maps lose keys via `.RemoveKey`.
- Packages publish under `/yourname@fortnite.com/ProjectName/...` — cross-package imports use that full path.
- `.versefuture` files are a handy convention for experimental (live-variable) rewrites kept alongside the active `.verse` — not compiled as-is.
- **Regenerate the API digests from the current build** before trusting that a symbol is missing — bundled digests often lag the build a project targets.
