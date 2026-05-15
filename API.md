# ProjectileCore API Reference

This is the long-form reference forprojectileCore. It covers authority modes, registration, simulated projectiles, physical projectiles, live patching, efffects, Bezier travel, homing, deflection, attraction, networking, debug visualizers, and performance settings.

@SonarHEXAGON plz stop arguing with mr Blaj here - Reze

## Require Path

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage");

local ProjectileCore = require(ReplicatedStorage["ModuleScripts"]["ProjectileCoreSystem"]:WaitForChild("ProjectileCore"));
```

## Authority Model

`Authority = "Server"` means the server owns hit detection, deflection, redirects, ownership, time-scale changes, damage callbacks, and destroy state. Clients should register the same projectile name with visual settings so replicated spawns render locally.

`Authority = "Client"` means the local machine owns simulation and hit callbacks. Use this for cosmetic projectiles, local previews, or trusted benchmark tests. Do not use client authority for real damage unless the server validates the result.

Server-authority client replicas are visual-only. They should not resolve their own gameplay hits. They render spawn, deflect, redirect, patch, state-sync, and destroy updates from the server.

## Package Layout

```text
ReplicatedStorage
  ModuleScripts
    ProjectileCoreSystem
      ProjectileCore
        Documentation
        Network
        Objects
          Projectiles
        Physical
        Shared
        Simulated
        Simulators
```

Projectile presets are normally stored at:

```text
ProjectileCore.Objects.Projectiles.<ProjectileName>
```

Unless somebody fucked up enough to make it wrong omfg. Anyways, a preset can be a direct `BasePart`, a `Model` with `PrimaryPart`, or a `ModuleScript` returning projectile/cache data.

## Core API

```text
ProjectileCore.RegisterProjectile(ProjectileName, ProjectileDefinition?) -> RegisteredProjectileDefinition
ProjectileCore.SpawnSimulated(ProjectileName, SpawnInformation) -> Identifier, Entry
ProjectileCore.SpawnPhysical(ProjectileInstance, SpawnInformation) -> Identifier, Entry
ProjectileCore.Deflect(Identifier, DeflectionParams, ShouldBroadcast?)
ProjectileCore.Redirect(Identifier, NewVelocity, NewPosition?, NewAcceleration?, NewIgnoreList?, ShouldBroadcast?)
ProjectileCore.DeflectArea(DeflectionParams) -> { BasePart }
ProjectileCore.AttractArea(AttractParams) -> { ProjectileEntry }
ProjectileCore.SetHoming(Identifier, HomingData?, ShouldBroadcast?)
ProjectileCore.SetOwner(Identifier, NewOwner?)
ProjectileCore.GetOwner(Identifier) -> Instance?
ProjectileCore.SetTimeScale(Identifier, SpeedScale, ShouldBroadcast?)
ProjectileCore.PatchProjectile(Identifier, PatchData, ShouldBroadcast?)
ProjectileCore.Cancel(Identifier, Reason?)
ProjectileCore.DestroyProjectile(Identifier, Reason?)
ProjectileCore.GetData(Identifier) -> ProjectileEntry?
ProjectileCore.GetAll() -> { [string]: ProjectileEntry }
ProjectileCore.IsActive(Identifier) -> boolean
ProjectileCore.GetOwnerFromPart(Part) -> Instance?
ProjectileCore.SetNetworkTransport("Auto" | "ByteNet" | "SoNET")
ProjectileCore.GetNetworkTransport() -> table
ProjectileCore.GetNetworkInfo() -> table
```

### Core API Details

| Function | What It Does | Use It For | Notes |
| --- | --- | --- | --- |
| `RegisterProjectile` | Stores defaults for one projectile name. | Shared setup for bullets, rockets, droplets, shuriken, beams, or benchmark presets. | Safe to call on server and client. The same name should be registered on both sides when server authority uses client visuals. |
| `SpawnSimulated` | Creates a fixed-step simulated projectile entry. | Fast bullets, magic shots, missiles, lobbed droplets, Bezier shots, auto-homing projectiles. | Returns `Identifier, Entry`. Server-authority spawns can replicate to clients. |
| `SpawnPhysical` | Registers an existing BasePart or Model as a physical projectile. | Rigidbody-style thrown objects, mover-driven projectiles, physics interactions. | Uses Roblox physics/movers instead of pure simulated stepping. |
| `Deflect` | Redirects one active projectile using reflection, return, locate, lob, or direct velocity data. | Parry windows, shields, bounce surfaces, projectile counters. | Usually pass `ShouldBroadcast = true` on server authority. |
| `Redirect` | Recasts a projectile from a new position, velocity, acceleration, and ignore list. | Manual course changes, teleport-like continuation, post-impact continuation. | Bezier travel is replaced from the redirect point onward. |
| `DeflectArea` | Finds projectiles inside an area and deflects them. | AoE parries, close-range counters like azure peri, shockwaves. | Use `AreaGroup` to prevent overlapping effects from arguing like gremlins. |
| `AttractArea` | Applies blackhole-style por vortex steering to projectiles inside an area. | Gravity wells, vortexes, vacuum pulls, repulsion-style controller setups. | Attraction does not change owner or enable deflection by itself. |
| `SetHoming` | Adds, replaces, or disables homing data on an active projectile. | Delayed homing, target swaps, auto-acquire, post-Bezier tracking. | Server-authority visuals need correction packets for close visual alignment. |
| `SetOwner` | Changes the owner/creator reference on the projectile. | Parry ownership transfer, redirected damage credit, projectile stealing. | Use this instead of trying to patch ownership. |
| `GetOwner` | Returns the current owner for an identifier. | Damage credit, team checks, validation, cleanup ownership. | Can return nil if the projectile no longer exists or has no owner. |
| `SetTimeScale` | Scales projectile simulation time. | Slow motion, pausing, speed boosts, local benchmark tests. | Simulated projectiles reject negative values. Physical projectiles may support negative/reverse behavior. |
| `PatchProjectile` | Live-patches validated simulated runtime fields. | Changing speed, acceleration, ray settings, homing, effects, correction settings. | Does not allow identity, ownership, template, or cache mutation. No spooky mutation soup. |
| `Cancel` | Stops and cleans up an active projectile. | Manual despawn, ability cancel, invalid owner cleanup, timeout cleanup. | Runs destroy/end paths with the supplied reason when supported. |
| `DestroyProjectile` | Alias for `Cancel`. | Same as `Cancel`. | Exists for clearer intent in cleanup-heavy code. |
| `GetData` | Returns one active projectile entry. | Inspection, debugging, advanced callbacks, runtime decisions. | Do not mutate random nested fields directly unless you enjoy future pain and shi. Use `PatchProjectile`. |
| `GetAll` | Returns the active projectile registry. | Benchmarking, debug panels, cmdr stuff incase added or whatever, area controllers. | Treat as read-mostly. Heavy scans every frame are how FPS gets shat. |
| `IsActive` | Checks if an identifier still exists and is active. | Delayed homing, delayed cleanup, async callbacks. | Always check before delayed `SetHoming`, `PatchProjectile`, or `Cancel`. |
| `GetOwnerFromPart` | Resolves an owner from a projectile visual/physical part. | Touch handlers, physical projectile integration, debug selection. | Works only when the part is known/mapped by ProjectileCore. |
| `SetNetworkTransport` | Selects the packet transport mode. | ByteNet Max integration, fallback behavior, debugging network setup. | `Auto` prefers ByteNet Max when available and falls back to SoNET. |
| `GetNetworkTransport` | Returns the active transport object/state. | Debugging transport setup. | Mostly internal/admin-facing. |
| `GetNetworkInfo` | Returns transport mode and availability information. | Startup logs, benchmark UI, diagnosis. | Helpful when ByteNet is expected but fallback happens. |

### Broadcast Rules

`ShouldBroadcast` controls whether a server-side runtime change is sent to client replicas. For server-authority projectiles, use `true` for changes that should be visible client-side, such as deflects, redirects, homing changes, time scale changes, patches, and owner-affecting visual state.

For client-only cosmetic projectiles, broadcasting is usually unnecessary. For server-only gameplay without client visuals, broadcasting is also unnecessary.

### Identifier Rules

Every spawned projectile has a unique identifier. Keep the identifier if you plan to mutate the projectile later.

```lua
local Identifier, ProjectileEntry = ProjectileCore.SpawnSimulated("Soul_Fireball", SpawnInformation);

task.delay(0.2, function()
	if ProjectileCore.IsActive(Identifier) then
		ProjectileCore.SetHoming(Identifier, HomingData, true);
	end;
end);
```

Do not call runtime APIs with the projectile name unless the API specifically expects a projectile definition name. `"Soul_Fireball"` is the registered type; `Identifier` is the specific live shot. Yes, this one has bitten people. No, we are not naming names.

## RegisterProjectile

```lua
ProjectileCore.RegisterProjectile("ProjectileName", {
	["ProjectileTemplate"]       = Template;
	["ProjectileCacheWarmCount"] = 32;
	["CreateProjectilePart"]     = true;
	["ProjectilePartParent"]     = Workspace["ProjectileVisuals"];
	["ReplicateToClients"]       = true;
	["SpherecastRadius"]         = 3;
	["UseShapecast"]             = false;
	["AllowRaycastDeflection"]   = true;
	["MaxDeflections"]           = 24;
	["Authority"]                = "Server";
});
```

### Registration Settings

| Setting | Type | Default | Description |
| --- | --- | --- | --- |
| `ProjectileTemplate` | `BasePart` or `Model` | preset lookup | Visual template used for projectile parts. |
| `ProjectileCache` | cache object | internal cache | Optional explicit cache object. |
| `ProjectileCacheWarmCount` | `number` | `10` | Number of cached objects pre-created. |
| `ProjectilePartParent` | `Instance` | `Workspace` | Parent for active visual projectile objects. |
| `CreateProjectilePart` | `boolean` | `true` | Creates visual part/model when true. |
| `ProjectileSettings` | `table` | default settings | Lifetime and step/pierce settings. |
| `EffectPolicy` | `table` | default policy | Managed effect toggling policy. |
| `PrepareProjectilePart` | `function` | nil | Runs after initial CFrame is valid and before fire. |
| `OnPartEnd` | `function` | nil | Custom visual cleanup before returning/destroying the part. |
| `SpherecastRadius` | `number` | nil | Uses spherecast when greater than zero. |
| `UseShapecast` | `boolean` | `false` | Uses projectile part as cast shape. |
| `IgnoreWater` | `boolean` | `false` | Raycast water behavior. |
| `AllowRaycastDeflection` | `boolean` | `true` | Allows automatic `CanDeflect` hit deflection. |
| `MaxDeflections` | `number` | `24` | Maximum deflections before final handling. |
| `ReplicateToClients` | `boolean` | `true` | Server broadcasts simulated spawns and runtime changes. |
| `Authority` | `"Server"` or `"Client"` | context-based | Simulation owner. |
| `TimeScale` | `number` | `1` | Initial simulation speed multiplier. |
| `TravelMode` | `string` | `"Linear"` | `Linear`, `QuadraticBezier`, or `CubicBezier`. |
| `TravelData` | `table` | nil | Bezier target/control/duration settings. |
| `HomingData` | `table` | nil | Initial homing settings. |
| `UserData` | `table` | nil | Custom data copied onto entries. |

### Registration Callback Reference

| Callback | Runs On | Purpose |
| --- | --- | --- |
| `PrepareProjectilePart(Entry, ProjectilePart, Context)` | Client or server where visual part is created | Last setup pass after initial CFrame is correct and before managed effects are enabled. |
| `OnPartEnd(Part, Reason, Cleanup)` | Client or server where visual part exists | Custom visual cleanup, impact hold, fade, shrink, and cache return timing. |
| `OnServerHit(Entry, HitResult, Reason)` | Server | Authoritative hit response, damage, explosion logic, status effects. |
| `OnClientHit(Entry, HitResult, Reason)` | Client | Cosmetic local impact response or client-authority hit behavior. |
| `OnServerDestroy(Entry, Reason)` | Server | Authoritative cleanup after cancel, timeout, distance limit, or hit. |
| `OnClientDestroy(Entry, Reason)` | Client | Visual cleanup for replicated/client-owned projectiles. |
| `OnPositionChange(Entry, Position, LastPosition)` | Simulation owner/visual replica | Extra orientation or lightweight visual logic during motion. |

Keep `OnPositionChange` lean. If it allocates tables, scans folders, and summons three particle systems every step, the profiler will find you.

## SpawnSimulated

Spawn settings override registration defaults for that shot.

```lua
local Identifier, ProjectileEntry = ProjectileCore.SpawnSimulated("Bullet", {
	["Position"]         = Muzzle["WorldPosition"];
	["Velocity"]         = Muzzle["WorldCFrame"]["LookVector"] * 420;
	["Acceleration"]     = Vector3.zero;
	["Creator"]          = Character;
	["IgnoreList"]       = { Character; Tool; };
	["Authority"]        = "Server";
	["SpherecastRadius"] = 2.5;
});
```

### Spawn Settings

| Setting | Type | Description |
| --- | --- | --- |
| `Position` / `StartingPosition` | `Vector3` | Initial world position. |
| `Velocity` | `Vector3` | Initial velocity. If absent, `Direction` plus `Speed` is used. |
| `Direction` | `Vector3` | Direction used with `Speed`. |
| `Speed` | `number` | Magnitude used with `Direction`. |
| `Acceleration` | `Vector3` | Constant acceleration, such as gravity or drift. |
| `Creator` | `Instance` | Owner source, usually Player, Character, Tool, or NPC Model. |
| `IgnoreList` | `{ Instance }` | Raycast/shapecast exclusions. |
| `Authority` | `"Server"` or `"Client"` | Explicit authority override. |
| `ProjectileIdentifier` | `string` | Optional custom identifier. |
| `SkipReplication` | `boolean` | Prevents server spawn broadcast for this projectile. |
| `UserData` | `table` | Custom data stored on `Entry.UserData`. |
| `HomingData` | `table` | Initial homing controller state. |
| `TravelMode` | `string` | Travel mode for this shot. |
| `TravelData` | `table` | Bezier/travel data for this shot. |
| `AttractState` | `table` | Advanced replicated attraction state. Usually use `AttractArea`. |

### Spawn Return Values

| Return | Description |
| --- | --- |
| `Identifier` | Unique live projectile id. Use this for `SetHoming`, `PatchProjectile`, `Cancel`, `Deflect`, and `Redirect`. |
| `Entry` | Runtime projectile entry. Contains owner, simulated state, visual part/model, callbacks, user data, and controller state. |

The identifier is the safe handle. The entry is useful for callbacks and inspection, but runtime mutation should go through APIs where possible.

## ProjectileSettings

```lua
["ProjectileSettings"] = {
	["MaxFlyTime"]        = 8;
	["MaxFlyDistance"]    = 2500;
	["RaysPerMove"]       = 8;
	["MaxPiercesPerStep"] = 8;
};
```

| Setting | Type | Description |
| --- | --- | --- |
| `MaxFlyTime` | `number` | Seconds before auto-destroy due to lifetime. |
| `MaxFlyDistance` | `number` | Studs before auto-destroy due to travelled distance. |
| `RaysPerMove` | `number` | Compatibility alias for `MaxPiercesPerStep`. |
| `MaxPiercesPerStep` | `number` | Maximum hit/pierce checks per fixed simulation step. |

## Replication Settings

These matter most for server-authority homing, Bezier, and attract-controlled projectiles.

| Setting | Type | Default | Description |
| --- | --- | --- | --- |
| `ReplicationSyncInterval` | `number` | `0.08` | Server correction packet interval. Set `0` to disable. |
| `ReplicationSmoothTime` | `number` | `0.1` | Client damping time for correction packets. |
| `ReplicationSnapDistance` | `number` | `60` | Drift distance where client hard-snaps to server state. |

Lower `ReplicationSyncInterval` values improve visual alignment but increase network traffic. Higher `ReplicationSmoothTime` hides correction jitter but trails farther behind the server. Lower `ReplicationSnapDistance` enforces accuracy sooner but can create visible trail snaps.

> Lower interval = more packets. shocking. who could have guessed.
> The practical range is usually `0.05` to `0.12`. - @SonarHEXAGON

## EffectPolicy

ProjectileCore can keep effects disabled until the initial CFrame is valid, run `PrepareProjectilePart`, then enable only the resolved allowed effects.

```lua
["EffectPolicy"] = {
	["ManagedClasses"] = { "ParticleEmitter"; "Trail"; "Light"; };

	["Allow"] = {
		["Classes"] = { "ParticleEmitter"; "Trail"; };
		["Names"]   = { "MuzzleTrail"; };
	};

	["Deny"] = {
		["Names"] = { "DoNotAutoEnable"; };
	};

	["AutoEnable"] = true;
};
```

| Setting | Description |
| --- | --- |
| `ManagedClasses` | Classes ProjectileCore may toggle. Defaults to `ParticleEmitter`, `Trail`, and `Light`. |
| `Allow.Classes` | Optional class allow-list. |
| `Allow.Names` | Optional name allow-list. |
| `Deny.Classes` | Optional class deny-list. |
| `Deny.Names` | Optional name deny-list. |
| `AutoEnable` | If false, effects remain disabled until patched. |

## PrepareProjectilePart

```lua
["PrepareProjectilePart"] = function(ProjectileEntry, ProjectilePart : BasePart, Context)
	ProjectilePart["Transparency"] = 0;
	ProjectilePart["Color"] = Color3.fromRGB(255, 230, 120);
end;
```

This runs after the projectile part has a valid initial CFrame but before the projectile is fired and before managed effects are enabled. This is the hook that prevents the classic "trail starts in Narnia" nonsense.

## OnPartEnd

```lua
["OnPartEnd"] = function(ProjectilePart : BasePart, Reason : string, Cleanup : () -> ())
	for _, Descendant in ProjectilePart:QueryDescendants("Trail") do
		Descendant["Enabled"] = false;
        Descendant:Clear();
	end;

	task.delay(0.25, Cleanup);
end;
```

Use this for impact visual holds, shrink tweens, fade-outs, trail cleanup, or returning cached parts after a delay. Always call `Cleanup()` when done. Seriously. The cache cannot read minds.

## HomingData

```lua
["HomingData"] = {
	["Enabled"]              = true;
	["Target"]               = TargetPart;
	["TargetPosition"]       = Vector3.new(0, 10, -100);
	["Offset"]               = Vector3.new(0, 1.5, 0);
	["TurnSpeed"]            = 8;
	["MinimumDistance"]      = 2;
	["AutoAcquire"]          = true;
	["AutoAcquireRange"]     = 220;
	["AutoAcquireInterval"]  = 0.08;
	["AutoAcquireRetarget"]  = true;
	["RequireLineOfSight"]   = false;

	["AutoAcquireValidator"] = function(TargetHumanoid : Humanoid, TargetCharacter : Model) : boolean
		return TargetHumanoid["Health"] > 0;
	end;
};
```

| Setting | Description |
| --- | --- |
| `Enabled` | False disables homing without clearing data. |
| `Target` | Instance target, usually BasePart, Model, or Attachment. |
| `TargetPosition` | Static world target position. |
| `Offset` | Offset added to resolved target position. |
| `TurnSpeed` | Steering strength. Higher values turn harder. |
| `MinimumDistance` | Distance where steering stops. |
| `AutoAcquire` | Searches registered humanoids for a target. |
| `AutoAcquireRange` | Max acquisition range. |
| `AutoAcquireInterval` | Retarget check interval. |
| `AutoAcquireRetarget` | Allows periodic target replacement when not false. |
| `RequireLineOfSight` | Requires clear ray to target. |
| `AutoAcquireValidator` | Custom humanoid/character filter. |

For server-authority homing, enable correction packets if the visual path matters. The client can predict, but the server owns the actual handling.

### Homing Behavior Notes

| Setting Group | Behavior |
| --- | --- |
| Direct target | Use `Target` or `TargetPosition` when the projectile already knows where to go. |
| Auto-acquire | Use `AutoAcquire = true` when the projectile should search for the nearest valid humanoid. |
| Retargeting | `AutoAcquireRetarget = true` allows the projectile to swap targets over time. This looks smarter, but can diverge visually without server correction packets. |
| Line of sight | `RequireLineOfSight = true` prevents homing through walls. It costs extra ray queries. |
| Validator | `AutoAcquireValidator` is where team checks, forcefields, boss immunity, and other game rules belong. |

## TravelData For Bezier

```lua
["TravelMode"] = "CubicBezier";
["TravelData"] = {
	["Target"]                = TargetPart;
	["TargetOffset"]          = Vector3.new(0, 1.5, 0);
	["TrackTarget"]           = true;
	["ControlPointA"]         = StartPosition + Vector3.new(-20, 35, 0);
	["ControlPointB"]         = StartPosition + Vector3.new(30, 20, -80);
	["Duration"]              = 1.15;
	["Speed"]                 = 220;
	["CompletionMode"]        = "Homing";
	["ContinueDuration"]      = 0.65;
	["HomingTurnSpeed"]       = 8;
	["HomingMinimumDistance"] = 2;
	["DestroyOnComplete"]     = false;
};
```

| Setting | Description |
| --- | --- |
| `End` / `Target` / `EndPosition` | End source. Can be position, CFrame, BasePart, Model, or Attachment. |
| `TargetOffset` | Offset added to the target. |
| `TrackTarget` | Updates end position as an Instance target moves. |
| `ControlPoint` | Quadratic control point. |
| `ControlPointA` | Cubic first control point. |
| `ControlPointB` | Cubic second control point. |
| `Duration` | Seconds to traverse the curve. |
| `Speed` | Used to derive duration and tangent velocity when duration is absent. |
| `CompletionMode` | `Linear`, `Destroy`, `Homing`, or `ContinueBezier`. |
| `ContinueDuration` | Segment duration when continuing Bezier toward a moving target. |
| `HomingTurnSpeed` | Turn speed when completion mode becomes homing. |
| `HomingMinimumDistance` | Minimum distance for post-curve homing. |
| `DestroyOnComplete` | Shortcut for `CompletionMode = "Destroy"`. |

`CompletionMode = "Homing"` is the cleanest way to make a Cubic Bezier projectile curve first, then directly track the target after the curve finishes.

### Travel Modes

| TravelMode | Control Points | Best 4 |
| --- | --- | --- |
| `Linear` | None | Normal bullets, rockets, and straightforward casts. |
| `QuadraticBezier` | `ControlPoint` | Simple arcs, soft curves, magical bends. |
| `CubicBezier` | `ControlPointA` and `ControlPointB` | Stylish multi-point curves, shuriken paths, cinematic tracking shots. |

### Completion Modes

| CompletionMode | Behavior |
| --- | --- |
| `Linear` | Leaves curve mode and continues using the current tangent velocity. |
| `Destroy` | Ends the projectile when the curve completes. |
| `Homing` | Switches into direct homing after the curve finishes. Use this when the curve is just the opening flourish. |
| `ContinueBezier` | Rebuilds/continues curve segments toward the target, useful for moving Bezier targets. |

## PatchProjectile

V1 patching is simulated-focused and validated. Unsupported keys fail cleanly instead of silently mutating mystery soup.

```lua
ProjectileCore.PatchProjectile(Identifier, {
	["Velocity"]                = Vector3.new(0, 0, -350);
	["Acceleration"]            = Vector3.zero;
	["IgnoreList"]              = { Character; Tool; };
	["IgnoreWater"]             = false;
	["SpherecastRadius"]        = 5;
	["UseShapecast"]            = false;
	["AllowRaycastDeflection"]  = true;
	["TimeScale"]               = 1.25;
	["EffectsEnabled"]          = true;
	["ReplicationSyncInterval"] = 0.05;
	["ReplicationSmoothTime"]   = 0.1;
	["ReplicationSnapDistance"] = 70;

	["HomingData"] = {
		["Enabled"]   = true;
		["Target"]    = TargetPart;
		["TurnSpeed"] = 10;
	};

	["EffectPolicy"] = {
		["AutoEnable"] = true;
	};
}, true);
```

Patching does not allow identifier, projectile type, template/cache source, or ownership changes. Use `SetOwner` for ownership.

### Patchable Fields

| Field | Effect |
| --- | --- |
| `Velocity` | Replaces current simulated velocity. |
| `Acceleration` | Replaces constant acceleration. |
| `HomingData` | Replaces active homing data. |
| `IgnoreList` | Replaces raycast/shapecast exclusions. |
| `IgnoreWater` | Updates water handling. |
| `SpherecastRadius` | Updates spherecast width. |
| `UseShapecast` | Toggles shape-based casts when supported. |
| `AllowRaycastDeflection` | Enables/disables raycast deflection behavior. |
| `TimeScale` | Updates simulation speed for simulated projectile. Negative values are rejected. |
| `EffectPolicy` | Updates managed visual effect policy. |
| `EffectsEnabled` | Enables or disables managed effects. |
| `ReplicationSyncInterval` | Updates server correction packet interval. |
| `ReplicationSmoothTime` | Updates client correction smoothing time. |
| `ReplicationSnapDistance` | Updates snap distance for severe visual drift. |

## Deflect

```lua
ProjectileCore.Deflect(Identifier, {
	["Deflector"]       = Character;
	["CenterPosition"]  = ProjectileEntry["Simulated"]["Position"];
	["ReflectionMode"]  = "Return";
	["SpeedMultiplier"] = 1.15;
	["MaxDeflectSpeed"] = 600;
	["SpreadAngle"]     = 4;
	["ReflectionBias"]  = 0.5;
	["ChangeOwner"]     = true;
}, true);
```

Common reflection modes include `Reflect`, `Return`, `Locate`, and `Lob`. Useful fields include `Velocity`, `Normal`, `Direction`, `Position`, `Deflector`, `IgnoreList`, `Acceleration`, `CenterPosition`, `Range`, `DeflectCooldown`, `OwnerSearchRange`, `LocateTargetValidator`, `IsTeammate`, `RespectDeflectable`, `RequireTouch`, and `Blacklist`.

### Deflection Types

| ReflectionMode | Behavior | Best For |
| --- | --- | --- |
| `Reflect` | Mirrors the incoming velocity using a surface normal or supplied `Normal`. | Wall bounces, shields, angled surfaces, ricochet bullets. |
| `Return` | Sends the projectile back toward the owner/source/deflector context depending on available data. | Parry returns, sword counters, enemy projectile revenge arcs. |
| `Locate` | Searches for a valid nearby target and redirects toward it. | Smart parries, target-seeking counters, redirect-to-nearest-enemy effects. |
| `Lob` | Produces an arcing redirected shot rather than a flat return. | Tossed bombs, magic rebounds, projectile volleyball nonsense. |
| Direct `Velocity` | Uses the supplied velocity directly instead of deriving one from mode math. | Full manual control when your ability already calculated the answer. |
| Direct `Direction` | Builds velocity from direction and speed settings. | Simple redirects where only aim direction changes. |

> [BlajahBean] reflect = bounce. return = no u. locate = find unlucky guy. lob = yeet.

### Deflection Parameters

| Parameter | Description |
| --- | --- |
| `Deflector` | Instance causing the deflection. Often Character, Player, shield part, or weapon. |
| `ReflectionMode` | Deflection type. Common values are `Reflect`, `Return`, `Locate`, and `Lob`. |
| `Velocity` | Direct velocity override. Highest control, least automatic math. |
| `Direction` | Desired outgoing direction. Usually combined with current speed or speed multipliers. |
| `Normal` | Surface normal used for reflection. Important for `Reflect`. |
| `Position` | New projectile position after deflect. Useful to push it out of a surface. |
| `CenterPosition` | Reference center for area/return math. |
| `IgnoreList` | New raycast ignore list after deflect. Add the deflecting part/source to prevent instant re-hit. |
| `Acceleration` | New acceleration after deflect. Useful for lobbed or gravity-influenced returns. |
| `SpeedMultiplier` | Multiplies current speed on output. |
| `MaxDeflectSpeed` | Clamps output speed so parries do not become railguns by accident. |
| `SpreadAngle` | Adds random angular spread to output direction. |
| `ReflectionBias` | Blends reflection direction against another reference direction when supported. |
| `ChangeOwner` | Transfers owner to the deflector. Use for damage credit after parry. |
| `OwnerSearchRange` | Search radius for locate-style target selection. |
| `LocateTargetValidator` | Function used to accept/reject locate targets. |
| `IsTeammate` | Team filter helper used by locate/return logic when supplied. |
| `RespectDeflectable` | Requires target/projectile source to be deflectable when true. |
| `RequireTouch` | Requires closer/touch-like validation for the area or deflect context. |
| `Blacklist` | Extra instances/models ignored by locate or deflect search logic. |

## DeflectArea

```lua
ProjectileCore.DeflectArea({
	["Center"]           = RootPart;
	["Range"]            = 22;
	["Deflector"]        = Character;
	["Character"]        = Character;
	["ReflectionMode"]   = "Return";
	["SpeedMultiplier"]  = 1.12;
	["DeflectCooldown"]  = 0.12;
	["AreaGroup"]        = "ParryWindow";
	["AreaPriority"]     = 1;
	["AreaLockDuration"] = 0.16;
	["ShouldBroadcast"]  = true;
});
```

`AreaGroup`, `AreaPriority`, and `AreaLockDuration` prevent ovverlapping parry or repulse areas from repeatedly replacing each other on the same projectile. Higher priority can replace lower priority.

### Area Grouping

| Setting | Description |
| --- | --- |
| `AreaGroup` | Logical group name for an area controller, such as `ParryWindow` or `Blackhole`. |
| `AreaPriority` | Higher priority can replace lower priority effects. Equal priority usually respects lock/cooldown rules. |
| `AreaLockDuration` | Time window where the projectile resists replacement by the same/lower priority area. |
| `DeflectCooldown` | Per-projectile cooldown to avoid immediate repeated deflect spam. |

Use grouping when multiple players, hitboxes, or effects can overlap. Without grouping, two close-range effects can keep stealing the same projectile back and forth like gremlins fighting over one spoon.

## AttractArea

```lua
ProjectileCore.AttractArea({
	["Center"]          = BlackholePart;
	["Range"]           = 60;
	["Strength"]        = 120;
	["Falloff"]         = 1.4;
	["Duration"]        = 0.95;
	["Method"]          = "VelocityDamp";
	["AreaGroup"]       = "Blackhole";
	["AreaPriority"]    = 2;
	["AffectSimulated"] = true;
	["AffectPhysical"]  = false;
	["ShouldBroadcast"] = true;
});
```

`VelocityDamp` redirects current velocity toward the center. It feels like steering.

`AccelerationDamp` applies acceleration toward the center. It feels more like gravity or blackhole pull.

Attraction does not change owner and does not implicitly enabledeflection.

### Attraction Types

| Method | Behavior | Best For |
| --- | --- | --- |
| `VelocityDamp` | Continuously bends current velocity toward the center. | Readable blackholes, gentle pulls, projectiles that should keep their speed identity. |
| `AccelerationDamp` | Adds acceleration toward the center over time. | Gravity wells, heavier pulls, slingshot behavior, more physical-looking motion. |

### Attract Parameters

| Parameter | Description |
| --- | --- |
| `Center` | Position source. Can be Vector3, CFrame, BasePart, Model, Attachment, or similar resolvable source. |
| `Range` | Radius where projectiles are affected. |
| `Strength` | Pull strength. Higher values steer/pull harder. |
| `Falloff` | Controls strength drop-off by distance. Higher values make edge pull weaker. |
| `Duration` | How long the attraction controller remains active. |
| `OneShot` | Applies attraction once instead of maintaining a controller, when supported. |
| `Method` | `VelocityDamp` or `AccelerationDamp`. |
| `AreaGroup` | Group name used for controller replacement rules. |
| `AreaPriority` | Higher priority can replace lower priority attract/area states. |
| `AreaLockDuration` | Prevents rapid replacement by same/lower priority controllers. |
| `AffectSimulated` | Applies to simulated projectiles. |
| `AffectPhysical` | Applies to physical projectiles when physical mapping is supported. |
| `ShouldBroadcast` | Sends the attraction state to client replicas for server authority. |
| `DebugVisualizeAreas` | Draws the attract radius when debug visualizers are enabled. |

Attraction stacks by replacement, not accumulation. The newest valid controller owns that projectile's active attract state unless grouping/priority prevents the replacement.

## SpawnPhysical

```lua
ProjectileCore.SpawnPhysical(ProjectileModelOrPart, {
	["Creator"]              = Character;
	["MaxDeflections"]       = 24;
	["Deflectable"]          = true;
	["AutoLifetime"]         = 8;
	["ProjectileIdentifier"] = "optional-id";
	["TimeScale"]            = 1;
	["TravelMode"]           = "Linear";
	["TravelData"]           = nil;

	["OnDeflect"] = function(ProjectileEntry, Deflector)
	end;

	["OnDestroy"] = function(ProjectileEntry, Reason : string)
	end;

	["OnOwnerChanged"] = function(ProjectileEntry, NewOwner)
	end;
});
```

Physical projectiles keep Roblox physics or movers. `SetTimeScale` can use negative values for physical behavior where supported. Simulated time scale rejects negative values.

### Physical Notes

| Topic | Behavior |
| --- | --- |
| Ownership | `SetOwner` can update damage credit or ownership metadata. |
| Deflection | Physical deflection maps into velocity/mover behavior rather than pure simulated recasting. |
| Bezier | Physical Bezier is guided steering, not teleporting along the curve. |
| Time scale | Physical time scale can support reverse/negative behavior depending on controller support. |
| Cleanup | Use callbacks and `AutoLifetime` to avoid orphaned parts. Orphaned projectiles are not a feature, they are little crimes. |

## Time Scale

```lua
ProjectileCore.SetTimeScale(Identifier, 0.2, true);
ProjectileCore.SetTimeScale(Identifier, 0, true);
ProjectileCore.SetTimeScale(Identifier, 1, true);
```

Simulated projectiles reject negative values. Physical projectiles can use reverse or negative behavior where supported.

| Scale | Simulated Behavior |
| --- | --- |
| `0` | Pauses movement and homing updates. |
| `0.5` | Moves at half speed. |
| `1` | Normal speed. |
| `2` | Moves at double speed. |
| Negative | Rejected for simulated projectiles. |

## Network Transport

```lua
ProjectileCore.SetNetworkTransport("Auto");
local NetworkInfo = ProjectileCore.GetNetworkInfo();
```

`Auto` prefers ByteNet Max when available and falls back to SoNET. ByteNet Max is expected at `ReplicatedStorage.ModuleScripts.Networking.ByteNetMax` in the current game layout.

Buffers are currently for packet/telemetry-style payloads, not live projectile entries. Live entries contain Instances, callbacks, caches, controller state, and other things that do not belong in a tiny serialized box pretending life is simple.

## Debug Visualizers

All visualizers are off by default.

```lua
["DebugVisualize"]              = true;
["DebugVisualizeCasts"]         = true;
["DebugVisualizeAreas"]         = true;
["DebugVisualizeHoming"]        = true;
["DebugVisualizerLifetime"]     = 0.15;
["DebugVisualizerColor"]        = Color3.fromRGB(0, 170, 255);
["DebugVisualizerTransparency"] = 0.65;
["DebugVisualizerAlwaysOnTop"]  = true;
```

Use cast visualizers to verify ray/sphere/shapecast sweeps. Use area visualizers for `DeflectArea` and `AttractArea`. Use homing visualizers to see acquire range and current target line.

### Visualizer Types

| Visualizer | Shows |
| --- | --- |
| Cast | Ray, spherecast, or shapecast sweep path. |
| Area | DeflectArea or AttractArea radius/volume. |
| Homing | Acquisition range and current target line. |
| Shape | Block/shapecast bounds where supported. |

## Server-Authority Example

```lua
--// SERVER

local ReplicatedStorage = game:GetService("ReplicatedStorage");

local ProjectileCore = require(
	ReplicatedStorage["ModuleScripts"]["ProjectileCoreSystem"]:WaitForChild("ProjectileCore")
);

ProjectileCore.RegisterProjectile("Rocket", {
	["CreateProjectilePart"] = false;
	["ReplicateToClients"]   = true;
	["SpherecastRadius"]     = 4;

	["ProjectileSettings"] = {
		["MaxFlyTime"]        = 8;
		["MaxFlyDistance"]    = 2500;
		["MaxPiercesPerStep"] = 8;
	};
});

local function FireRocket(Player : Player, StartPosition : Vector3, TargetPosition : Vector3)
	local Character = Player["Character"];
	if not Character then
		return nil;
	end;

	local FireDirection = TargetPosition - StartPosition;
	if FireDirection.Magnitude <= 1e-4 then
		FireDirection = Vector3.new(0, 0, -1);
	end;

	return ProjectileCore.SpawnSimulated("Rocket", {
		["Position"]     = StartPosition;
		["Velocity"]     = FireDirection.Unit * 360;
		["Acceleration"] = Vector3.zero;
		["Creator"]      = Character;
		["IgnoreList"]   = { Character; };
		["Authority"]    = "Server";

		["OnServerHit"] = function(ProjectileEntry, HitResult : RaycastResult?)
			local HitInstance = HitResult and HitResult["Instance"];
			local HitModel = HitInstance and HitInstance:FindFirstAncestorWhichIsA("Model");
			local Humanoid = HitModel and HitModel:FindFirstChildWhichIsA("Humanoid");

			if Humanoid then
				LoadedModules.TakeDamage(...);
			end;
		end;
	});
end;
```

```lua
--// CLIENT

local ReplicatedStorage = game:GetService("ReplicatedStorage");
local Workspace = game:GetService("Workspace");

local ProjectileCore = require(
	ReplicatedStorage["ModuleScripts"]["ProjectileCoreSystem"]:WaitForChild("ProjectileCore")
);

local VisualFolder = Workspace:FindFirstChild("ProjectileVisuals") or Instance.new("Folder");
VisualFolder["Name"] = "ProjectileVisuals";
VisualFolder["Parent"] = Workspace;

ProjectileCore.RegisterProjectile("Rocket", {
	["ProjectilePartParent"]    = VisualFolder;
	["CreateProjectilePart"]    = true;
	["ReplicateToClients"]      = false;
	["ProjectileCacheWarmCount"] = 32;

	["EffectPolicy"] = {
		["ManagedClasses"] = { "ParticleEmitter"; "Trail"; "Light"; };
		["AutoEnable"]     = true;
	};
});
```

## Client-Authority Cosmetic Example

```lua
ProjectileCore.SpawnSimulated("Spark", {
	["Position"]           = Muzzle["WorldPosition"];
	["Velocity"]           = Muzzle["WorldCFrame"]["LookVector"] * 260;
	["Acceleration"]       = Vector3.new(0, -8, 0);
	["Creator"]            = Character;
	["IgnoreList"]         = { Character; Tool; };
	["Authority"]          = "Client";
	["SkipReplication"]    = true;
	["ReplicateToClients"] = false;
	["SpherecastRadius"]   = 2;

	["OnClientHit"] = function(ProjectileEntry, HitResult : RaycastResult?)
		print("Local cosmetic hit", HitResult and HitResult["Instance"]);
	end;
});
```

## Performance Notes

- Prefer server authority for damage, PvP, NPC combat like Plyr vs Sky Drake, and real hit handling.
- Use `CreateProjectilePart = false` on server definitions when clients render visuals.
- Use cached visual projectiles and warm the cache to expected burst counts.
- Prefer spherecasts for fast/fatass bullets instead of large overlap loops.
- Keep spherecast radius as small as gameplay allows.
- Tune replication intervals for homing/Bezier visuals instead of over-sending every frame.
- Keep `AutoAcquireInterval` reasonable under high NPC counts.
- Use grouped `DeflectArea` and `AttractArea` to avoid controller churn (*Polar here, but yeah this is important for FPS safety*).
- Keep debug visualizers disabled in production benchmarks.
- Avoid expensive work in `OnPositionChange` because this is called a lot of times to keep motion smooth.
- Use `PatchProjectile` for live simulated changes instead of destroy/respawn churn.
