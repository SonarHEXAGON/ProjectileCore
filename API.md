# ProjectileCore API Reference

Reference for the **integrated `ProjectileSystem_Test` build inspected on 30 October 2026**. This repository is documentation-only: it does not currently provide the source tree, dependencies, or an installable release. The older standalone export is a separate build, not evidence that every change below has reached that package.

This is fine. Please read the authority and callback sections before giving a cosmetic blade the nuclear launch codes.

## Contents

- [Setup, Authority, And Types](#setup-authority-and-types)
- [Public API Index](#public-api-index)
- [Registration And Spawn Settings](#registration-and-spawn-settings)
- [World Collision And Bouncing](#world-collision-and-bouncing)
- [Callbacks And Terminal Lifetime](#callbacks-and-terminal-lifetime)
- [Preparation And Cache Ownership](#preparation-and-cache-ownership)
- [Bursts And Patterns](#bursts-and-patterns)
- [Projectile Groups](#projectile-groups)
- [Movement, Ownership, And Patching](#movement-ownership-and-patching)
- [Safe Phase Transitions](#safe-phase-transitions)
- [Homing And Bezier Travel](#homing-and-bezier-travel)
- [Deflection And Shields](#deflection-and-shields)
- [Area Controllers](#area-controllers)
- [Control Immunity](#control-immunity)
- [Projectile Clashes](#projectile-clashes)
- [Impact Context And Attachments](#impact-context-and-attachments)
- [Distance And Age Curves](#distance-and-age-curves)
- [Physical Projectiles](#physical-projectiles)
- [Visuals And Effects](#visuals-and-effects)
- [Networking, User Data, And Recovery](#networking-user-data-and-recovery)
- [Collision Backends And Dynamic Occupancy](#collision-backends-and-dynamic-occupancy)
- [Profiling And Parallel Runtime](#profiling-and-parallel-runtime)
- [Registry And Compatibility Helpers](#registry-and-compatibility-helpers)
- [Current Limits And Verification](#current-limits-and-verification)

## Setup, Authority, And Types

An existing integrated installation normally uses `ReplicatedStorage.ModuleScripts.ProjectileCoreSystem.ProjectileCore`. It also imports game-owned utility, networking, humanoid, and VFX dependencies; this path is not a complete installation manifest. See [README](./README.md#setup-and-authority).

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage");

local ProjectileCore = require(ReplicatedStorage["ModuleScripts"]["ProjectileCoreSystem"]:WaitForChild("ProjectileCore"));
```

Require on the server and each client. Register gameplay settings on the server and matching visual names/templates on clients. Definitions are runtime-local; registering a client template does not configure the server's collision shape.

`Authority = "Server"` makes server simulation the gameplay truth. Its client replicas are presentation, not damage authorities. `Authority = "Client"` is for local effects/previews. Spawn authority defaults to the executing side. Setting a local table to `Authority = "Server"` does not send an authorized spawn command to the server.

The root exports Luau types and also exposes `ProjectileCore["Types"]`, the shared Types module. Use direct-require type aliases in ordinary consumers. Registration factories running during root initialization must require `Shared.Types` instead of requiring the root again; see [the client registration example](./README.md#client-visual-registration). No dependency Ouroboros, thank you.

Types describe contracts; they do not replace runtime validation. Some diagnostic/configuration interfaces remain dynamic. Method names are case-sensitive. Call module APIs with brackets/dot-call semantics; handle methods use colons.

## Public API Index

These are the current public functions. `nil` below means no success result is promised; do not treat a mutator's return as an acknowledgement unless its signature explicitly returns a boolean. Invalid creation/configuration requests can assert. Many mutations quietly ignore missing/ended IDs.

```text
RegisterProjectile(Name, Definition?) -> RegisteredProjectileDefinition
PreloadProjectile(Name, Options?) -> PreparationResult
GetProjectileCacheStats(Name) -> ProjectileCacheStats?
SpawnSimulated(Name, SpawnInformation) -> string, SimulatedProjectileEntry
SpawnPhysical(Projectile, SpawnInformation) -> string, PhysicalProjectileEntry
SpawnBurst(Name, BurstInformation) -> BurstHandle
CreateProjectileGroup(GroupInformation?) -> ProjectileGroupHandle
TransitionProjectile(Identifier, Information) -> boolean, string?
GetProjectilePhase(Identifier) -> ProjectilePhaseStatus?
SetProjectileCurves(Identifier, Curves?, ShouldBroadcast?) -> boolean, string?
ResetProjectileCurves(Identifier, ShouldBroadcast?) -> boolean, string?
GetCurveValue(IdentifierOrEntry, Name) -> number?
GetProjectileCurveState(IdentifierOrEntry) -> ProjectileCurveState?
GetImpactContext(Entry) -> ProjectileImpactContext?
GetImpactCFrame(Context, Placement?) -> CFrame
AttachImpact(Context, Template, Options?) -> ImpactHandle?, string?
SetDeflectionPartConfig(PartOrModel, Config) -> nil
GetDeflectionPartConfig(PartOrModel) -> DeflectionPartConfig?
ClearDeflectionPartConfig(PartOrModel) -> nil
Deflect(Identifier, Parameters, ShouldBroadcast?) -> nil
DeflectArea(Parameters) -> { BasePart }
AttractArea(Information) -> { ProjectileEntry }
CreateAreaController(Information) -> AreaControllerHandle
Redirect(Identifier, Velocity, Position?, Acceleration?, IgnoreList?, ShouldBroadcast?) -> nil
SetHoming(Identifier, HomingData?, ShouldBroadcast?) -> nil
SetOwner(Identifier, Owner?) -> nil
GetOwner(Identifier) -> Instance?
SetControlImmunity(Identifier, ImmunityPatch, ShouldBroadcast?) -> nil
SetTimeScale(Identifier, Scale, ShouldBroadcast?) -> nil
PatchProjectile(Identifier, PatchData, ShouldBroadcast?) -> nil
Cancel(Identifier, Reason?) -> nil
DestroyProjectile(Identifier, Reason?) -> nil
GetData(Identifier) -> ProjectileEntry?
GetAll() -> { [string] : ProjectileEntry }
IsActive(Identifier) -> boolean
GetOwnerFromPart(Part) -> Instance?
SetNetworkTransport(Mode?) -> nil
GetNetworkTransport() -> table
GetNetworkInfo() -> table
SetPerformanceProfilingEnabled(Enabled, Options?) -> nil
GetPerformanceStats() -> table
SaveScriptProfilingData(FileName?) -> boolean, any
ConfigureVisualBudget(Options?) -> nil
SetCollisionMode(Mode, Options?) -> nil
BakeCollisionMap(Options?) -> table
ClearCollisionMap() -> nil
GetCollisionInfo() -> table
RegisterDynamicCollider(Collider) -> DynamicColliderHandle
UnregisterDynamicCollider(Collider) -> nil
ConfigureProjectileCollisions(Options?) -> nil
ConfigureParallel(Mode, Options?) -> nil
DetectHumanoidsNearby(Origin, Range, IgnoreList, Owner?, Visualizer, IgnoreForceFields, IgnoreFriendly?, PartLimit?, VisualizerProperties?, UseRobloxQuery?) -> boolean, { Humanoid }
```

`DestroyProjectile` aliases `Cancel`; `GetNetworkInfo` aliases `GetNetworkTransport`. `Types` is a module/type surface, not a spawn method. Underscored fields, worker buffers, numeric network IDs, serialized control snapshots, and recovery remotes are internal plumbing, not another gameplay API.

## Registration And Spawn Settings

### RegisterProjectile

`RegisterProjectile(Name : string, Definition? : ProjectileDefinition)` registers/replaces defaults on the current side and returns the resolved definition. A previously unknown name is also registered with defaults when spawned. Register explicitly anyway: typos should not be your content pipeline.

Presets under `Objects.Projectiles.<Name>` can supply a BasePart, Model, or ModuleScript-based projectile/cache record. A Model needs a resolvable BasePart, preferably an intentional PrimaryPart. Use separate client visual templates, not a live character assembly.

Definition and spawn settings are resolved per field; this is not a universal shallow `table.clone`. Explicit boolean false is respected where supported. Nested projectile settings, immunity, replication policy, and clash settings have their own merge paths. Avoid cyclic configuration/UserData tables. A spawn override does not rewrite the registered definition.

### SpawnSimulated

`SpawnSimulated(Name : string, SpawnInformation : SimulatedSpawnInfo)` returns an ID and its entry. Give `Velocity : Vector3`, or `Direction : Vector3` plus optional `Speed`. Position and vectors must be finite. `Direction` is normalized; a near-zero direction falls back to local/world forward `(0, 0, -1)`.

| Setting | Default / meaning |
| --- | --- |
| `Position`, alias `StartingPosition` | Vector3.zero if omitted. Supply an actual muzzle position rather than relying on this. |
| `Velocity` | Takes precedence over Direction/Speed; required unless Direction is provided. |
| `Speed` | 125 when resolving Direction; not a replacement for an omitted Direction/Velocity. |
| `Acceleration` | Vector3.zero; studs/second squared. Gravity is opt-in. |
| `Creator` | Resolved current/original owner. A Player resolves to its Character when available. |
| `IgnoreList` | Flat Instance array, copied; resolved owner is also excluded. Nested arrays are not flattened. |
| `Authority` | Executing side, Server or Client. |
| `Group` | Optional runtime-local group handle. Never replicated. |
| `UserData` | Dynamic ability data; see replication/privacy below. |
| `HomingData` | Optional homing configuration. |
| `TravelMode`, `TravelData` | Linear by default; Bezier settings below. |
| `ProjectileIdentifier` | Optional public ID; an already-active ID returns that existing entry, not a second independent shot. Omit for normal spawning; bursts prohibit it. |
| `SkipReplication` | Optional suppression for a server spawn; not a client permission switch. |

Common definition/spawn settings:

| Setting | Default / contract |
| --- | --- |
| `ProjectileTemplate`, `ProjectileCache` | Optional visual template/custom cache. Standard Core cache is preferred for preparation/stats. |
| `ProjectileCacheWarmCount` | 10. Configures warm minimum, not an active-projectile cap. |
| `ProjectileCacheMaxSize` | Idle retention defaults to max(64, warm count); explicit limits cannot be below warm minimum. |
| `CacheWarmMode` | Definition setting: Immediate; Background registers immediately and warms over later Heartbeats. |
| `CreateProjectilePart` | true. false skips the visual, not gameplay simulation. |
| `ProjectilePartParent` | Optional visual container. Do not use an assembly that would drag gameplay objects. |
| `PrepareProjectilePart`, `OnPartEnd`, `EffectPolicy` | Visual initialization, retention, and managed effects. |
| `TimeScale` | 1; simulated values must be nonnegative. |
| `CollisionEnabled` | true. false disables world contacts, close-hit probing, and projectile clashes while motion/lifetime continue. |
| `CollisionMode` | Inherit backend; backend starts Native. |
| `SimulationStepMode` | Legacy. Auto permits the eligible optimized linear path; it is not Actor dispatch. |
| `AllowEmptySpaceSkip` | Opt-in override, otherwise backend setting; requires verified baked coverage. |
| `SpherecastRadius`, `BlockcastSize`, `UseShapecast` | No sphere/block volume, false shapecast. Only one world volume may be active. |
| `HumanoidSpherecastRadius` | nil/0: ordinary world cast behavior. Positive enables the separate living-character sweep. |
| `IgnoreWater` | false. |
| `Bounce`, `ProjectileCollision`, `Curves` | Opt-in configurations below. |
| `AngularSpin`, `SpinStrength`, `SpinDirection` | false, 0 degrees/second, absent local-forward axis. |
| `AllowRaycastDeflection`, `MaxDeflections` | true, 24. |
| `ControlImmunity` | Empty by default; definition plus spawn overrides. |
| `ReplicateToClients` | true for server spawns unless suppressed. |
| `ReplicateHitInfo` | false; additional impact metadata is opt-in. |
| `ReplicateUserData`, `ReplicatedUserData` | true, nil; sanitized UserData is the default simulated sidecar. |
| `ReplicationSyncInterval` | 0.08 seconds; 0 disables ordinary periodic correction. Forced lifecycle corrections still exist. |
| `ReplicationSmoothTime`, `ReplicationSnapDistance` | 0.1 seconds, 60 studs. |
| `ReplicationPolicy` | Automatic relevance, owner always relevant, instant-hit visuals enabled. |
| `AllowVisualBudget` | false; client particle/light distance budgeting opt-in. |

`ProjectileSettings` defaults: `MaxFlyTime = 10`, `MaxFlyDistance = 5000`, `RaysPerMove = 8`, `MaxPiercesPerStep = 8`. It also accepts `CollisionMode`, `CollisionEnabled`, `AllowEmptySpaceSkip`, `BlockcastSize`, and close-hit fields. Prefer the documented top-level collision/snap settings for spawn/patch overrides; inspect field precedence rather than supplying contradictory copies in several places. Total flight limits are not reset by phases or curve resets.

## World Collision And Bouncing

### Cast Volumes

Rays are the default. Choose **one** active world-volume setting: `SpherecastRadius > 0`, positive `BlockcastSize : Vector3`, or `UseShapecast = true`. Conflicting final settings assert at registration/spawn/patch. Sphere radius and block components are validated up to 512; block components must be positive. Shapecast needs usable geometry. Block orientation follows movement-facing cast state.

`HumanoidSpherecastRadius` is a separate positive sphere sweep for living characters, not a second world-volume selector. For example, world sphere radius 5 plus humanoid radius 15 allows generous character hits without hitting a wall with the 15-stud sphere. Dead-character geometry uses the ordinary world path. This feature is incompatible with a custom `RaycastFunction`; it does not add stationary overlap detection.

`CollisionEnabled = false` still advances movement, controls, age/distance, and lifetime. Ability-owned hitboxes are unaffected. Re-enable from the current position; the disabled path is not swept retroactively. Do not describe this as collision immunity against every external damage system.

For phasing through ordinary hits, `CanPierce(Entry, SwiftProjectile, HitResult) -> true` continues the cast. Shield deflection is considered before piercing. Use a separate server hitbox for repeat damage if that is your intended mechanic; do not abuse terminal-hit callbacks as an overlap loop.

### Close-Hit Snap

`SnapHitEnabled = true`, `SnapHitDistance = 12`, `SnapHitTime = 0.05` are present. A one-time early probe uses the smaller distance/time window to resolve an immediately terminal hit before visible travel. A miss does not advance the projectile or fire a fake callback. Existing sphere/shapecast, deflection and normal hit cleanup paths remain relevant.

Root spawn resolution disables the inherited snap default when a custom `CanPierce` exists. An explicit spawn `SnapHitEnabled` (including its ProjectileSettings override) takes precedence; forcing true means accepting that callback/probe interaction. Raw SwiftCast remains conservative unless configured through Core. Snap is simulated-only and is not a general hitscan gun implementation.

### Bounce

Provide `Bounce = { ["Enabled"] = true; ... }` on a simulated definition/spawn/patch:

| Field | Default |
| --- | --- |
| `MaximumBounces` | 4 |
| `Restitution` | 0.65 |
| `SurfaceFriction` | 0.1 |
| `MinimumBounceSpeed` | 12 |
| `MaximumBouncesPerStep` | 2 |
| `SeparationDistance` | 0.03 |
| `MaterialRestitution` | Optional Enum.Material-to-number overrides |
| `CanBounce` | Optional synchronous `(Entry, HitResult, IncomingVelocity) -> boolean` |
| `OnBounce` | Optional protected notification `(Entry, HitResult, NewVelocity, BounceCount)` |

Terrain/non-humanoid geometry is eligible by default; humanoids need explicit CanBounce approval. World contact precedence is deflection, piercing, then bounce/terminal hit. Count, speed, progress, and corner safeguards prevent an eternal vibrating grenade. Successful bounces are not terminal impacts. Physical projectiles retain Roblox-native collision/bounce behavior.

## Callbacks And Terminal Lifetime

| Callback | Arguments / role |
| --- | --- |
| `PrepareProjectilePart` | `(Entry, ProjectilePart, Context)`; Context has Position, Authority, ProjectileName. Initial placement is valid before this runs. Keep it fast and non-yielding. |
| `OnServerHit`, `OnClientHit` | `(Entry, HitResult?, Reason, ImpactContext?)`; terminal Hit notification on the relevant side. |
| `OnServerDestroy`, `OnClientDestroy` | `(Entry, Reason, ImpactContext?)`; terminal reason may be cancellation, timeout, clash, or other cleanup, not just a world hit. |
| Spawn `OnDestroy` | Entry-level `(Entry, Reason, ImpactContext?)` notification in addition to side-specific destruction. |
| `OnPositionChange` | Simulated `(Entry, Position, LastPosition)`; hot movement notification, not a place to preload or enumerate the entire world. |
| `CanPierce` | `(Entry, SwiftProjectile, HitResult) -> boolean`; synchronous collision decision. |
| `RaycastFunction` | `(Entry, Origin, Displacement, RaycastParams) -> RaycastResult?`; synchronous replacement query. |
| `OnDeflect` | `(Entry, Deflector?)`; notification, not permission to choose a different committed collision. |
| Spawn `OnOwnerChanged` | `(Entry, NewOwner?)`. Actual owner changes detach group membership before notification. |
| `OnPartEnd` | `(ProjectilePart, Reason, Cleanup, ImpactContext?)`; retain the leased visual for a deliberate fade, then call Cleanup. |

Decision hooks must not yield: this includes piercing, raycast, bounce/clash eligibility, and allegiance predicates. Protected terminal notifications run independently from mandatory cleanup. Hit is invoked before destroy in that notification chain, but active registry/cache ownership does not wait for a yielded user callback. An entry retained by a callback is not an active visual lease.

`Cleanup()` is idempotent. Core forces forgotten `OnPartEnd` cleanup after the maximum eight-second grace; this is a safety net, not your effect lifetime. Stale warmup/deferred work is guarded against recycled leases. Do not keep moving a projectile Part from a terminal callback after it may have returned to cache.

On replicated server hits, **`Reason == "Hit"` does not guarantee a non-nil HitResult**. `ReplicateHitInfo` defaults false. For authoritative contact position/normal/material/local target placement, opt in on the server, then prefer `ImpactContext`. A reconstructed client hit record is not a guarantee of a native RaycastResult userdata or an available streamed Instance.

```lua
local function OnClientHit(Entry : ProjectileCore.SimulatedProjectileEntry, HitResult : RaycastResult?, Reason : string, Context : ProjectileCore.ProjectileImpactContext?) : ()
	if Reason ~= "Hit" then return; end;

	local Position = if Context then Context["Position"] elseif HitResult then HitResult["Position"] else Entry["Simulated"]["Position"];
	if typeof(Position) ~= "Vector3" then return; end;

	--// Ability-owned impact effects belong here; damage belongs on the server.
end;
```

The tracked position fallback above is not necessarily the surface contact point of a volume cast. Use the optional metadata when that distinction matters.

## Preparation And Cache Ownership

`PreloadProjectile(Name, Options?) -> PreparationResult` yields until its shared preparation job finishes or fails. The name must already be registered **on this side** with a Core visual cache/template. Missing registrations/templates/custom unsupported caches return a result with `CacheReady = false` and `CacheError`; they do not create an active shot.

Options: `WarmCount` defaults to the registered warm count; `PreloadAssets` defaults true, but only clients request asset loading. Warm toward a minimum **available** count, coalesce concurrent jobs, and obey retention. The shared clone step uses a 1ms/four-object soft frame budget; one clone itself cannot be interrupted.

| Result fields | Meaning |
| --- | --- |
| `ProjectileName`, `CacheReady`, `CacheError`, `Available` | Cache identity/readiness and idle availability. |
| `MetadataReady`, `MetadataAvailable` | Prepared visual metadata availability. |
| `AssetsRequested`, `AssetsReady`, `AssetFailures`, `AssetError` | Separate client asset outcome; failures contain Asset and Enum.AssetFetchStatus. Server cache readiness does not mean clients loaded assets. |
| `RequestedWarmCount`, `TargetWarmCount`, `LimitedByRetention`, `ElapsedSeconds` | Effective target/cap and elapsed wall time. |

`GetProjectileCacheStats(Name)` returns nil without a Core cache; otherwise a snapshot with Available, InUse, PendingWarmCount, RetentionLimit, EmptyCacheCreations, TotalCreated/Destroyed, HighWaterCreated, MaxSize, MetadataAvailable, CreationSeconds, and MaximumCreationSeconds. Do not confuse retained idle objects with leaked active leases.

Idle retention is not an active-shot cap. Surplus idle visuals become eligible for trimming after ten seconds through shared bounded maintenance. Ordinary firing can proceed during background warming and may create cache misses. Preparation does not reserve a position, start age/lifetime, enable effects, or send a spawn packet. Run it in loading/ability preparation, not receive/movement/hit handlers.

## Bursts And Patterns

`SpawnBurst(Name, Information)` emits ordinary **simulated** projectiles through normal spawn/collision/replication. It does not make extra shots free, send one compact pattern packet, or manage already-fired shots after cancellation.

| Field | Default / contract |
| --- | --- |
| `Origin` | Required CFrame, BasePart, or Attachment. Generated shots start at its sampled world position. |
| `FollowOrigin` | false snapshots origin at call time; true samples the live source before each volley. Removed followed origins cancel pending firing. |
| `SpawnInformation` | Shared SimulatedSpawnInfo, including optional Group. |
| `Pattern` | Type Fan, Count 1, SpreadAngle 0, PhaseAngle 0, optional signed 32-bit Seed. |
| `Volleys`, `VolleyInterval`, `Delay` | 1, 0, 0. Multiple volleys require a positive finite interval. |
| `ShotOverrides` | Indexed spawn override tables, reused by shot index across volleys. |
| `PrepareShot` | Synchronous `(ShotInformation, Context) -> overrides, false, or nil`; false skips, nil keeps it. Errors/yields stop pending firing. |

Count is 1-256 per volley; requested total is at most 4,096. SpreadAngle is the **total** spread, not the gap between shots. Fan evenly centers yaw directions on local forward, including even counts. Ring spans the local XZ plane without a duplicate closing sample. Cone samples uniform solid angle with full opening angle 0-180 degrees. Sphere uses Fibonacci directions with seeded orientation. PhaseAngle rotates around local up; seed plus volley index makes repeatable variation.

Settings apply shared info, generated position/direction, indexed override, then hook override. Generated direction removes shared Velocity; explicit per-shot Velocity remains supported. Context contains `Index` (global requested emission index), `VolleyIndex`, `ShotIndex`, sampled `Origin`, and generated `Direction`. Caller settings are copied. IDs/network IDs are prohibited in burst settings, and an assigned burst cannot switch Group through shot overrides/hooks.

Zero-delay first volleys run synchronously. Later volleys share a lazy scheduler: at most four whole scheduled volleys per Heartbeat, one per burst, under a 1ms **soft** inter-volley budget. A whole volley can exceed that budget. Missed intervals do not dump several catch-up volleys into one frame.

| BurstHandle method | Result |
| --- | --- |
| `Cancel(Reason?)` | Idempotent pending-only cancellation; emitted projectiles continue. |
| `IsActive()` | Whether firing work remains. |
| `GetStatus()` | Copy: State, Emitted, Skipped, Requested, VolleysCompleted, Reason, Error. State is Scheduled/Firing/Completed/Cancelled/Failed. |
| `GetIdentifiers()` | Copied successful shot IDs in emission order, including IDs whose shots may already have ended. |

### Centered Triple Spread

Assumes ProjectileCore, Character and a live `Muzzle : Attachment` are available from trusted server ability code:

```lua
local Burst = ProjectileCore["SpawnBurst"]("Rocket", {
	["Origin"] = Muzzle;

	["Pattern"] = {
		["Type"]        = "Fan";
		["Count"]       = 3;
		["SpreadAngle"] = 24;
	};

	["SpawnInformation"] = {
		["Speed"]                = 180;
		["Creator"]              = Character;
		["Authority"]            = "Server";
		["CreateProjectilePart"] = false;
		["IgnoreList"]           = { Character; };
	};
});
```

This gives -12, 0, +12 degrees. A manual loop is still valid for custom attacks; SpawnBurst is an authoring/scheduling helper, not a mandatory tax.

## Projectile Groups

`CreateProjectileGroup({ Name?, OnFinished? }?)` returns a runtime-local handle for one attack. Members can be simulated or registered physical entries, with one group per projectile. Ownership changes detach members synchronously; same-owner assignments retain membership. A stolen projectile does not remain controllable by its original attack group.

| Method | Contract |
| --- | --- |
| `Add(Identifier)`, `Remove(Identifier)` | boolean, optional rejection reason. Add requires active eligible entry/open group; duplicate same-group Add is harmless. Remove detaches, not cancels. |
| `Contains(Identifier)`, `GetCount()` | Current identity-checked membership, not a spawn history. |
| `GetIdentifiers()` | Copied active member IDs; ordering unspecified. |
| `Seal()` | Reject manual additions/new burst assignments. Existing burst reservations may finish emitting. |
| `Cancel(Reason?)`, `Destroy(Reason?)` | Close admissions, cancel assigned pending bursts, cancel members. Destroy aliases Cancel, not detach-all. |
| `IsActive()`, `GetStatus()` | Open/Sealed remain active. Status copy contains Name, State, MemberCount, PendingBursts, Reason. |
| `SetTimeScale(Scale, ShouldBroadcast?)`, `SetControlImmunity(Patch, ShouldBroadcast?)` | Bulk calls across both projectile types. Group scale validation requires finite nonnegative values. |
| `SetHoming(Data?, ShouldBroadcast?)`, `Redirect(Velocity, ShouldBroadcast?)`, `Patch(Data, ShouldBroadcast?)`, `Transition(Information)` | Simulated-only bulk operations; physical members are skipped. Redirect preserves each member's position/acceleration/filters. |

Bulk results contain `Attempted`, `Skipped`, `Failed`, `FirstError`; attempted calls are not guaranteed state changes. Arguments are validated first. Commands snapshot membership and recheck activity/identity; additions during a command are excluded and individual failures do not abort unrelated members.

Completion means **Sealed + no members + no pending bursts**. Empty Open groups remain usable. `OnFinished(Status)` is protected and runs once for logical terminal state, independently of retained fades/yielded notifications. Groups create no per-member polling loop, are not replicated identities, and cannot enroll server-authoritative client mirrors for local control. Death/unequip cancellation remains ability-owned.

### Grouped Ring And Cleanup

Assumes ProjectileCore, Character, and Muzzle from the previous example:

```lua
local Group = ProjectileCore["CreateProjectileGroup"]({ ["Name"] = "BladeRing"; });

ProjectileCore["SpawnBurst"]("Blade", {
	["Origin"]         = Muzzle;
	["FollowOrigin"]   = true;
	["Volleys"]        = 3;
	["VolleyInterval"] = 0.2;

	["Pattern"] = {
		["Type"]  = "Ring";
		["Count"] = 12;
	};

	["SpawnInformation"] = {
		["Group"]                = Group;
		["Speed"]                = 120;
		["Creator"]              = Character;
		["Authority"]            = "Server";
		["CreateProjectilePart"] = false;
	};
});

Group:Seal();

--// In the ability's death/unequip cleanup, if it should cancel this attack:

local function StopAttack() : ()
	Group:Destroy("AbilityEnded");
end;
```

The already-assigned burst may finish after Seal. Destroy stops both those pending emissions and current members. Keep the cleanup function in the ability's existing lifecycle owner; this snippet deliberately does not invent another connection manager.

## Movement, Ownership, And Patching

These functions operate on an active ID. Call server gameplay mutations on the server and local cosmetic mutations locally; editing a server-authoritative mirror is not an authoritative command.

| Function | Parameters, result, and limits |
| --- | --- |
| `Redirect` | Required new Velocity; optional Position, Acceleration, IgnoreList preserve current tracked values. Simulated-only, no success result. Recasts/reconciles motion and interrupts a pending authored phase notification. Very-near unchanged position/velocity can take an update-only shortcut. |
| `SetHoming` | HomingData or nil to clear; simulated-only. Does **not** replace Bezier travel. Use TransitionProjectile for an explicit Bezier-to-homing change. Interrupts the authored phase notification. |
| `SetTimeScale` | Finite scale; simulated values below 0 are ignored, 0 pauses, 1 is normal. Physical values go through the physical adapter, not the simulated validation path. |
| `SetOwner` | New owner Instance, or nil to clear. Uses owner resolution; actual changes detach groups and interrupt phases. No automatic reattachment when ownership later returns. |
| `GetOwner` | Current resolved owner Instance or nil. |
| `SetControlImmunity` | Boolean immunity patch; supports both projectile types and refreshes captures. false removes that key's protection. |
| `PatchProjectile` | Supported simulated fields only, validated before application. It does not replace identity, ownership, template, cache, callbacks, lifetime limits, or group membership. |
| `Cancel`, `DestroyProjectile` | Both types; optional reason defaults Cancelled. Idempotent for an absent/ended entry. Normal terminal notifications/visual release still run. |

Where exposed, `ShouldBroadcast` defaults to replication unless explicitly false and is relevant to replicated simulated state. It does not make physical state travel through a new simulated lifecycle system. Prefer owner/control APIs over mutating entry internals.

Supported patch keys are `Velocity`, `Acceleration`, `HomingData`, `IgnoreList`, `IgnoreWater`, `SpherecastRadius`, `HumanoidSpherecastRadius`, `BlockcastSize`, `UseShapecast`, `Bounce`, `ProjectileCollision`, `CollisionEnabled`, `CollisionMode`, `SimulationStepMode`, `AllowEmptySpaceSkip`, `TimeScale`, `AngularSpin`, `SpinStrength`, `SpinDirection`, the three SnapHit fields, `EffectPolicy`, `EffectsEnabled`, `ReplicationPolicy`, `ReplicationSyncInterval`, `ReplicationSmoothTime`, `ReplicationSnapDistance`, `UserData`, `ReplicatedUserData`, `ReplicateUserData`, `ReplicateHitInfo`, `ControlImmunity`, `Curves`, and `AllowVisualBudget`.

Lua tables do not carry a nil-valued key. Assigning nil in a patch is not a universal clear operation; use dedicated clearing APIs where offered. Sphere-to-block switches can disable SpherecastRadius with 0 in the same validated patch. Do not assume a nullable BlockcastSize or template can be cleared by an omitted field. Use a new configured shot when no supported clear path exists.

## Safe Phase Transitions

`TransitionProjectile(Identifier, Information) -> boolean, reason?` is the explicit atomic movement switch. `GetProjectilePhase(Identifier)` returns a copied status or nil. Both are simulated-only; transitions reject missing/terminal entries, pending casts, server mirrors on clients, unsafe decision hooks, and stale expected revisions before committing.

| Information field | Contract |
| --- | --- |
| `Mode` | Required Linear, Homing, QuadraticBezier, or CubicBezier. |
| `Name` | Optional diagnostic phase name. |
| `Velocity` | Omit to preserve effective current momentum; explicit value replaces it. |
| `Acceleration` | Omit to preserve base acceleration rather than superseded field contributions. |
| `HomingData` | Required valid homing settings for Homing; incompatible with other modes. |
| `TravelData` | Required valid travel target/settings for Bezier; curve starts at current tracked position. Incompatible with Linear/Homing. |
| `CollisionEnabled` | Optional new world/clash collision state. |
| `ReleaseControllers` | Explicit live Motion/Force controller handles to release; not an ID list or every controller in the world. |
| `Duration`, `OnPhaseComplete` | Optional finite **positive** effective-time timer, with protected `(Identifier, Status)` notification. Callback requires Duration. |
| `ExpectedRevision` | Optional optimistic check against current phase Revision. |

Preserves position, public/network identity, owner, group membership, total age/distance limits, visuals, effects, and angular spin. Does not rerun preparation, close-hit snap, spawn, or destroy notifications. Superseded homing/travel is reconciled and stale worker/collision results invalidated; Wind momentum is merged once rather than duplicated.

Named released controllers cannot recapture that entry until those controllers end. Other controls, one-shot attraction, immunity, and FrozenMoment remain. Release must not overwrite the new velocity with an old capture velocity. This is an explicit release list, not an "ignore every blackhole forever" option.

Status: `Name`, `Revision`, `Mode`, `Elapsed`, `Duration`, `TimerCompleted`, `Interrupted`. Duration expiry reports completion **once** but does not stop movement or auto-transition. Slowdown affects the clock; frozen effective time pauses it. Replacement, actual ownership change, deflection, and base-motion-changing APIs invalidate the pending notification. External field influence alone does not.

### Orbit To Launch

Assume `Identifier` was spawned with CollisionEnabled false, `OrbitController` is its live controller, and `LaunchVelocity` comes from trusted ability code:

```lua
local Applied, Reason = ProjectileCore["TransitionProjectile"](Identifier, {
	["Name"]               = "Launched";
	["Mode"]               = "Linear";
	["Velocity"]           = LaunchVelocity;
	["Acceleration"]       = Vector3.zero;
	["CollisionEnabled"]   = true;
	["ReleaseControllers"] = { OrbitController; };
});

if not Applied then
	warn("Launch rejected: " .. tostring(Reason));
end;
```

Call from the ability's existing release/input/timing logic. Keep ownership of the controller and destroy it during ability cleanup. A phase is not an automatic sequencer; Group:Transition applies the same request independently to each current simulated member.

## Homing And Bezier Travel

HomingData supports `Enabled`, Instance `Target` (legacy `HomingInstance`), `TargetTag`, Vector3 `TargetPosition`, local/world target `Offset`, `TurnSpeed`, `MinimumDistance`, `AutoAcquire`, `AutoAcquireRange`, `AutoAcquireInterval`, `AutoAcquireRetarget`, `RequireLineOfSight`, `AutoAcquireValidator`, and `DebugVisualizerInterval`.

Direct homing uses TurnSpeed 10 and MinimumDistance 0 when not supplied. Auto acquisition is opt-in, samples at 0.1 seconds by default, and needs an explicit useful range (omitted range resolves to 0). Provide the synchronous `AutoAcquireValidator(Humanoid, Character) -> boolean` for your game's living/enemy/ForceField rules. Do not assume a target selector knows your entire faction system. Acquisition scheduling spreads eligible refresh work; it does not make expensive custom validators free.

Bezier TravelData accepts `Target`/`End` (Vector3, CFrame, BasePart, Model, Attachment), `EndPosition`, `TargetOffset`, `TrackTarget`, `ControlPoint` (quadratic), `ControlPointA`/`ControlPointB` (cubic), `Duration`, `Speed`, `OrientationMode`, `CompletionMode`, `HomingTurnSpeed`, `HomingMinimumDistance`, `ContinueDuration`, and `DestroyOnComplete`. Author Duration when timing matters rather than relying on derived distance/speed timing.

CompletionMode is Linear, Destroy, Homing, or ContinueBezier. A travel-complete destruction is not necessarily a world Hit. Tracking a moving target changes the evaluated endpoint; deflection and explicit phase changes clear superseded travel/homing state consistently. SetHoming alone does not erase a still-active Bezier.

For per-shot homing patterns, use a non-yielding PrepareShot override and normal HomingData. Origin/direction are already generated; return only the intended override:

```lua
local function PrepareHomingShot(Information : ProjectileCore.SimulatedSpawnInfo, Context : ProjectileCore.BurstShotContext) : ProjectileCore.SimulatedSpawnInfo?
	if not TargetPart:IsDescendantOf(workspace) then return nil; end;

	return {
		["HomingData"] = {
			["Enabled"]         = true;
			["Target"]          = TargetPart;
			["TurnSpeed"]       = 10;
			["MinimumDistance"] = 2;
		};
	};
end;
```

Here TargetPart is an ability-selected BasePart and the function is assigned to BurstInformation.PrepareShot. Nil keeps the normal generated shot; return false instead if your ability should skip a missing target.

## Deflection And Shields

### Direct And Area Deflection

`Deflect(Identifier, Parameters, ShouldBroadcast?)` accepts simulated/registered physical IDs and returns no success value at the root API. Parameters include explicit `Velocity`, `Normal`, `Direction`, `Position`, `Acceleration`, replacement `IgnoreList`, `Deflector`, `ChangeOwner`, `ReflectionMode`, `CenterPosition`, `OwnerSearchRange`, `SpeedMultiplier`, `MaxDeflectSpeed`, `SpreadAngle`, `ReflectionBias`, `DeflectCooldown`, `LocateTargetValidator`, and `IsTeammate`.

ReflectionMode supports Reflect, Return, Locate, Lob. Direct simulated defaults are Reflect, search range 500, speed multiplier 1.15, spread 0, reflection bias 0.5. The physical direct wrapper defaults to Return; set the mode explicitly for cross-type consistency. Physical options also include RespectDeflectable, RequireTouch, IgnoreSet, Blacklist. Deflection respects immunity, count limits, and cooldowns; root returning nil does not mean a redirect succeeded.

`DeflectArea(Parameters) -> { BasePart }` adds area geometry, IgnoreList, Deflector/Character/Owner/Tool references, AreaGroup/Priority/LockDuration, optional LeadTime/UseIncomingFilter, ShouldBroadcast, debug options, and `DeflectFunction(Part?, Entry?, Position)`. That callback runs after each successful deflection; a server-invisible simulated entry can have nil Part. The returned array contains existing parts only, so it is **not** a count of every successful partless deflection. The physical area path can consider eligible physical parts under its existing deflectability rules.

`AreaGroup` locks prevent repeated overlapping operations from replacing each other indiscriminately. Their priority/lock rules are separate from projectile groups. Do not confuse "these twelve blades belong to my Group" with "this field has AreaGroup Blackhole".

### Configurable Surfaces

`SetDeflectionPartConfig(PartOrModel, Config)` and `ClearDeflectionPartConfig(PartOrModel)` are **server-only**. Setting replaces the config, getting returns a copy or nil. Reliable updates/recovery distribute serializable surface settings; authoritative deflection outcomes still use the ordinary lifecycle path.

| Config field | Default / behavior |
| --- | --- |
| `Enabled` | true. Explicit false suppresses surface deflection; clearing restores normal CanDeflect attribute behavior. |
| `Mode` | Reflect: collision normal; Direction: supplied world vector; Facing: current surface orientation transforms LocalDirection; Return: aim at current incoming owner, else reflect. |
| `Direction` | Required finite nonzero vector for Direction mode. |
| `LocalDirection` | `(0, 0, -1)`; used by Facing/front-face logic. |
| `SpeedMultiplier`, `MaximumSpeed` | 1, optional speed cap. |
| `TwoSided` | true. false accepts front-side incoming motion only; rejected back contacts continue normal hit/pierce handling. |
| `Owner` | Optional explicit ownership transfer. Configured surfaces otherwise preserve ownership. |

Configured surfaces need no separate CanDeflect attribute. Direct part configuration takes precedence over ancestor Model configuration. Simulated contacts and registered physical autocast share resolution; no automatic Touched listener/polling loop is installed per shield. Existing unconfigured CanDeflect surfaces retain their legacy behavior. Shield deflection precedes piercing/bouncing and does not bypass immunity or deflection limits.

Assume ShieldPart/RedirectPart are server-owned BaseParts:

```lua
ProjectileCore["SetDeflectionPartConfig"](ShieldPart, {
	["Mode"]            = "Reflect";
	["TwoSided"]        = false;
	["SpeedMultiplier"] = 1;
});

--//

ProjectileCore["SetDeflectionPartConfig"](RedirectPart, {
	["Mode"]           = "Direction";
	["Direction"]      = Vector3.new(0, 1, -1)["Unit"];
	["MaximumSpeed"]   = 500;
});
```

## Area Controllers

`CreateAreaController(Information) -> AreaControllerHandle` creates a persistent field, normally on the server. Clients run broadcast predictive shadows for simulated visuals; they do not replace authoritative gameplay. Defaults affect simulated and physical projectiles, priority 0, broadcast enabled. Lifetime omitted/zero persists until destroyed. Fields do not automatically belong to a projectile Group.

| Information | Contract |
| --- | --- |
| `Mode` | Orbit, Tornado, Attract, Wind, FrozenMoment. Orbit is the default; BehaviorMode is a legacy alias. |
| `Center`, `CenterPosition` | Vector3/CFrame or live BasePart/Model/Attachment center; moving sources are sampled. |
| `Area` | Capture geometry; independent from motion distribution Shape. |
| `Behavior` | Lifetime, ReleaseMode, FacePath, FacePathTime, slowdown entry/exit fields. |
| `Motion` | Orbit/tornado distribution, Attract parameters, or Wind direction/speed/response settings. |
| `Shape` | Orbit/tornado motion distribution: Sphere, Box, Cylinder, Disc with ShapePartial, ShapeStyle, ShapeInOut. |
| `AreaPriority`, `AreaGroup` | Lane priority and diagnostic/grouping context. |
| `AffectSimulated`, `AffectPhysical`, `ShouldBroadcast` | true unless explicitly false. Physical capture is server-side. |
| Debug options | Explicit opt-in area visualization, color/lifetime/transparency/AlwaysOnTop. |

Area supports Sphere, Box, Cylinder, Disc, Cone, Pyramid, Capsule, HalfSphere through Shape/AreaShape, Center, CFrame/AreaCFrame, Range/Radius, Size, Height, Thickness, Direction/Normal/Axis. Disc is a **thin cylinder**, not an infinite plane. Supply geometry/orientation intentionally; a cone is not just a sphere with an ambitious name.

| Lane | Modes / overlap behavior |
| --- | --- |
| Motion | Orbit, Tornado, persistent Attract compete by priority. |
| Force | Wind competes with other Wind fields; Motion control or active one-shot attraction suspends its contribution. |
| Time | FrozenMoment independently controls effective time. |

Higher AreaPriority wins within a lane; ties retain the existing controller. Discovery uses the spatial index at 10Hz and captured control stepping at 30Hz. Runtime immunity changes can release captures. Broadcast deterministic controls normally use 0.25s periodic corrections; server-only controls needing visual correction retain 0.08s, with reliable capture/release corrections. These intervals are not packet-latency guarantees.

Handle methods: `Destroy(Reason?)`, `Pause()`, `Resume()`, `SetCenter(Center)`, `SetPriority(Priority)`, `GetCaptured()`, `IsActive()`. GetCaptured returns a current array of entries; Destroy releases/stops the field. Pause suspends controller activity, not projectile lifetime or every unrelated influence. Own field cleanup in your ability lifecycle.

### Orbit, Tornado, And Attract

Behavior defaults: `ReleaseMode = "PreserveVelocity"`, `FacePath = true`, `FacePathTime = 0.08`. RestoreOriginal requests original velocity for ordinary Motion releases; active speed-curve reconciliation favors current momentum instead of restoring stale curve-era velocity. FrozenMoment restores captured time scale independently.

Motion supports Axis, Radius (20), Height (24), Cycles (1), HoverHeight (0), SpinSpeed, CorrectionSpeed (12). SpinSpeed is **radians/second**, unlike AngularSpin's degrees/second. Without SpinSpeed, finite lifetime derives speed from Cycles; otherwise it defaults to one revolution/second. Zero Axis enables smooth per-projectile axis drift. Motion values accept numbers or `{ Min | Minimum, Max | Maximum }` tables, rolled per capture; Lifetime rolls once per controller.

Orbit/tornado Shape supports `ShapePartial` (default 1), Surface/Volume ShapeStyle, and Outward/Inward/InAndOut ShapeInOut. Orbit remains mainly circular/radial; Tornado adds vertical swirl. Area controls **which entries are captured**; Shape controls **how they are distributed**.

Persistent Attract uses Motion.Strength (20), Falloff (1.4), and Method VelocityDamp or AccelerationDamp. Prefer one lifetime-controlled field for a sustained vortex.

`AttractArea(Information) -> { ProjectileEntry }` remains the separate pulse API: same family of center/area shapes plus Strength, Falloff, Duration, Method, OneShot, IgnoreList, affected-type flags, priority/group and broadcast options. It selects current eligible entries rather than continuously discovering newcomers. Do not send ten overlapping broadcasts/second merely to impersonate a persistent controller.

### Wind

Motion defaults: `Direction = Vector3.new(0, 0, -1)`, `DirectionSpace = "Local"`, `Speed = 30`, `ResponseTime = 0.35`. Direction must be finite/nonzero, Speed nonnegative, ResponseTime positive. World uses a world vector; Local follows the center orientation, with position-only centers using world axes.

Wind is a bounded additive velocity approaching the requested current, **not another full velocity added every frame**. Ordinary homing/gravity continue; Bezier uses separate displaced path state. Release preserves displacement and reconciles momentum once. FrozenMoment pauses accumulation without saving an impulse to launch later. Physical adapters apply velocity differences without teleporting or duplicating forces.

Example: a server wind field with a separately immune meteor definition:

```lua
ProjectileCore["RegisterProjectile"]("MiniMeteor", {
	["CreateProjectilePart"] = false;
	["SpherecastRadius"]     = 5;

	["ControlImmunity"] = {
		["Wind"] = true;
	};
});

--//

local Wind = ProjectileCore["CreateAreaController"]({
	["Mode"]   = "Wind";
	["Center"] = CFrame.lookAt(StartingPosition, StartingPosition + PushDirection);

	["Area"] = {
		["Shape"] = "Sphere";
		["Range"] = 30;
	};

	["Behavior"] = {
		["Lifetime"] = 1;
	};

	["Motion"] = {
		["DirectionSpace"] = "Local";
		["Direction"]      = Vector3.new(0, 0, -1);
		["Speed"]          = 90;
		["ResponseTime"]   = 0.2;
	};
});
```

StartingPosition and nonzero PushDirection are ability-supplied vectors. The field pushes along one uniform current, not radially away from its center or along a spline.

### FrozenMoment

Behavior `SlowDownMultiplier` defaults 0.5; 0 means normal speed, 1 means stopped (`TimeScale = 1 - multiplier`). Entry/exit easing default 0.35 seconds. These may also be supplied as top-level slowdown fields. Release restores each entry's pre-field time scale, not blindly 1. Fully stationary frozen projectiles skip useless movement casts but remain clash candidates if otherwise eligible.

```lua
local Freeze = ProjectileCore["CreateAreaController"]({
	["Mode"]   = "FrozenMoment";
	["Center"] = FreezePart;

	["Area"] = {
		["Shape"] = "Sphere";
		["Range"] = 45;
	};

	["Behavior"] = {
		["Lifetime"]            = 2.5;
		["SlowDownMultiplier"]  = 1;
		["ProjectileEntryTime"] = 0.35;
		["ProjectileExitTime"]  = 0.35;
	};
});
```

## Control Immunity

Definition immunity merges with spawn immunity; explicit false overrides that key. `SetControlImmunity(Identifier, Patch, ShouldBroadcast?)` updates existing simulated/physical entries and releases newly blocked captures. These protections affect Core control systems, not world hits, damage, cleanup, or external hitboxes.

| Key | Blocks |
| --- | --- |
| `All` | Every listed control/clash response. |
| `Deflect` | Direct and raycast deflection, and area deflection fallback. |
| `DeflectArea` | Area deflection only, in addition to Deflect protection. |
| `Attract` | Attraction operations. |
| `AreaController` | All controller modes. |
| `AreaMotion` | Orbit, Tornado, Wind family motion control. |
| `AreaTime` | FrozenMoment. |
| `Orbit`, `Tornado`, `FrozenMoment`, `Wind` | The named mode. |
| `ProjectileCollision` | Incoming projectile clash responses; Deflect additionally blocks clash deflection. |

false on Deflect does not punch through an independently true All/DeflectArea guard. Physical Deflectable/RespectDeflectable remain backward-compatible and are not replacements for these keys.

```lua
ProjectileCore["RegisterProjectile"]("BossMeteor", {
	["CreateProjectilePart"] = false;

	["ControlImmunity"] = {
		["Deflect"]   = true;
		["Attract"]   = true;
		["AreaTime"]  = true;
	};
});
```

## Projectile Clashes

Both entries must explicitly enable `ProjectileCollision` and be server-authoritative simulated gameplay projectiles. Client-only effects, physical projectiles, and arbitrary unregistered objects cannot clash. **CreateProjectilePart false does not disqualify an actual server gameplay entry.** Clash radius is dedicated gameplay geometry, independent from visual scale or world-cast dimensions.

| Setting | Default / behavior |
| --- | --- |
| `Enabled` | false |
| `Power` | 1; finite, nonnegative. Only strictly stronger applies its response. Equal power passes through. |
| `Radius` | Required positive radius when enabled. |
| `Response` | Destroy, Trigger, or Deflect; default Destroy. |
| `ConsumeOnClash` | false; winner continues unless explicitly consumed after success. |

`ConfigureProjectileCollisions({ IsFriendly = Callback? }?)` is server-only; nil clears the custom predicate. Same resolved owner/player and ordinary matching non-neutral player Teams are always excluded. The optional synchronous `(FirstEntry, SecondEntry) -> boolean` adds NPC/custom allegiance rules. Unknown ownership is neutral; errors reject the interaction with bounded diagnostics.

Destroy uses normal cleanup reason `ProjectileClashDestroyed`, not a manufactured world Hit. Trigger requires the **victim's** server `OnClashTrigger(Entry, OtherEntry?, Context)`; invokes once then cleans up as `ProjectileClashTriggered`, including callback failure. Without that hook the response is rejected. Deflect reflects the weaker projectile, preserves its speed, transfers current ownership, and reuses count/immunity/reset behavior. An immune/rejected response does not silently turn into destruction.

`OnProjectileClash` (server) and `OnClientProjectileClash` (visual notification) receive Entry, optional OtherEntry, and committed context with outcome/role, contact position/normal, IDs, and incoming velocities. They cannot rewrite the committed outcome. Existing destroy callbacks still run: abilities that explode for **every** destruction will still do so. Guard reasons explicitly if engulfing should suppress that explosion.

Clashes use a swept serial cohort/spatial candidate lane, relative-motion tests, contact deduplication, and stale-result invalidation. World contacts take precedence on numerical ties. Initial overlap and frozen participants do not imply free computation; dense clumps can still be expensive. Clients receive reliable outcomes, not the entire candidate-pair graph. Physical clashes and mesh-accurate clash shapes are outside this version.

## Impact Context And Attachments

`GetImpactContext(Entry)` reads the immutable snapshot retained on a callback entry, including after registry removal. It accepts an **entry**, not an ID. There is no global impact-history cache.

Context contains `ProjectileIdentifier`, surface `Position`, optional tracked `ProjectilePosition`, `Normal`, `Material`, `Instance`, `IncomingVelocity`, `IncomingDirection`, `LocalPosition`, `LocalNormal`, `LocalDirection`, and `Authoritative`. Instance references can be missing/destroyed/streamed out. Snapshots contain no arbitrary UserData or reference to the recycled projectile visual.

Surface Position is not automatically the projectile center for a sphere/block/shape contact. Local values are captured against ordinary hit parts. Context is captured for committed terminal simulated contacts and accepted physical autocast hits, not speculative probes, pierced contacts, successful deflections, or bounces.

`GetImpactCFrame(Context, Placement?) -> CFrame` resolves a pose without creating anything. Placement: `Alignment = "Incoming"` (default) or `"Normal"`, local `Offset = CFrame.identity`, and `FollowTarget = true`. With a live target/local data it resolves against the target's current pose; otherwise it uses captured world data. Missing direction/normal degrades to available context and safe forward, not invented contact metadata.

`AttachImpact(Context, Template, Options?) -> ImpactHandle?, reason?` is **client-only**. It clones a separate BasePart, Model, or Attachment. Options add finite nonnegative `HoldTime = 1.5`, `FadeTime = 0.35` to placement fields. Returns nil/reason on rejection. The handle has idempotent `Destroy()`, `IsActive()`, and `GetObject()` (nil after disposal).

Decorative parts are anchored/noncolliding/nontouching/nonqueryable. Following uses captured local space and moves all Model parts with shared batched updates; it does **not** weld to the target or alter a gameplay assembly. Attachment templates can use native local parenting; missing targets/Terrain use world placement/carriers. Followed effects end when the target leaves Workspace; no replacement search/wait occurs. Effects are positioned and stale trails cleared before restoration; fading disables continuing emission and fades geometry/decals. No automatic project-specific bursts, damage, or VFX calls occur.

### Embedded Blade

Assumes a separate client `EmbeddedBladeTemplate` and authoritative Context from the hit callback:

```lua
local function EmbedBlade(Context : ProjectileCore.ProjectileImpactContext) : ()
	local Handle, Reason = ProjectileCore["AttachImpact"](Context, EmbeddedBladeTemplate, {
		["Alignment"]    = "Incoming";
		["Offset"]       = CFrame.new(0, 0, -0.35);
		["FollowTarget"] = true;
		["HoldTime"]     = 1.5;
		["FadeTime"]     = 0.35;
	});

	if not Handle then
		warn("Embedded blade rejected: " .. tostring(Reason));
	end;
end;
```

Offset/depth depends on your authored pivot and forward axis; Core does not infer a blade tip from mesh bounds or inherit random projectile spin. For a moving-target particle effect, pass an Attachment template, get its clone through `Handle:GetObject()`, then emit from a known child ParticleEmitter yourself. Do not reparent/recycle that owned clone. Misfortune's existing weapon-specific implementation is unchanged; migrating means requesting hit metadata and cloning a separate template through this helper, not handing it the active projectile Part.

ReplicateHitInfo enables context metadata through existing reliable Destroy/InstantHit delivery; there is no additional per-frame impact remote. Physical contexts use your ability's own VFX delivery when clients need them.

## Distance And Age Curves

Simulated-only `Curves` can be authored on definitions/spawns and replaced through patches or `SetProjectileCurves(Identifier, Curves?, ShouldBroadcast?)`. Nil in the setter clears the configuration; an empty Curves patch clears it too. Replacement preserves progress and reconciles momentum. `ResetProjectileCurves(Identifier, ShouldBroadcast?)` restarts **curve progress only**; it returns false/NoCurves if none exist. Both setters return boolean/reason and reject unsafe decision hooks or client-mirror gameplay mutation.

`GetCurveValue(IdentifierOrEntry, Name)` reads a built-in channel or named numeric value, or nil when absent. `GetProjectileCurveState(IdentifierOrEntry)` returns copied Revision, effective Age, committed Distance, and Values. Entry-based reads remain usable in terminal callbacks. Values do not automatically apply damage or mutate UserData.

| Curves field | Output |
| --- | --- |
| `Speed` | Base-motion multiplier, finite nonnegative; 0 holds base movement. |
| `VisualScale` | Positive visual multiplier relative to prepared baseline. |
| `CollisionScale` | Positive world/humanoid sweep multiplier, independent from visual size. Rays remain rays; clash radius is unchanged. |
| `Values` | Up to 16 named finite numeric channels, such as Damage or BlastRadius. |
| `ScaleParticles` | false by default; opt in to saved-baseline Size/Speed/Acceleration scaling. |

Each curve has `Input = "Age"` or `"Distance"`, and 1-32 `Keyframes`. Each keyframe has finite nonnegative strictly increasing `At`, finite `Value`, and optional native EasingStyle/EasingDirection controlling the segment **ending at that point**. Defaults are Linear/Out. Values hold outside endpoints; there are no loops or random envelopes. Named channels are 1-64 bytes and cannot collide with built-in/reserved names.

Age uses effective projectile seconds. Distance accumulates committed path travel through turns, bounces and orbiting, excluding sync corrections/teleports. Contact progress is committed before terminal damage reads, not based on an untravelled remainder. Recasts, deflection, ownership and phases preserve progress; only explicit reset restarts it.

Speed modifies base motion without multiplying the previous result repeatedly. Gravity/homing remain integrated, Wind is a separate contribution, and exclusive Orbit/Tornado/Attract control wins while captured. On release current momentum is reconciled. Bezier speed changes progression, not only a reported velocity. Zero Speed still allows Age progression; FrozenMoment pauses effective movement/curve time. Redirect/deflection/clear reconcile the current multiplier once rather than stacking it.

Visual baselines are captured per lease and restored before cache return. Whole Models, attachments and applicable joint offsets are handled through Core's visual writer. Geometry updates are capped at 30Hz, optional particle properties at 15Hz, with initial/final values forced and unchanged values skipped. No per-projectile Tween or extra movement connection is created.

CollisionScale validates supported native dimensions, including maximum radius/block components. Scaled shapecasts need supported decoupled cast geometry; unsupported configurations reject instead of silently using rendered size. Expansion while stationary does not add overlap detection. Configuration/revision/progress use existing replication; evaluated channel values are not streamed every frame. Speed/CollisionScale stay serial; purely visual/value curves can remain eligible for validated main-thread commits with the worker path.

### Growing Meteor And Distance Damage

```lua
ProjectileCore["RegisterProjectile"]("GrowingMeteor", {
	["CreateProjectilePart"] = false;
	["SpherecastRadius"]     = 3;

	["Curves"] = {
		["VisualScale"] = {
			["Input"]     = "Age";
			["Keyframes"] = {
				{ ["At"] = 0; ["Value"] = 1; };
				{ ["At"] = 3; ["Value"] = 5; ["EasingStyle"] = Enum.EasingStyle.Exponential; ["EasingDirection"] = Enum.EasingDirection.Out; };
			};
		};

		["Values"] = {
			["Damage"] = {
				["Input"]     = "Distance";
				["Keyframes"] = {
					{ ["At"] = 0; ["Value"] = 80; };
					{ ["At"] = 250; ["Value"] = 25; };
				};
			};
		};
	};

	["OnServerHit"] = function(Entry : ProjectileCore.SimulatedProjectileEntry, HitResult : RaycastResult?, Reason : string, Context : ProjectileCore.ProjectileImpactContext?) : ()
		local Damage = ProjectileCore["GetCurveValue"](Entry, "Damage");
		if not Damage or Reason ~= "Hit" then return; end;

		--// Apply validated ability damage here using Damage and committed contact data.
	end;
});
```

This grows the **visual only**; add a separate CollisionScale curve if gameplay collision should grow too. A Speed curve from 0.5 to 2 over Age accelerates a needle relative to its base speed. For a return attack, TransitionProjectile preserves curve progress; explicitly call ResetProjectileCurves after a successful transition only if the return should restart the curve. Do not reset lifetime limits accidentally to keep one blade alive forever.

## Physical Projectiles

`SpawnPhysical(Projectile : BasePart | Model, Information : PhysicalSpawnInfo)` registers an existing parented Roblox physics object and returns ID/entry. A Model needs a resolvable BasePart. Core builds mover/constraint state, tracks ownership/deflection/void/lifetime, and moves the object into its physical runtime container. It is not a simulated partless grenade and does not use simulated visual replication.

Spawn fields: Creator, UserData, ControlImmunity, Group, ProjectileIdentifier, MaxDeflections (24), Deflectable (enabled unless false), AutoLifetime (optional), TimeScale (1), TravelMode (Linear), TravelData, PhysicalCast, callbacks. Physical UserData stays local to its entry; this API does not add a simulated Spawn sidecar or automatically deliver physical impact metadata to clients.

`AutoLifetime` opts into finite lifetime tracking; physical time/control behavior uses physical adapters rather than promising identical simulated integration. World collisions and physical bounce remain Roblox's responsibility. `OnDeflect(Entry, Deflector?)` and `OnDestroy(Entry, Reason)` are entry notifications. Legacy `OnDeflected(Deflector?)`, `OnDestroyed()`, and `OnVoided()` have different argument shapes. The existing physical owner-change adapter has legacy NewOwner notification semantics; do not blindly reuse a simulated callback shape without checking that integration.

### PhysicalCast

Root SpawnPhysical **does pass through** PhysicalCast, OnPhysicalHit and OnPositionChange. This is no longer the old half-wired example.

| Field | Behavior |
| --- | --- |
| `Enabled` | Set true explicitly to request autocast. |
| `Mode`, alias `CastMode` | Raycast, Spherecast, Blockcast, Shapecast; legacy Ray/SphereCast/BlockCast/ShapeCast spellings accepted. Omitted mode can infer a supplied shape/size/radius, otherwise ray. |
| `Radius`, alias `SpherecastRadius` | Sphere size. |
| `Size`, alias `BlockSize`, optional `CFrame` | Block dimensions/pose. |
| `Shape`, alias `ShapePart` | BasePart cast geometry. |
| `IgnoreList`, `IgnoreOwner`, `IgnoreWater`, `RespectCanCollide` | Query filtering; IgnoreOwner is opt-in. |
| `CanPierce` | Synchronous `(Entry, HitResult, CastMode) -> boolean`; true continues without an accepted terminal contact. |
| `CancelOnHit` | true unless false. false permits accepted hits without automatic cancellation. |

`OnPhysicalHit(Entry, HitResult, CastMode, ImpactContext)` captures each accepted autocast hit independently, even with CancelOnHit false. `OnPositionChange(Entry, Position, LastPosition, Velocity, DeltaTime)` reports physical stepping. Native physics/Touched contacts without autocast do not produce this context automatically.

Assume GrenadePart is an existing parented unanchored BasePart with its velocity/movers already configured:

```lua
local Identifier, Entry = ProjectileCore["SpawnPhysical"](GrenadePart, {
	["Creator"]      = Character;
	["Deflectable"]  = true;
	["AutoLifetime"] = 5;

	["PhysicalCast"] = {
		["Enabled"]     = true;
		["Mode"]        = "Spherecast";
		["Radius"]      = 1;
		["IgnoreOwner"] = true;
		["CancelOnHit"] = true;
	};

	["OnPhysicalHit"] = function(ProjectileEntry : ProjectileCore.PhysicalProjectileEntry, HitResult : RaycastResult, CastMode : string, Context : ProjectileCore.ProjectileImpactContext) : ()
		local Position = Context["Position"];
		--// Server damage / ability-owned VFX delivery uses this surface position.
	end;
});
```

OnDestroyed is cleanup, not proof of a hit. Do not explode again in both OnPhysicalHit and an unconditional destroy callback unless you enjoy duplicate damage reports.

## Visuals And Effects

`PrepareProjectilePart` runs after valid tracked placement, before managed effect restoration/visibility. Its Context.Position is the intended tracked point, not a guessed template pivot. Transform warmup and the renderer own movement. Never add a second CFrame/Model pivot writer in OnPositionChange just to fight the renderer more efficiently.

`EffectPolicy` supports ManagedClasses, Allow/Deny filters by Classes/Names, and AutoEnable. `EffectsEnabled` is a spawn/patch switch. Core's managed effect/cache path clears stale trails, restores saved template values, and protects recycled leases. If a custom effect is outside that policy, initialize/clean it explicitly. A long OnPartEnd fade retains that leased visual; AttachImpact deliberately uses a separate clone instead.

### Angular Spin

Simulated visual-only settings: AngularSpin false, SpinStrength 0 **degrees/second**, optional local-space SpinDirection. Missing/near-zero axis uses local forward `(0, 0, -1)` for roll while facing travel. Spin uses elapsed time and TimeScale, not frame count, and does not redirect collision, velocity, homing, Bezier or field state.

Current renderer logic spins when AngularSpin is true **or absolute SpinStrength is nonzero**. AngularSpin true with strength 0 produces no angular progress. To stop a previously nonzero spin, patch **both** AngularSpin false and SpinStrength 0; setting the boolean alone does not override the strength in this build.

Historical `Angles = CFrame.Angles(...)` examples need a warning: the inspected renderer reads an internal Angles value, but the current public root spawn merge/types/patch surface do not wire that top-level field through. It is **not a verified public setting in this baseline**. Orient your template appropriately; this docs-only pass does not repair that missing plumbing.

### Visual Budgets And Debugging

`ConfigureVisualBudget(Options?)` configures client-only distance budgets; nil restores disabled defaults. Options: Enabled false, NearDistance 256, FarDistance 768, FarParticleRate 0.35, LightDistance 384. Only definitions/spawns with AllowVisualBudget true opt in. Existing managed rates/light state are restored during reconfiguration. Geometry, trails, explicit bursts, damage and server collision accuracy are not silently reduced.

DebugVisualize, DebugVisualizeCasts/Areas/Homing default false. DebugVisualizerLifetime defaults 0.15, Transparency 0.65, AlwaysOnTop true, Color optional. Use debug visualizers deliberately and disable them for gameplay measurements. They are not free content.

## Networking, User Data, And Recovery

### Transport And Relevance

`SetNetworkTransport(Mode?)` accepts Auto, ByteNet, SoNET; nil/invalid mode resolves to Auto in this integrated build. Auto uses successfully initialized ByteNet when available, otherwise SoNET. `GetNetworkTransport()` returns diagnostics including requested Mode, resolved ActiveTransport, availability, limits, queues and failures. `GetNetworkInfo()` is its alias. Inspect resolved state rather than assuming a requested string proves initialization succeeded.

Keep transport selection consistent on both sides and avoid switching while deliveries are pending; this API does not promise a cross-transport lifecycle barrier. Explicit SoNET is supported. Do not mix this integrated recovery endpoint with the older standalone package's different runtime/remotes.

ReplicationPolicy supports Mode Automatic/Global/Spatial/Owner/None, Distance, BoundsRadius, AlwaysRelevant, OwnerAlwaysRelevant, Priority, InstantHitVisual, PredictVisualHits. Automatic resolves Spatial when StreamingEnabled, Global otherwise. Spatial distance defaults to max(StreamingTargetRadius + 256, 512), BoundsRadius to template bounding radius, owner relevance true, AlwaysRelevant false, Priority 0, InstantHitVisual true. Supply BoundsRadius on a server-invisible large projectile if the server has no template from which to infer it.

Interest checks run at 10Hz with 128 studs of exit hysteresis, late-spawn on entry, and visual-only cull on departure. Relevance affects network/presentation, not authoritative simulation/damage. A culled visual is not a cancelled server projectile.

PredictVisualHits is an optional cosmetic ray-clipping aid for eligible server-owned linear visuals. It never reports gameplay hits. It is not a substitute for authoritative hit metadata, spherecasts, or actual client-side damage permission.

Lifecycle spawn/redirect/bounce/deflect/destroy/cull/forced corrections are reliable, ordered, byte-budgeted and paced to at most eight packets/player/Heartbeat. Ordinary sync uses compact numeric IDs, byte-limited batches, a 12 KiB/s per-player application budget with burst allowance, newest-state coalescing and fair deferral. Unknown sync IDs are safely ignored until reliable spawn arrives. Reliable queue saturation defers work; it does not silently drop lifecycle events or switch transports.

Immutable lifecycle encoding is shared across recipients; recipient filtering/progress/fragmentation stays independent. This saves preparation work, not the bandwidth of sending to more machines. Single oversized lifecycle events use bounded reliable fragmentation: 256 KiB/message, eight pending messages, 1 MiB aggregate reassembly, five-second timeout. This is not permission to ship your inventory in every meteor.

Spawn/destroy within the current server flush window can collapse to InstantHit with authoritative start/contact data and a resolved transient client visual/callback path rather than a travelling SwiftCast. Existing callbacks remain relevant. Short paths are not guaranteed to produce a visible Roblox Trail sample; instant-hit/tracer policy and effect design matter.

### UserData

For simulated entries, ReplicateUserData defaults **true**. At spawn, explicit ReplicatedUserData supplies the replicated subset; otherwise Entry.UserData is sanitized for the client when the fallback flag is enabled. ReplicateUserData false disables that fallback, **not every possible UserData delivery path**. Explicit replicated subsets remain intentionally sendable. Never put secrets or large object graphs in these fields.

Sanitization whitelists payload keys, removes functions/unsupported values/cycles, caps nested tables at depth 16 and 128 fields/table, and truncates strings to 1,024 bytes in the final inspected source. Parented Instance references can travel; missing streamed instances may be nil client-side. Numbers, booleans, strings, Vector3/CFrame/Color3/EnumItem and bounded tables are the intended sidecar family. Arbitrary buffers/threads/functions are not a portable UserData schema.

**Current patch caveat:** PatchProjectile accepts UserData/ReplicatedUserData, merges that patch with Entry.UserData, and stores the merged result as the explicit replicated copy. Patch payloads can also carry the patched fields. Consequently, ReplicateUserData false or an earlier small explicit subset is **not a privacy boundary after such patches**; later patch/recovery/terminal delivery can expose data that was originally server-only. Keep private ability state in a separate server-owned table instead. This docs-only pass documents that behavior; it does not repair it.

Do not assume replacing an entry table replicates itself. SpawnPhysical's UserData is not automatically emitted through simulated spawn packets.

### Recovery And Guards

The existing integrated GetSnapshot command now returns a **small acknowledgement** with Status, optional RecoveryIdentifier, and RetryAfter; bundled clients consume paced reliable recovery records and completion messages instead of a giant returned array. Ability code should not call these internal remotes directly.

Recovery preserves one active job/player, one accepted request per five seconds, and at most 256 recovered projectiles per job with explicit truncation. Shared build work is bounded by 1ms, 64 candidate checks, or 16 serialized records per Heartbeat. Normal relevance discovery can handle remaining entries. Stale identity/generation/relevance checks prevent recovery resurrecting ended projectiles.

Inbound defaults: 120 calls/second, burst 240, per-player **per transport family**; independent ByteNet buffer/reference byte allowances and 512 packet-wide reference nodes; SoNET application bytes 64 KiB/second, burst 128 KiB, with strict packed input/output 64 KiB, depth 16, 4,096 nodes, 2,048 collection entries, 256 references, 64 arguments. At most two guarded invokes/player are active across the shared guard state.

Core accepts no client remote Spawn/Hit/Destroy/Deflect/Patch/owner commands. Packed SoNET events need a registered server listener **before decompression**. Recovery requires the engine-authenticated sender, exact GetSnapshot action, zero extra arguments, and a separate attempt budget of 2/second burst 4. Unknown commands do not allocate attacker-named buckets/diagnostic keys. Rejections are counted with bounded/rate-limited diagnostics, not automatic kicks/bans.

The Studio benchmark remote is Studio-only: exact known actions, 4 requests/second burst 8, weighted 96 emissions/second burst 192 charged before execution. Trusted heavier tests call server APIs directly; clients have no protection-disabling switch.

These are boundary guardrails, **not a complete weapon anti-exploit system**. Validate equipped/alive state, cooldown, ammo, aim/origin, teams, damage and physical network ownership in authoritative ability code. Optional strict codec limits preserve legacy/default encoding calls; shared transport compatibility still needs real-game testing.

## Collision Backends And Dynamic Occupancy

`SetCollisionMode(Mode, Options?)` sets Native (default), Auto, DDA, or Bolt. Definition/spawn/ProjectileSettings can override a projectile's mode. `GetCollisionInfo()` reports requested/resolved mode, thresholds, Bolt load status, and map Baked/Dirty/Truncated/Version/Coverage diagnostics. This is a broadphase/fallback optimization, not a new collision truth.

Options: AutoPreferredMode DDA, AllowEmptySpaceSkip false, NativeTimeThreshold 0.00035 **seconds**, CastsPerFrameThreshold 500, DecisionInterval 0.5 seconds. Auto uses Core-local samples, not global Stats CPU. Modes and thresholds are not guarantees of a speedup.

`BakeCollisionMap(Options?) -> info` accepts Root or Roots, CellSize (64), MaxParts (12,000), MaxCellsPerQuery (256), AnchoredOnly (true), TrackWorldChanges (true), optional synchronous ShouldIncludePart. Baking is opt-in and may be expensive; do it during loading, not per shot. `ClearCollisionMap()` clears grid/coverage/watchers and disables its dynamic occupancy tracking; it does not delete map geometry.

**Current empty-space skipping requires verified Include-filter roots with complete coverage.** Ordinary Exclude/IgnoreList casts, incomplete/truncated roots, unverified Workspace/Terrain coverage and dirty maps fall back to native. Sphere/block/shapecasts remain native. DDA/Bolt do not magically accelerate every ordinary weapon merely because a map was baked.

Bolt loads lazily. Its candidate math can avoid confirmed-empty ray work under permitted coverage/skip settings; nonempty/unsafe cases still use native raycasts for actual contacts. Moving/static map changes can dirty coverage and disable skipping until rebaked. Keep TrackWorldChanges enabled unless you knowingly own all invalidation.

`RegisterDynamicCollider(Collider : BasePart | Model) -> Handle` tracks explicitly moving geometry; the handle has idempotent Destroy. `UnregisterDynamicCollider(Collider)` removes explicit registration. The CollectionService tag `ProjectileCoreDynamicCollider` also tracks geometry; tagged objects stay tracked until their tag is removed. Character bounds are included through shared tracking. Occupancy updates at 15Hz, processes changed bounds, and expands with swept velocity. Register moving anchored obstacles; stale static occupancy is not a clever optimization.

## Profiling And Parallel Runtime

`SetPerformanceProfilingEnabled(Enabled, Options?)` toggles Core's internal profiling; options include MicroProfilerLabelsEnabled and StatsSamplingEnabled. `GetPerformanceStats()` returns rolling Metrics, Counters, frame summaries, broad Stats context, and Extra snapshots for active entries, caches, visual metadata, preparation, queries/homing schedule, network, fields, curves, groups/bursts, impacts, spatial indexing, dynamic occupancy and workers.

This diagnostic getter enumerates current state. Sample it at a modest debug/UI rate, not once per projectile per frame. Profile labels include PCore.NativeCast, PCore.BoltQuery, PCore.BakedDDA, PCore.ActorBatch, PCore.AreaController and PCore.Replication. Stats values are broad context and may be unavailable/zero; they do not establish a zero-cost frame or decide collision mode by themselves.

`SaveScriptProfilingData(FileName?) -> boolean, resultOrError` is an optional Studio-only helper. It does not make ScriptProfilerService a live gameplay dependency or guarantee every environment permits saving. Use MicroProfiler captures with labels for frame investigation.

Network diagnostics are available through GetNetworkTransport/GetNetworkInfo and `GetPerformanceStats().Extra.Network`: resolved transport/packet limits, encoding/reuse/preparation time, queues/high-water/age, sync sent/deferred/budget saturation, fragmentation/reassembly, recovery requests/denials/build/latency, client inbox depth/age/drain/callback failures, plus fixed `RemoteGuards` rejection/throttling/decode/active-invoke counters. Counts and wall-time measurements are different things; do not label a queue wait as network RTT.

### ConfigureParallel

`ConfigureParallel(Mode, Options?)` supports Disabled, Auto (default), Enabled. **AutoReady is currently false: Auto remains serial pending parity/performance promotion.** Enabled explicitly requests the experimental server worker lane, still subject to worker initialization/availability and workload thresholds. It is not the recommended fix for a client inbox queue.

Defaults: WorkerCount 8, MinimumEligibleProjectiles 256, EntryBudgetMilliseconds 1.5, ExitBudgetMilliseconds 0.75, MaximumStaleFrames 2. Workers perform eligible computation/casts; authoritative commits, callbacks and registry/replication remain serial. Failure/stale results fall back rather than resurrecting entries.

The current eligibility check excludes client authority, physical entries, non-linear/Bezier, homing, controller/Attract/Wind state, clashes, gameplay-changing curves, separate humanoid sweeps, custom raycast/pierce callbacks, shapecast, raycast-deflection-enabled shots and bouncing. Since AllowRaycastDeflection defaults true, ordinary defaults do **not** automatically enter this lane. Do not change that gameplay setting only to win a synthetic benchmark.

## Registry And Compatibility Helpers

`GetData(Identifier)` returns a live entry or nil; `IsActive(Identifier)` checks current registry activity. `GetAll()` returns a **shallow copied map** of active entries: changing map membership cannot mutate the registry, but the entry values themselves are still live references. Use supported mutations instead of raw nested-state writes.

`GetOwnerFromPart(Part : BasePart)` resolves a registered entry or the existing Creator-tag fallback. It is not proof that a client owns permission to fire or damage with that part.

`DetectHumanoidsNearby(Origin, Range, IgnoreList, Owner?, Visualizer, IgnoreForceFields, IgnoreFriendly?, PartLimit?, VisualizerProperties?, UseRobloxQuery?) -> boolean, { Humanoid }` is a **game-integrated compatibility alias** to HumanoidDetection/Hitbox_Module, not the standalone package's native replacement. IgnoreList must be a table. IgnoreForceFields defaults true (exclude protected targets), IgnoreFriendly false, positive PartLimit caps results otherwise uncapped, UseRobloxQuery false uses the registry path. VisualizerProperties supports Color, Lifetime, Transparency. Supply the owner for the existing game allegiance rules; result order is not guaranteed.

Use the alias only when that external integration is installed. Do not copy the old shorter positional signature from an unrelated UtilityFunctions helper and assume argument seven means the same thing. This is how a ForceField becomes a teammate by accident.

## Current Limits And Verification

- This reference describes the inspected integrated source, not an unpublished standalone refresh. There is no package download/source release in this repository yet.
- Top-level Angles is not wired through the inspected public spawn/types surface despite internal renderer support. Runtime registration discovery is not an arbitrary recursive dependency loader. Neither limitation was repaired during documentation work.
- ReplicateUserData is not a comprehensive privacy switch for UserData patches; see the explicit caveat above. Keep private server state outside replicated entry data.
- Final readback caught a concurrent Replication edit that removed the RecoveryRecords initialization while leaving its increment in the recovery scheduler. That source needs repair before treating current recovery as operational; it still passes syntax compilation. The scheduler contract above describes the existing design, not a successful runtime retest of that edit. No Studio code was changed during this documentation pass.
- Preloading, scheduled acquisitions, shared encoding, reusable filters and caches reduce particular costs. They do not establish a universal projectile capacity, latency bound, allocation-free loop, or leak-free game.
- Recent eight-local-client tests delivered 48/48 projectile callbacks/visual releases per client through each transport, with zero reported send/flush failures and terminal queue/active-resource residue in that harness. They also saw receive-to-prepare delays up to about 1.55 seconds and high client frame intervals on the shared laptop. Hardware contention and inbox scheduling need isolated measurements; these results are not a normal-player latency guarantee.
- Age-aware catch-up/budget-debt and cheaper InstantHit work discussed after that test are **proposals, not implemented fixes** in this baseline. Do not advertise them as shipped.
- Serial/native remains the conservative default. DDA coverage restrictions and unpromoted Actor Auto are deliberate guardrails, not a promise that flipping a string doubles FPS.
- Native syntax checks, callback-count harnesses and bounded cleanup cycles are narrower than manual appearance/jitter checks, adverse-network/device testing, long-duration memory certification or 16-player readiness. Measure those separately. No, multiplying an overloaded eight-client laptop result by two is not a benchmark.
