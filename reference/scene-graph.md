# Scene Graph (entity / component model)

The modern UEFN object model, in `/Verse.org/SceneGraph` (+ `/Verse.org/Simulation`). Replaces the older `creative_device`-centric style with **entities** that hold **components**. Lifecycle methods, signalling, and spawn/branch here are `<no_rollback>` (see `effects-failure-concurrency.md`).

## Entities & prefabs

`entity` (`/Verse.org/SceneGraph`) is the base world object: hierarchical, `<unique>`, carries **tags** (`has_tags`). A class that derives from `entity` is a **prefab** — usually authored in the editor and surfaced in `Assets.digest.verse`, but you can also subclass in code (e.g. marker bases `marker_base := class<castable>(entity){}`).

> Convention: keep logic in **components**, not in the entity subclass. Prefabs should be thin; behavior is composed from components so you can restructure prefabs without refactoring class hierarchies.

Entity methods:

```verse
Entity.GetParent[]                         # <reads><decides>:entity
Entity.RemoveFromParent()                  # detach (runs children's OnEnd/OnRemoving)
Parent.AddEntities(array{Child1, Child2})  # attach children (re-parents if needed)
Entity.GetEntities()                       # <reads>:[]entity  (direct children only)
Entity.GetComponent[some_component]        # <reads><decides>:some_component
Entity.GetComponents()                     # <reads>:[]component
Entity.AddComponents(array{Comp1, Comp2})  # add components
Entity.GetGlobalTransform() / SetGlobalTransform(T)
Entity.GetLocalTransform()  / SetLocalTransform(T)
Entity.GetSimulationEntity[]               # root entity of the experience (<transacts><decides>)
# tags:
Entity.AddTag(SomeTag) -> tag_key          Entity.RemoveTag[Key]
Entity.ContainsTag[tag_type]   Entity.ContainsAnyTag[tag_types]   Entity.ContainsAllTags[tag_types]
```

`GetComponent` is `<decides>` (fails if absent) — pair with the heap effect of its caller and use it in a failure context: `if (M := Entity.GetComponent[mesh_component]):`.

## Component

```verse
my_component<public> := class<final_super>(component):
    @editable var Speed:float = 100.0

    OnBeginSimulation<override>():void =
        (super:)OnBeginSimulation()
        # subscribe to events, set up state
```

- Subclass with **`<final_super>`** (required to be addable to an entity — guarantees it derives directly from `component`). Further subclassing of *your* component is fine without `<final_super>`.
- **One component per subclass-group per entity.** Only one `light_component` (any subtype) on an entity; use multiple entities for multiple lights.
- `component` is `<abstract><unique><castable>`. From inside, `Entity` is the parent entity (always present after construction).

### Lifecycle (override; always call `(super:)`)

```
OnAddedToScene            # in the scene; component queries are valid now
  OnBeginSimulation       # set up TickEvents/subscriptions that must complete synchronously
    OnSimulate <suspends> # async update logic (loops); cancelled before OnEndSimulation
  OnEndSimulation         # cancel cached cancelables here
OnRemovingFromScene       # final teardown
```

All are `<no_rollback>` and `<native_callable>`. `OnSimulate` is where the component's behavior over time lives: await events and let structured concurrency (`race`/`sync`/sequencing) play the gameplay out — **not** a polling tick loop (see `patterns.md` §1). `RemoveFromEntity()` removes a component (flows `OnEndSimulation`→`OnRemovingFromScene`). Predicates `IsInScene[]`, `IsSimulating[]` (`<reads><decides>`).

Typical subscription lifecycle:

```verse
var<private> Subs:[]cancelable = array{}
OnBeginSimulation<override>():void =
    (super:)OnBeginSimulation()
    set Subs += array{ SomeEvent.Subscribe(OnSomething) }
OnEndSimulation<override>():void =
    (super:)OnEndSimulation()
    for (C : Subs) { C.Cancel() }
    set Subs = array{}
```

## Hierarchy queries — return **generators**

On any `entity` (and there are component/tag variants):

```verse
Entity.FindDescendantEntities(entity_type)            # generator(entity_type)
Entity.FindDescendantComponents(component_type)        # generator(component_type)
Entity.FindDescendantEntitiesWithComponent(comp_type)  # generator(entity)
Entity.FindDescendantEntitiesWithTag(tag_type)         # generator(entity)
Entity.FindAncestorEntities(entity_type)               # generator(entity_type)
Entity.FindAncestorComponents(component_type)          # generator(component_type)
Entity.FindAncestorEntitiesWithComponent(comp_type)    # generator(entity)
Entity.FindAncestorEntitiesWithTag(tag_type)           # generator(entity)
```

`Find*` **include** the starting entity, order is unspecified, and they return **`generator(t)`** (lazy, single-iteration). To take one: `first{ X : Gen }`. To materialize: `for (X : Gen) { X }`.

```verse
# tag-based "find the manager" idiom
(InEntity:entity).GetMyManager()<transacts><decides>:my_manager =
    Sim := InEntity.GetSimulationEntity[]
    MgrEntity := first{ E : Sim.FindDescendantEntitiesWithTag(my_manager_tag) }
    MgrEntity.GetComponent[my_manager]
```

> **Convenience helpers are usually utils-file extensions, not engine APIs.** Calls like `entity.GetFirstDescendantComponent[type]` and `entity.GetDescendantComponentsWithInterface(some_iface)` are thin extension methods written on top of the engine's `Find*` generators. Typical definitions:
> ```verse
> (E:entity).GetFirstDescendantComponent(ct:castable_subtype(component))<transacts><decides>:ct =
>     first{ C : E.FindDescendantComponents(ct) }
> p_interface := interface{}   # empty marker base so interfaces can be a castable_subtype
> (E:entity).GetDescendantComponentsWithInterface(iface:castable_subtype(p_interface))<transacts>:[]iface =
>     for (C : E.FindDescendantComponents(component), IF := iface[C]) { IF }
> ```
> See `patterns.md` for the full set of these idiomatic helpers.

## Scene events

A lightweight message bus distinct from `listenable` events:

```verse
my_event<public> := class(scene_event):  Payload:int
# send:
Entity.SendDown(my_event{Payload := 5})  # this entity's components, then each child (recursive)
Entity.SendUp(my_event{...})             # this entity's components, then up to parent
Component.SendDown(...)                   # to this component's entity downward
# receive (override on a component):
OnReceive<override>(SceneEvent:scene_event):logic =
    if (E := my_event[SceneEvent]) { DoThing(E); true } else { false }
```

`OnReceive` returns `logic` — return `true` to **consume** and halt propagation. (Itemization uses scene events like `find_inventory_event`/`add_item_query_event` to choose/veto inventories.)

## Per-frame ticks

```verse
var<private> TickEvents<protected>:tick_events   # on every component
# TickEvents.PrePhysics / .PostPhysics are execution_listenable (payload = DeltaTime:float)
Cancelable := TickEvents.PrePhysics.Subscribe(OnPrePhysics)   # OnPrePhysics(DeltaTime:float):void
```

Use ticks only for frame-coupled work that must run before/after physics. Everything else should be event-reactive in `OnSimulate` (await, don't poll); reserve `Sleep`-cadence loops for effects that are genuinely periodic.

## Transforms & the TWO SpatialMath namespaces

There are **two** `vector3`/`rotation`/`transform` types. Mixing them is a type error.

| Namespace | `vector3` axes | Handedness | Used by |
|---|---|---|---|
| `/Verse.org/SpatialMath` (current) | **Forward / Left / Up** | right-handed | Scene Graph `entity` transforms, `creative_prop` physics, most new APIs |
| `/UnrealEngine.com/Temporary/SpatialMath` (deprecated) | **X / Y / Z** | left-handed | `fort_character.GetTransform()`/`GetViewRotation()`/`GetViewLocation()`, many older device APIs, `vector2`/`vector2i` |

Convert with the bridge functions in `/UnrealEngine.com/Temporary/SpatialMath`:
`FromTransform`, `FromRotation`, `FromVector3` (translation), `FromScalarVector3` (scale/magnitude), and the reverse overloads.

```verse
# fort_character.GetTransform() is Temporary (X/Y/Z) → convert to Verse.org (LUF)
CharXform := FromTransform(Char.GetTransform())
Forward   := CharXform.Rotation.GetForwardAxis()        # vector3 Forward/Left/Up
SpawnPos  := CharXform.Translation + Forward * 200.0
New.SetGlobalTransform(transform{ Translation := SpawnPos, Rotation := IdentityRotation(),
                                  Scale := vector3{Forward := 1.0, Left := 1.0, Up := 1.0} })
```

`/Verse.org/SpatialMath` essentials: `vector3{Forward:=, Left:=, Up:=}`, `rotation` (`MakeRotationFromYawPitchRollDegrees`, `IdentityRotation`, `Slerp`, `MakeShortestRotationBetween`, `GetForward/Left/UpAxis`, `Invert`), `transform{Translation:=, Rotation:=, Scale:=}`, `DotProduct`, `CrossProduct`, `Distance`, `(V).Length()`, `(V).MakeUnitVector()`, `Lerp`, `v * rotation`, `v * transform`. (To pin the namespace when both are imported, qualify: `(LUF:)vector3{...}` via a module alias, or `(/Verse.org/SpatialMath:)vector3{...}`.)

## Collision / spatial queries

On `entity`:

```verse
Entity.FindOverlapHits()                                   # generator(overlap_hit)
Entity.FindOverlapHits(GlobalTransform, Volume)            # custom pose + volume
Entity.FindSweepHits(Displacement)                         # generator(sweep_hit)
Entity.FindSweepHits(Displacement, StartGlobalTransform, Volume)
```

- `collision_volume` shapes: `collision_sphere`/`collision_capsule`/`collision_box`/`collision_point` (each `collision_element` with a `CollisionProfile`).
- `collision_profile` = a `collision_channel` (`CollisionChannels.stationary/dynamic/avatar/visibility/camera/physics`) + a channel→`collision_interaction` (`Ignore`/`Overlap`/`Block`) map. Presets in `CollisionProfiles` (`VisibilityOverlapAll`, `DynamicBlockAll`, …).
- `sweep_hit` fields: `ContactPosition`, `ContactNormal`, `SourceHitDistance`, `TargetComponent` (→ `.Entity`), `SourceHitTranslation`. `overlap_hit`: `SourceVolume`/`TargetComponent`/`TargetVolume`.

```verse
Probe := collision_sphere{ Radius := 5.0, CollisionProfile := VisibilityOverlapAll }
Hits  := Sim.FindSweepHits(Delta, StartTransform, Probe)
if (Hit := first{ H : Hits }):  Place(Hit.ContactPosition, Hit.ContactNormal)
```

`mesh_component` exposes overlap **events**: `EntityEnteredEvent`/`EntityExitedEvent : listenable(entity)` (needs `Queryable := true`).

## Built-in components (catalog)

All are `class<final_super>(component, ...)` you add to entities.

- **`transform_component`** — `var GlobalTransform`, `var LocalTransform`, `var Origin:?origin`. (Or use the `entity.Get/SetGlobalTransform` extension methods, which auto-create one.)
- **`mesh_component`** — render a mesh; `var Collidable/Queryable/Visible:logic`, `EntityEntered/ExitedEvent`. Subclasses in `/UnrealEngine.com/BasicShapes`: `cube`, `sphere`, `plane`, `cone`, `cylinder`.
- **`light_component`** family — `directional_/sphere_/spot_/rect_/capsule_light_component` (`var Intensity`, `ColorFilter`, `CastShadows`, `enableable`).
- **`camera_component`** family `[4110+]` — `perspective_/orthographic_/physical_camera_component`, `camera_director_component`, modifier stacks. `[experimental]`.
- **`sound_component`** / **`particle_system_component`** — `Play()`/`Stop()`, `var AutoPlay`, `enableable`.
- **`keyframed_movement_component`** (`KeyframedMovement`) — `SetKeyframes([]keyframed_movement_delta, playback_mode)`, `Play`/`Pause`/`Stop`, `StoppedEvent`/`KeyframeReachedEvent`. Great for projectiles/movers. `keyframed_movement_delta{Transform:=, Duration:=, Easing:=}` with easing functions (`linear_/ease_*`); playback `oneshot_/loop_/pingpong_*`.
- **`interactable_component`** / **`basic_interactable_component`** — player interaction prompts. `StartedEvent`/`SucceededEvent`/`CanceledEvent : listenable(agent)`; override `CanInteract`/`OnStarted`/`InteractMessage`; configurable `Cooldown`/`CooldownPerAgent`/`SuccessLimit`/`InteractableDuration`. Foundation for pickups, craft buttons, slots.
- **`stackable_component`** / **`basic_stackable_component`** — `var StackSize`/`MaxStackSize`, `SetStackSize`, `Split[Amount]`, `CanMergeInto[Target]`, `MergeInto[Target]`, `ChangeStackSizeEvent`. Pairs with `has_merge_rules`.
- **`possessable_component`**, **`icon_component`** (`var Icon:texture`), **`description_component`** (`Name`/`Description`), **`rarity_component`** (`var Rarity:rarity`).
- **Itemization** (`/UnrealEngine.com/Itemization`): **`inventory_component`** (`AddItem`/`AddItemDistribute`/`RemoveItem` → `result`, `GetItems`/`FindItems`/`GetInventories`, `AddItemEvent`/`RemoveItemEvent`/`EquipItemEvent`/`UnequipItemEvent`), **`item_component`** (`GetParentInventory[]`, `IsEquipped[]`, `Equip`/`Unequip` → `result`, `Drop`/`PickUp`, `ChangeInventoryEvent`/`ChangeEquippedEvent`, `Categories`). Fortnite specializations in `/Fortnite.com/Itemization` (`fort_inventory_component`, weapon/build/resource/ammo hotbars). See `apis.md`.

`enableable` interface (`Enable()`/`Disable()`/`IsEnabled[]`) is implemented by light/mesh/sound/particle/interactable components and many devices.

## Agents, players, session

`/Verse.org/Simulation`: `agent` is `class(entity)`; `player` is `class(agent)` (and a `weak_map` key); `session` (per-round, `weak_map` key, `GetSession()`); `team`. `agent`/`player` carry the gameplay extension methods (`GetFortCharacter[]`, …) — see `apis.md`. Note: you currently **cannot attach entities/components to an `agent`** — hanging them off a player uses the state-entity proxy workaround; plain per-player data has several normal homes (`patterns.md` §4).
