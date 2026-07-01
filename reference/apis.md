# API Reference (Verse / UnrealEngine / Fortnite)

The API surface you call. Three layers: `/Verse.org/*` (core language libs), `/UnrealEngine.com/*` (engine: UI, itemization, diagnostics), `/Fortnite.com/*` (gameplay: characters, damage, devices, weapons).

> **Digest = source of truth, and it's versioned.** The authoritative signatures live in the generated `*.digest.verse` files for your project's build (Verse.digest.verse, UnrealEngine.digest.verse, Fortnite.digest.verse, plus your project's Assets.digest.verse). The digests referenced while writing this skill were build `++Fortnite+Release-41.10-CL-55335788`. **If a symbol/overload isn't in your digest, regenerate the digests from the current build** before assuming it's missing — the engine adds APIs frequently, and many carry `@available{MinUploadedAtFNVersion := N}` gates and `@experimental`. To find a symbol fast, `grep` the digest rather than reading it whole (the Fortnite digest is ~11.5k lines).

---

## `/Verse.org/Verse` — core

- **`Print(Message:string, ?Duration:float, ?Color:color)<transacts>`** (also `message`/`diagnostic` overloads). For logs prefer `/UnrealEngine.com/Temporary/Diagnostics`'s `log`.
- **`Err(Message:string)<computes>:false`** — unrecoverable runtime error; yields bottom type.
- **Math** (all `/Verse.org/Verse`): `Abs`, `Min`, `Max`, `Clamp` (int + float), `Floor`/`Ceil`/`Round`/`Int` (float→int, `<decides>`), `Sqrt`, `Pow`, `Exp`, `Ln`, `Log(B,X)`, `Sin`/`Cos`/`Tan`/`ArcSin`/`ArcCos`/`ArcTan`/`ArcTan(Y,X)` + hyperbolic, `Lerp(From,To,T)`, `Sgn`, `Mod[a,b]`/`Quotient[a,b]` (`<decides>`, Euclidean — `Mod` sign follows divisor), `(X).IsFinite[]`, `PiFloat`, `Inf`, `NaN`. **`Floor(R:rational)` parens vs `Floor[F:float]` brackets.**
- **Strings/arrays:** `ToString(int|float|char|string)`, `Concatenate`, `Join(Strings, Sep)`, array methods `Find`/`Insert`/`Remove`/`Replace*`/`Slice`/`RemoveFirstElement`/`RemoveAllElements`.
- **Events & async interfaces:** `event(t)` (`Signal`/`Await`), `sticky_event(t)`, `subscribable_event(t)`; interfaces `awaitable(t)`, `signalable(t)`, `subscribable(t)`, `listenable(t)` (= awaitable+subscribable), `cancelable` (`.Cancel()`), `cancelable_container`. (See `effects-failure-concurrency.md`.)
- **`result(success_type, error_type)`** interface — `MakeSuccess(v)`, `MakeError(e)`, `.GetSuccess[]`, `.GetError[]`.
- **Marker interfaces:** `disposable` (`Dispose`), `invalidatable` (`IsValid[]`), `enableable` (`Enable`/`Disable`/`IsEnabled[]`), `showable` (`var Show:logic`).
- **`modifier(t)`** (`Evaluate(InValue:t)<reads>:t`) and **`modifier_stack(t)`** (`AddModifier(m, Position:rational)→cancelable`, `Evaluate`) — composable transform pipelines.
- **`message`** (localized text; `Localize(Msg)<reads>:string`, `Join(Messages, Sep)`), `diagnostic` (`ToDiagnostic(Value:any)`), `locale`.
- **`Easing`** submodule: `CubicBezier`, `Linear`/`Ease`/`EaseIn`/`EaseOut`/`EaseInOut` (`(:float)<reads>:float`).

## `/Verse.org/Simulation`

- **`agent`** (`class(entity)`), **`player`** (`class(agent)`; `IsActive[]`; a `weak_map` key), **`team`**, **`session`** (`GetSession()<reads>:session`, `.Environment()` → `Edit`/`Private`/`Live`).
- **`Sleep(Seconds:float)<suspends>`**, `GetSimulationElapsedTime()<transacts>:float`.
- **`Tags`** submodule: `tag` (`class<abstract><castable>`), `has_tags` (Add/Remove/Contains…), `tag_key`, `tag_view` (`Has`/`HasAny`/`HasAll`), `tag_search_criteria`.
- **`@editable`** attribute family lives here: `editable`, `editable_number(t){MinValue/MaxValue}`, `editable_slider(t)`, `editable_text_box{MultiLine}`, `editable_vector_*`, `editable_container`.

## `/Verse.org/SpatialMath` (current — Forward/Left/Up)

`vector3{Forward:=, Left:=, Up:=}`, `rotation`, `transform{Translation:=, Rotation:=, Scale:=}`. `MakeRotationFromYawPitchRollDegrees`, `MakeRotationFromEulerDegrees`, `IdentityRotation()`, `Slerp`, `MakeShortestRotationBetween`, `(R).GetForwardAxis/GetLeftAxis/GetUpAxis/Invert/GetYawPitchRollDegrees`, `DotProduct`, `CrossProduct`, `Distance`, `(V).Length()/MakeUnitVector()/IsFinite[]`, `Lerp`, `v * rotation`, `v * transform`, `DegreesToRadians`. See `scene-graph.md` for the **two SpatialMath namespaces** and `From*` converters.

## Other `/Verse.org` modules

- **`Colors`**: `color{R:=,G:=,B:=}` (ACES linear), `color_alpha`, `MakeColorFrom*` (SRGB/Hex/HSV/Temperature), `NamedColors.*` (CSS names).
- **`Assets`**: `mesh`, `material`, `texture`, `sound_wave`, `particle_system`, `animation_sequence`, `input_action(t)`, `input_mapping`, `has_icon` (`var Icon:texture`).
- **`Random`**: `GetRandomFloat(Lo,Hi)`, `GetRandomInt(Lo,Hi)`, `Shuffle(Arr)`.
- **`Input`** `[4000+]`: `GetPlayerInput(Player)→player_input` (`AddInputMapping`/`RemoveInputMapping`/`GetInputEvents(action)`), `input_events(t)` (`TriggerActivationEvent`/`BeginDetectEvent`/`CancelActivationEvent`/… as `listenable(tuple(player, t, …))`), `input_method` enum, `(Player).ProjectWorldToViewport[]`/`DeprojectViewportToWorld()`.
- **`Concurrency`**: `awaitable(t)`, `task(t)` (`Await()<suspends>`).
- **`AgentGroup`** `[4000+ experimental]`: `agent_group(member_info)`, `member_info_interface`.
- **`Progression`** `[experimental]`: `quest`, `quest_collection`, `quest_participant`, `agent_quest_participant`.
- **`Chat`** `[4000+ experimental]`: `voice_channel`/`text channel`, `entity.AddChatChannel`.

---

## `/Fortnite.com/Characters` — `fort_character` (central)

`fort_character` is an `interface` implementing `positional, healable, healthful, damageable, shieldable, game_action_instigator, game_action_causer`.

```verse
Char := Agent.GetFortCharacter[]            # (agent).GetFortCharacter()<transacts><decides>:fort_character
Char.GetAgent[] / Char.GetEntity[]          # bridge to agent / Scene Graph entity
Char.EliminatedEvent()                       # listenable(elimination_result)
Char.GetViewRotation() / GetViewLocation()   # ⚠ returns Temporary (X/Y/Z) SpatialMath
Char.GetTransform()                          # ⚠ Temporary SpatialMath
Char.IsActive[] / IsDownButNotOut[] / IsCrouching[] / IsOnGround[] / IsInAir[] / IsFalling[] / IsGliding[] / IsFlying[] / IsInWater[]
Char.TeleportTo[Position, Rotation]          # Temporary vectors
Char.PutInStasis(stasis_args) / ReleaseFromStasis()
Char.Show() / Hide() / SetVulnerability(logic) / IsVulnerable[]
Char.GetLinearVelocity() / SetLinearVelocity(v) / ApplyLinearImpulse(v) / ApplyForce(v) / GetMass()   # Verse.org vectors
Char.JumpedEvent() / CrouchedEvent() / SprintedEvent()
```

## `/Fortnite.com/Game` — damage / health / shield (build your combat on this)

```verse
healthful  : GetHealth()<transacts>:float, SetHealth(H)<transacts>, GetMaxHealth(), SetMaxHealth(M)
             # Health clamps to [1.0, MaxHealth]; CANNOT SetHealth(0) — eliminate via Damage instead.
shieldable : GetShield/SetShield/GetMaxShield/SetMaxShield, DamagedShieldEvent(), HealedShieldEvent()
damageable : Damage(Amount:float), Damage(damage_args), DamagedEvent():listenable(damage_result)
healable   : Heal(Amount:float),  Heal(healing_args),  HealedEvent():listenable(healing_result)

damage_args   = struct{ ?Instigator:?game_action_instigator, ?Source:?game_action_causer, Amount:float }
damage_result = struct{ Target:damageable, Amount:float, ?Instigator, ?Source, IsWeakpointDamage:logic }
healing_args/healing_result — analogous.
game_action_instigator / game_action_causer — marker interfaces (who/what caused it).
elimination_result = struct{ EliminatedCharacter:fort_character, EliminatingCharacter:?fort_character }
fort_round_manager : SubscribeRoundStarted(cb)/SubscribeRoundEnded(cb)  via entity.GetFortRoundManager[]
```

Apply damage with a known source: `Char.Damage(damage_args{Amount := 25.0, Instigator := MyInstigator})`. Listen: `Char.DamagedEvent().Subscribe(OnDamaged)`.

## `/Fortnite.com/Playspaces` & `/Fortnite.com/Teams`

- **`fort_playspace`** (`creative_object.GetPlayspace()` or `entity.GetPlayspaceForEntity[]`): `GetPlayers()`, `GetParticipants()`, `GetTeamCollection()`, `PlayerAddedEvent()`/`PlayerRemovedEvent()`/`ParticipantAddedEvent()`/`ParticipantRemovedEvent()` (all `listenable`).
- **`fort_team_collection`**: `GetTeams()`, `AddToTeam[Agent,Team]`, `IsOnTeam[Agent,Team]`, `GetAgents[Team]`, `GetTeam[Agent]`, `GetTeamAttitude[a,b]` → `team_attitude` (`Friendly`/`Neutral`/`Hostile`).

## Itemization — `/UnrealEngine.com/Itemization` (+ `/Fortnite.com/Itemization`)

- **`inventory_component`**: `AddItem[Item]`/`AddItemDistribute[Item, ?AllowMergeItems]`/`RemoveItem[Item]` → `result(*_result, []*_error)`; `GetItems()`/`GetItems(type)`/`FindItems()`/`FindItems(type)` (generators incl. descendants); `GetInventories()`/`FindInventories()`; `GetEquippedItems()`; events `AddItemEvent`/`RemoveItemEvent`/`EquipItemEvent`/`UnequipItemEvent` (`listenable`). `CanAddItem[…]`/`CanRemoveItem[…]`.
- **`item_component`**: `GetParentInventory[]`, `IsEquipped[]`, `Equip[]`/`Unequip[]` → `result`, `Drop[]`/`PickUp[Inv]`, `ChangeInventoryEvent`/`ChangeEquippedEvent` (`listenable(change_*_result)`), `Categories:[]item_category`.
- Inventory selection uses **scene events**: `find_inventory_event` (sent down to choose), `add_item_query_event`/`equip_item_query_event` (sent up to veto with `AddError`).
- **Fortnite** (`/Fortnite.com/Itemization`): `fort_inventory_component` (+ `fort_inventory_weapon_hotbar_component`, `build_hotbar`, `harvest_tool`, `trap`, `resources`, `ammo`, `currencies`), `FortniteItemCategories.*` (`WorldItem`/`Currency`/`Trap`/`Ammo`/`Resource`/`WeaponMelee`/`WeaponRanged`), `FortniteRarities.*` (`Common`…`Exotic`), `fort_item_pickup_interactable_component` `[4040+]`.
- `rarity` / rarity subclasses live in `/Verse.org/SceneGraph` (`common_rarity`…`legendary_rarity`); `rarity_component`.

## `/Fortnite.com/Devices` — creative devices & props

- **`creative_device`** (`class<concrete>(creative_object_interface)`): override **`OnBegin()<suspends>`** / **`OnEnd()`**; `GetTransform()`/`TeleportTo[]`/`MoveTo()<suspends>`/`GetGlobalTransform()`/`Show()`/`Hide()`. This is the classic non-Scene-Graph entry point; a Scene-Graph project mostly uses components instead, but a `creative_device` is still a common bootstrapping root.
- **`creative_prop`** (`creative_object`, `invalidatable`): `SpawnProp(asset, transform)→tuple(?creative_prop, spawn_prop_result)`; `SetMesh`/`SetMaterial`, `Show`/`Hide`, `Dispose()`, physics (`ApplyForce`/`ApplyImpulse`/`ApplyTorque`/`SetLinearVelocity`/`GetMass`/`SetDynamic`), `var CanBeDamaged`. `animation_controller` (`SetAnimation([]keyframe_delta, ?Mode)`, `Play`/`Pause`/`Stop`, `AwaitNextKeyframe()<suspends>`).
- The module also defines **~140 `creative_device_base` subclasses** (button/trigger/timer/teleporter/conditional_button/item_granter/score_manager/vfx_spawner/sentry/…) + a `Patchwork` submodule (music/audio devices). All follow a uniform `Enable()`/`Disable()` + `listenable` events shape. **`grep` the digest** for a specific device.

## Other `/Fortnite.com` modules

- **`Armory`**: customizable weapon components — `fort_weapon_component`, `fort_trace_weapon_component` (`var Damage`, `FireRate`, `MagazineCapacity`, spread/recoil multipliers, FX/sounds), `fort_multi_trace_weapon_component`; `pistol_/assault_rifle_/sub_machine_gun_/shotgun_` `_template`(entity)/`_mesh`(mesh_component).
- **`Animation/PlayAnimation`**: `fort_character.GetPlayAnimationController[]` → `PlayAndAwait(seq, …)<suspends>:play_animation_result` / `Play(seq, …):play_animation_instance`.
- **`AI`**: `npc_behavior` (override `OnBegin()<suspends>`/`OnEnd()`; `agent.GetNPCBehavior[]`), `npc_actions_component`/`guard_actions_component` (`NavigateTo`/`Attack`/`RoamAround`/… `<suspends>` → `result(_, ai_action_error_type)`), `npc_awareness_component`, `navigatable`/`focus_interface`/`fort_leashable` (via `fort_character.Get*`), `MakeNavigationTarget`, sidekicks.
- **`Vehicles`**: `fort_vehicle` (`fort_character.GetVehicle[]`), `fort_vehicle_seat`.
- **`Marketplace`**: `offer`/`entitlement_offer`/`bundle_offer`, `entitlement`, `BuyOffer(Player, Offer)<suspends>:logic`, `GrantEntitlement`/`ConsumeEntitlement`/`GetPurchasedEntitlements`, `price_vbucks`, `offer_interactable_component` (Scene Graph).
- **`Items`** / **`Weapons`**: flat catalogs of named prefabs (`SomeItem_BR_CHxSy_Rarity := class<final><concrete>(entity)`). Reference data — `grep` for the exact one.
- **`FortPlayerUtilities`**: `(player).SendToLobby()`, `(agent).Respawn[Pos,Rot]`, spectator queries.
- **`(Fortnite.com:)UI`**: HUD control (`fort_playspace.GetHUDController()` → `ShowElements`/`HideElements` over `hud_element_identifier`s), styled buttons (`button_loud`/`button_regular`/`button_quiet`), `text_block`, `slider_regular`.

---

## `/UnrealEngine.com/Temporary/UI` — widgets (build screen UI)

`GetPlayerUI[Player]→player_ui` (`AddWidget(W, ?Slot)`, `RemoveWidget`, `SetFocus`). `widget` (`SetVisibility(widget_visibility)`, `SetEnabled`). Concrete widgets:

- Containers: `canvas` (+`canvas_slot{Anchors, Offsets, Alignment, ZOrder}`, `MakeCanvasSlot(W, Pos, ?Size,…)`), `stack_box{Orientation}` (+`stack_box_slot`), `overlay`, `button` (+`button_slot`).
- Leaves: `text_block`/`text_base` (`SetText(message)`, `SetTextColor/Size/Justification/OverflowPolicy`, `var AutoWrap/WrapWidth`), `texture_block` (`SetImage(texture)`, `SetTint`, `SetDesiredSize`, tiling), `material_block`, `color_block`.
- Layout types: `anchors{Minimum,Maximum:vector2}`, `margin{Left,Top,Right,Bottom}`, `horizontal_alignment`/`vertical_alignment`/`orientation`, `widget_visibility` (`Visible`/`Collapsed`/`Hidden`), `player_ui_slot{ZOrder, InputMode:ui_input_mode}`, `widget_message{Player, Source}`.
- **`vector2`/`vector2i` live in `/UnrealEngine.com/Temporary/SpatialMath`** (X/Y), not the Verse.org SpatialMath.

UI is usually built declaratively in a `<constructor>` with a `let:` block (see `patterns.md`).

## Other `/UnrealEngine.com` modules

- **`Temporary/Diagnostics`**: `log`/`log_channel`/`log_level` (`log.Print(msg, ?Level)`), `debug_draw`/`debug_draw_channel` (`DrawSphere`/`DrawBox`/`DrawCapsule`/`DrawCone`/`DrawCylinder`/`DrawLine`/`DrawArrow`/`DrawPoint`/`DrawText`, with `?Color`/`?Duration`/`?DrawDurationPolicy`; `ShowChannel`/`HideChannel`/`ClearChannel`).
- **`Temporary/Curves`**: `editable_curve.Evaluate(Time)`.
- **`Temporary/SpatialMath`**: deprecated X/Y/Z `vector3`/`rotation`/`transform`, plus `vector2`/`vector2i`, `MakeRotation`/`ApplyYaw/Pitch/Roll`/`RotateVector`, and the `From*`/`To*` converters to/from `/Verse.org/SpatialMath`.
- **`Itemization`** (the base of Fortnite's): see above.
- **`Conversations`** `[4000+ experimental]`: LLM NPCs — `persona_component` (`Personality`/`Knowledge`/`Session`/`Voice`, `RequestToTalk`, `SubscribeToResponseType`), `llm_session.Prompt(prompt, response_type)<suspends>:result(...)`, `basic_prompt`/`persona_prompt`, voice models.
- **`Progression`**: `basic_quest`/`quest_objective`/`progress_quest_objective`/`quest_reward`/`entitlement_quest_reward` (auto-complete + auto-reward quests).
- **`(UnrealEngine.com:)JSON`**: `Parse(str)[]→value`, `value.AsObject[]`/`AsArray[]`/`AsInt[]`/`AsFloat[]`/`AsString[]`/`AsNull[]`.
- **`BasicShapes`**: `cube`/`sphere`/`plane`/`cone`/`cylinder` (all `mesh_component` subclasses).
- **`(UnrealEngine.com:)Assets`**: `SpawnParticleSystem(asset, position, ?rotation, ?delay)→cancelable`.
- **`WebAPI`**: `client_id`/`client.Get(path)<suspends>:response`/`body_response.GetBody()`, `MakeClient(id)`.
- **`SortBy((Array, Less))<computes>:[]t`** (`/UnrealEngine.com/Temporary`).
