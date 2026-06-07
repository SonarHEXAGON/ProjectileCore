# ProjectileCore API Reference

Long-form reference for ProjectileCore: registration, simulated projectiles, physical projectiles, homing, Bezier travel, deflection, attraction, area controllers, control immunity, networking, visual setup, and performance behavior.

This is fine.

## Require Path

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage");

local ProjectileCore = require(ReplicatedStorage["ModuleScripts"]["ProjectileCoreSystem"]:WaitForChild("ProjectileCore"));
```

## Authority Model

`Authority = "Server"` means the server owns hit detection, deflection, redirects, ownership, time-scale changes, damage callbacks, and destroy state. Clients can register the same projectile name with visual settings so replicated spawns render locally.

`Authority = "Client"` means the local machine owns simulation and hit callbacks. Use this for cosmetic projectiles, local previews, or benchmark tests. Do not use client authority for real damage unless the server validates the result.

Server-authority client replicas are visual-only. They render spawn, deflect, redirect, patch, state-sync, area-controller, and destroy updates from the server.

## Package Layout

```text
ReplicatedStorage
  ModuleScripts
    ProjectileCoreSystem
      ProjectileCore
        Network
        Objects
          Projectiles
        Physical
        Shared
        Simulated
        Simulators
```

Projectile presets normally live at:

```text
ProjectileCore.Objects.Projectiles.<ProjectileName>
```

A preset can be a direct `BasePart`, a `Model` with `PrimaryPart`, or a `ModuleScript` returning projectile/cache data. Unless somebody shoved the template in a random folder and then asked why cache is nil.

## Core API

```text
ProjectileCore.RegisterProjectile(ProjectileName, ProjectileDefinition?) -> RegisteredProjectileDefinition
ProjectileCore.SpawnSimulated(ProjectileName, SpawnInformation) -> Identifier, Entry
ProjectileCore.SpawnPhysical(ProjectileInstance, SpawnInformation) -> Identifier, Entry
ProjectileCore.Deflect(Identifier, DeflectionParams, ShouldBroadcast?)
ProjectileCore.Redirect(Identifier, NewVelocity, NewPosition?, NewAcceleration?, NewIgnoreList?, ShouldBroadcast?)
ProjectileCore.DeflectArea(DeflectionParams) -> { BasePart }
ProjectileCore.AttractArea(AttractParams) -> { ProjectileEntry }
ProjectileCore.CreateAreaController(ControllerInformation) -> AreaControllerHandle
ProjectileCore.SetControlImmunity(Identifier, ImmunityPatch, ShouldBroadcast?)
ProjectileCore.SetHoming(Identifier, HomingData?, ShouldBroadcast?)
ProjectileCore.SetOwner(Identifier, NewOwner?)
ProjectileCore.GetOwner(Identifier) -> Instance?
ProjectileCore.SetTimeScale(Identifier, SpeedScale, ShouldBroadcast?)
ProjectileCore.PatchProjectile(Identifier, PatchData, ShouldBroadcast?)
ProjectileCore.Cancel(Identifier, Reason?)
ProjectileCore.DestroyProjectile(Identifier, Reason?)
ProjectileCore.GetData(Identifier) -> ProjectileEntry?
ProjectileCore.GetAll() -> { [string] : ProjectileEntry }
ProjectileCore.IsActive(Identifier) -> boolean
ProjectileCore.GetOwnerFromPart(Part) -> Instance?
ProjectileCore.SetNetworkTransport("Auto" | "ByteNet" | "SoNET")
ProjectileCore.GetNetworkTransport() -> table
ProjectileCore.GetNetworkInfo() -> table
```

| Function | What It Does | Notes |
| --- | --- | --- |
| `RegisterProjectile` | Stores defaults for one projectile name. | Safe on server and client. Same name should be registered on both sides for replicated visuals. |
| `SpawnSimulated` | Creates a fixed-step simulated projectile. | Best for bullets, missiles, Bezier, homing, droplets, and controlled orbit shots. |
| `SpawnPhysical` | Registers an existing BasePart or Model. | Best for Roblox physics/mover-driven objects. |
| `Deflect` | Redirects one active projectile. | Supports reflect, return, locate, lob, or direct velocity. |
| `Redirect` | Recasts from a new position/velocity/acceleration. | Good for delayed launch, manual course correction, or post-phase arming. |
| `DeflectArea` | Deflects projectiles inside a volume. | Uses group/priority locks to prevent controller slap-fights. |
| `AttractArea` | Applies vortex/blackhole steering. | Can affect simulated and physical entries. |
| `CreateAreaController` | Persistent Orbit/Tornado/FrozenMoment controller. | Uses Motion and Time conflict lanes. |
| `SetControlImmunity` | Runtime patch for CC immunity. | Releases blocked area-controller captures when immunity changes. |
| `SetHoming` | Adds/replaces/disables homing. | Can be delayed after spawn. |
| `SetOwner` / `GetOwner` | Owner/damage-credit management. | Use this instead of raw entry mutation. |
| `SetTimeScale` | Simulation speed scaling. | Fully paused projectiles skip movement raycasts. |
| `PatchProjectile` | Validated live simulated field patching. | Do not mutate random nested fields unless you enjoy cursed bugs. |
| `Cancel` / `DestroyProjectile` | Cleanup/despawn. | `DestroyProjectile` is an alias for intent clarity. |

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

	["ControlImmunity"] = {
		["Attract"] = true;
	};

	["ProjectileSettings"] = {
		["MaxFlyTime"]     = 10;
		["MaxFlyDistance"] = 5000;
		["RaysPerMove"]    = 8;
	};
});
```

### Registration Settings

| Setting | Type | Default | Description |
| --- | --- | --- | --- |
| `ProjectileTemplate` | `BasePart` or `Model` | preset lookup | Visual template used for projectile parts. |
| `ProjectileCache` | cache object | internal cache | Optional explicit object cache. |
| `ProjectileCacheWarmCount` | `number` | `10` | Number of cached objects pre-created. |
| `ProjectilePartParent` | `Instance` | `Workspace` | Parent for active projectile visuals. |
| `CreateProjectilePart` | `boolean` | `true` | Creates visual part/model when true. |
| `ProjectileSettings` | `table` | default settings | Lifetime and cast stepping settings. |
| `EffectPolicy` | `table` | nil | Managed effect toggling policy. |
| `PrepareProjectilePart` | `function` | nil | Runs after initial CFrame/transform is valid. |
| `OnPartEnd` | `function` | nil | Custom visual cleanup before cache return/destroy. |
| `ControlImmunity` | `table` | nil | Default guardrail against deflect/attract/area control systems. |
| `SpherecastRadius` | `number` | nil | Uses spherecast when greater than zero. |
| `UseShapecast` | `boolean` | `false` | Uses projectile part/model shape where supported. |
| `IgnoreWater` | `boolean` | `false` | Raycast water behavior. |
| `AllowRaycastDeflection` | `boolean` | `true` | Allows automatic `CanDeflect` hit deflection. |
| `MaxDeflections` | `number` | `24` | Maximum deflections before final handling. |
| `ReplicateToClients` | `boolean` | `true` | Server broadcasts simulated spawns/runtime changes. |
| `Authority` | `"Server"` or `"Client"` | context-based | Simulation owner. |
| `TimeScale` | `number` | `1` | Initial simulation speed multiplier. |
| `TravelMode` | `string` | `"Linear"` | `Linear`, `QuadraticBezier`, or `CubicBezier`. |
| `TravelData` | `table` | nil | Bezier target/control/duration settings. |
| `HomingData` | `table` | nil | Initial homing settings. |
| `UserData` | `table` | nil | Custom data copied onto entries. |

### Callbacks

| Callback | Runs On | Purpose |
| --- | --- | --- |
| `PrepareProjectilePart(Entry, ProjectilePart, Context)` | Side where visual part exists | Final setup after initial transform/CFrame is correct and before managed effects turn on. |
| `OnPartEnd(Part, Reason, Cleanup)` | Side where visual part exists | Impact hold, fade, shrink, trail cleanup, cache return timing. |
| `OnServerHit(Entry, HitResult, Reason)` | Server | Authoritative damage/explosion/status logic. |
| `OnClientHit(Entry, HitResult, Reason)` | Client | Cosmetic impact response or client-authority hit behavior. |
| `OnServerDestroy(Entry, Reason)` | Server | Cleanup after cancel, timeout, distance limit, or hit. |
| `OnClientDestroy(Entry, Reason)` | Client | Visual cleanup for replicated/client-owned projectiles. |
| `OnPositionChange(Entry, Position, LastPosition)` | Simulation owner/visual replica | Lightweight orientation or visual follow logic. |

Keep `OnPositionChange` lean. If it allocates tables, scans folders, emits particles, and calls three remotes per step, the profiler will arrive with a clipboard.

## SpawnSimulated

```lua
local Identifier, ProjectileEntry = ProjectileCore.SpawnSimulated("Bullet", {
	["Position"]         = Muzzle["WorldPosition"];
	["Velocity"]         = Muzzle["WorldCFrame"]["LookVector"] * 420;
	["Acceleration"]     = Vector3.zero;
	["Creator"]          = Character;
	["IgnoreList"]       = { Character; Tool; };
	["Authority"]        = "Server";
	["SpherecastRadius"] = 2.5;
	["Angles"]           = CFrame.Angles(0, math.rad(90), 0);
});
```

### Spawn Settings

| Setting | Type | Description |
| --- | --- | --- |
| `Position` / `StartingPosition` | `Vector3` | Initial world position. |
| `Velocity` | `Vector3` | Initial velocity. If absent, `Direction` plus `Speed` is used. |
| `Direction` | `Vector3` | Direction used with `Speed`. |
| `Speed` | `number` | Magnitude used with `Direction`. |
| `Acceleration` | `Vector3` | Constant acceleration, such as gravity. |
| `Creator` | `Instance` | Owner source, usually Player, Character, Tool, or NPC Model. |
| `IgnoreList` | `{ Instance }` | Cast exclusions. |
| `Authority` | `"Server"` or `"Client"` | Explicit authority override. |
| `ProjectileIdentifier` | `string` | Optional custom identifier. |
| `SkipReplication` | `boolean` | Prevents server spawn broadcast for this projectile. |
| `UserData` | `table` | Custom data stored on `Entry.UserData`. |
| `ControlImmunity` | `table` | Spawn-level immunity override/patch over definition defaults. |
| `HomingData` | `table` | Initial homing state. |
| `TravelMode` | `string` | Travel mode for this shot. |
| `TravelData` | `table` | Bezier/travel data for this shot. |
| `Angles` | `CFrame` | Orientation offset applied to direction-facing visuals when wired into the simulated data. |
| `AttractState` | `table` | Advanced attraction state; usually use `AttractArea`. |
| `CanPierce` | `function` | Return true to continue through a hit. |
| `RaycastFunction` | `function` | Custom cast function override. |

The returned `Identifier` is the safe handle for later APIs. The returned `Entry` is useful in callbacks and inspection, but runtime mutation should go through `PatchProjectile`, `Redirect`, `SetHoming`, `SetOwner`, or `SetControlImmunity`.

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
| `MaxPiercesPerStep` | `number` | Maximum hit/pierce checks per fixed step. |

## Close-Hit Snap

Close-hit snap is intended to resolve very short spawn-time hits before the projectile gets its first rendered travel frame.

```lua
ProjectileCore.RegisterProjectile("PointBlankBolt", {
	["SnapHitEnabled"]  = true;
	["SnapHitDistance"] = 12;
	["SnapHitTime"]     = 0.05;
});
```

| Setting | Type | Intended Default | Description |
| --- | --- | --- | --- |
| `SnapHitEnabled` | `boolean` | `true` through ProjectileCore | Enables first-step close-hit precheck. |
| `SnapHitDistance` | `number` | `12` | Max studs for close-hit snap. |
| `SnapHitTime` | `number` | `0.05` | Predicted travel window for the precheck. |

Custom `CanPierce` projectiles should not use snap by default unless the source explicitly supports it.

## EffectPolicy And PrepareProjectilePart

ProjectileCore can keep managed effects disabled until the initial CFrame/transform is valid, then run `PrepareProjectilePart`, then enable allowed effects.

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

```lua
["PrepareProjectilePart"] = function(ProjectileEntry, ProjectilePart : BasePart, Context)
	ProjectilePart["Transparency"] = 0;
	ProjectilePart["Color"]        = Color3.fromRGB(255, 230, 120);
end;
```

`PrepareProjectilePart` is the correct place to enable mesh/material/visibility setup. Do not manually enable trails at the template origin before ProjectileCore sets the first transform unless you want a trail line from Narnia or wtever.

## HomingData

```lua
ProjectileCore.SetHoming(Identifier, {
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
		return TargetHumanoid["Health"] > 0 and TargetCharacter ~= OwnerCharacter;
	end;
}, true);
```

| Setting | Description |
| --- | --- |
| `Enabled` | False disables homing without clearing all data. |
| `Target` | Instance target, usually BasePart, Model, or Attachment. |
| `TargetPosition` | Static world target position. |
| `Offset` | Offset added to resolved target position. |
| `TurnSpeed` | Steering strength. Higher values turn harder. |
| `MinimumDistance` | Distance where steering stops. |
| `AutoAcquire` | Searches humanoids for a target. |
| `AutoAcquireRange` | Max acquisition range. |
| `AutoAcquireInterval` | Retarget check interval. Keep this sane. |
| `AutoAcquireRetarget` | Allows target replacement over time. |
| `RequireLineOfSight` | Requires clear ray to target. Extra ray cost. |
| `AutoAcquireValidator` | Team checks, forcefields, boss immunity, no-projectile-targeting, etc. |

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
| `TargetOffset` | Offset added to target. |
| `TrackTarget` | Updates end position as an Instance target moves. |
| `ControlPoint` | Quadratic control point. |
| `ControlPointA` / `ControlPointB` | Cubic control points. |
| `Duration` | Seconds to traverse the curve. |
| `Speed` | Used to derive duration/tangent velocity when duration is absent. |
| `CompletionMode` | `Linear`, `Destroy`, `Homing`, or `ContinueBezier`. |
| `ContinueDuration` | Segment duration when continuing Bezier toward a moving target. |
| `HomingTurnSpeed` | Turn speed when completion mode becomes homing. |
| `HomingMinimumDistance` | Minimum distance for post-curve homing. |
| `DestroyOnComplete` | Shortcut for `CompletionMode = "Destroy"`. |

`CompletionMode = "Homing"` is the clean way to curve first, then track directly afterward.

## ControlImmunity

`ControlImmunity` blocks ProjectileCore CC/control systems without disabling normal hit detection, damage callbacks, cleanup, or collision behavior.

```lua
ProjectileCore.RegisterProjectile("BossMeteor", {
	["ControlImmunity"] = {
		["Deflect"]  = true;
		["Attract"]  = true;
		["AreaTime"] = true;
	};
});

local Identifier = ProjectileCore.SpawnSimulated("BossMeteor", {
	["Position"] = StartPosition;
	["Velocity"] = Direction.Unit * 280;
	["Creator"]  = BossCharacter;

	["ControlImmunity"] = {
		["Deflect"] = false;
	};
});

ProjectileCore.SetControlImmunity(Identifier, {
	["All"] = true;
}, true);
```

| Key | Blocks |
| --- | --- |
| `All` | All ProjectileCore control systems. |
| `Deflect` | Direct/raycast deflect and fallback for DeflectArea. |
| `DeflectArea` | Area deflect only, falling back to `Deflect`. |
| `Attract` | AttractArea control. |
| `AreaController` | All persistent area-controller modes. |
| `AreaMotion` | Orbit and Tornado lanes. |
| `AreaTime` | FrozenMoment lane. |
| `Orbit` | Orbit mode only. |
| `Tornado` | Tornado mode only. |
| `FrozenMoment` | FrozenMoment mode only. |

Definition and spawn immunity tables merge. Spawn values take priority; `false` explicitly removes/overrides a default immunity. Runtime updates can release a projectile from a controller if the new immunity blocks the current lane.

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

| ReflectionMode | Behavior |
| --- | --- |
| `Reflect` | Mirrors incoming velocity using a surface normal or supplied `Normal`. |
| `Return` | Sends the projectile back toward owner/source/deflector context. |
| `Locate` | Searches for a valid nearby target and redirects toward it. |
| `Lob` | Produces an arcing redirected shot. |
| Direct `Velocity` | Uses supplied velocity directly. |
| Direct `Direction` | Builds velocity from direction and speed settings. |

`ControlImmunity.Deflect` rejects direct and raycast deflection. `ControlImmunity.DeflectArea` rejects only area deflect, unless `Deflect` or `All` also applies.

> [BlajahBean] reflect = bounce. return = no u. locate = find unlucky guy. lob = yeet.

## DeflectArea

```lua
ProjectileCore.DeflectArea({
	["Center"]           = RootPart;
	["Range"]            = 22;
	["Shape"]            = "Sphere";
	["Deflector"]        = Character;
	["ReflectionMode"]   = "Return";
	["SpeedMultiplier"]  = 1.12;
	["DeflectCooldown"]  = 0.12;
	["AreaGroup"]        = "ParryWindow";
	["AreaPriority"]     = 1;
	["AreaLockDuration"] = 0.16;
	["ShouldBroadcast"]  = true;
});
```

Supported area shapes include `Sphere`, `Box`, `Cylinder`, `Disc`, `Cone`, `Pyramid`, `Capsule`, and `HalfSphere`. FAR WARNING tho, `Disc` is treated as a thin cylinder, not an infinitely thin math plane that makes physics cry.

| Setting | Description |
| --- | --- |
| `Center` / `CenterPosition` | Position source. |
| `Range` / `Radius` | Radius for spherical/cylindrical shapes. |
| `Shape` / `AreaShape` | AreaQuery capture shape. |
| `CFrame` / `AreaCFrame` | Oriented area frame. |
| `Size` | Box/block-style size. |
| `Height` / `Thickness` | Cylinder/disc/capsule height. |
| `AreaGroup` | Logical group name for replacement rules. |
| `AreaPriority` | Higher priority beats lower priority. |
| `AreaLockDuration` | Prevents rapid replacement. |
| `DeflectCooldown` | Per-projectile deflect cooldown. |

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

| Method | Behavior |
| --- | --- |
| `VelocityDamp` | Bends current velocity toward the center. |
| `AccelerationDamp` | Adds acceleration toward the center over time. |

AttractArea skips entries with `ControlImmunity.Attract` or `ControlImmunity.All`.

## AreaController

`CreateAreaController` creates a persistent controller that scans active projectiles at a bounded rate and steps captured projectiles at controller rate. It uses velocity/time-scale control instead of direct teleporting, because teleport trails look like a bug report with particles.

```lua
local Controller = ProjectileCore.CreateAreaController({
	["Mode"]            = "Orbit";
	["Center"]          = RootPart;
	["AreaPriority"]    = 5;
	["AffectSimulated"] = true;
	["AffectPhysical"]  = true;
	["ShouldBroadcast"] = true;

	["Area"] = {
		["Shape"] = "Sphere";
		["Range"] = 32;
	};

	["Behavior"] = {
		["Lifetime"]     = 5;
		["ReleaseMode"]  = "PreserveVelocity";
		["FacePath"]     = true;
		["FacePathTime"] = 0.08;
	};

	["Motion"] = {
		["Axis"]            = Vector3.new(0, 1, 0);
		["Radius"]          = { ["Min"] = 8; ["Max"] = 14; };
		["Height"]          = { ["Min"] = 0; ["Max"] = 12; };
		["Cycles"]          = 2;
		["HoverHeight"]     = 2;
		["SpinSpeed"]       = { ["Min"] = 6; ["Max"] = 10; };
		["CorrectionSpeed"] = 12;
	};

	["Shape"] = {
		["Shape"]        = "Disc";
		["ShapePartial"] = 1;
		["ShapeStyle"]   = "Surface";
		["ShapeInOut"]   = "InAndOut";
	};
});
```

### Handle Methods

| Method | Description |
| --- | --- |
| `Destroy(Reason?)` | Releases captured entries and stops the controller. |
| `Pause()` | Stops scanning/stepping without destroying state. |
| `Resume()` | Resumes a paused controller. |
| `SetCenter(Center)` | Updates center source. |
| `SetPriority(Priority)` | Updates priority for conflict checks. |
| `GetCaptured()` | Returns captured entries. |
| `IsActive()` | Returns active state. |

### Modes

| Mode | Lane | Behavior |
| --- | --- | --- |
| `Orbit` | Motion | Orbits around a center using tangent velocity and correction. |
| `Tornado` | Motion | Orbit plus vertical swirl/height shaping. |
| `FrozenMoment` | Time | Eases projectiles into slow/frozen time and restores prior time scale on exit. |

### FrozenMoment

```lua
local Controller = ProjectileCore.CreateAreaController({
	["Mode"] = "FrozenMoment";
	["Center"] = FreezePart;

	["Area"] = {
		["Shape"] = "Sphere";
		["Range"] = 50;
	};

	["Behavior"] = {
		["Lifetime"]            = 3;
		["SlowDownMultiplier"]  = 1;
		["ProjectileEntryTime"] = 0.35;
		["ProjectileExitTime"]  = 0.35;
	};
});
```

`SlowDownMultiplier = 0` means normal speed. `SlowDownMultiplier = 1` means halted. Internally this maps to `TimeScale = 1 - SlowDownMultiplier`. Fully paused movement skips raycasts for optimization.

### Conflict Rules

AreaController maintains two conflict lanes:

| Lane | Modes |
| --- | --- |
| `Motion` | `Orbit`, `Tornado` |
| `Time` | `FrozenMoment` |

Higher `AreaPriority` wins within a lane. Equal priority keeps the current controller to prevent jitter flicker. `AreaMotion`, `AreaTime`, and mode-specific immunity keys can reject capture or release already captured entries.

## SpawnPhysical

```lua
local Identifier, Entry = ProjectileCore.SpawnPhysical(ProjectilePart, {
	["Creator"]      = Character;
	["Deflectable"]  = true;
	["AutoLifetime"] = 5;
	["TimeScale"]    = 1;

	["PhysicalCast"] = {
		["Enabled"]            = true;
		["Mode"]               = "Spherecast";
		["Radius"]             = 3;
		["IgnoreOwner"]        = true;
		["IgnoreWater"]        = true;
		["RespectCanCollide"]  = false;
		["CancelOnHit"]        = true;
	};

	["OnPhysicalHit"] = function(ProjectileEntry, HitResult : RaycastResult, CastMode : string)
		print(CastMode, HitResult["Instance"]);
	end;

	["OnDestroyed"] = function()
		print("Physical projectile ended");
	end;
});
```

### Physical Settings

| Setting | Description |
| --- | --- |
| `Creator` | Owner/creator source. |
| `UserData` | Custom entry data. |
| `ControlImmunity` | Guardrail table for control systems. |
| `MaxDeflections` | Maximum deflection count. |
| `Deflectable` | Whether direct/area deflect can affect it. Backward-compatible with old behavior. |
| `AutoLifetime` | Lifetime before automatic cleanup. |
| `ProjectileIdentifier` | Optional custom id. |
| `TimeScale` | Physical time scaling through cached movers/velocities. |
| `TravelMode` / `TravelData` | Guided physical Bezier/linear steering. |
| `PhysicalCast` | Optional cast config for movement hit detection. |
| `OnPhysicalHit` | Callback when PhysicalCast detects a terminal hit. |
| `OnPositionChange` | Callback for physical movement stepping when wired. |

### PhysicalCast Settings

| Setting | Description |
| --- | --- |
| `Enabled` | Enables movement casts. |
| `Mode` / `CastMode` | `Raycast`, `Spherecast`, `Blockcast`, or `Shapecast`. Also accepts `Ray`, `SphereCast`, `BlockCast`, `ShapeCast`. |
| `Radius` / `SpherecastRadius` | Spherecast radius. |
| `Size` / `BlockSize` | Blockcast size. |
| `CFrame` | Optional blockcast frame. |
| `Shape` / `ShapePart` | Shapecast part. |
| `IgnoreList` | Extra cast exclusions. |
| `IgnoreOwner` | Adds owner/original owner to exclusions. |
| `IgnoreWater` | Raycast water behavior. |
| `RespectCanCollide` | Raycast collision-respect override. |
| `CanPierce` | Return true to keep flying after the hit. |
| `CancelOnHit` | Defaults to true. Set false to keep active after callback. |

Use `PhysicalCast` when `.Touched` is not precise enough. `.Touched` can still be useful for gimmicks, but for projectile truth it behaves like a witness who changes their story.

## Time Scale

```lua
ProjectileCore.SetTimeScale(Identifier, 0.2, true);
ProjectileCore.SetTimeScale(Identifier, 0, true);
ProjectileCore.SetTimeScale(Identifier, 1, true);
```

| Scale | Simulated Behavior |
| --- | --- |
| `0` | Pauses movement, homing, and movement raycasts. |
| `0.5` | Moves at half speed. |
| `1` | Normal speed. |
| `2` | Moves at double speed. |
| Negative | Rejected for simulated projectiles. |

Physical projectiles may support reverse/negative behavior depending on the mover/time-scale path.

## PatchProjectile

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
	["ControlImmunity"]         = { ["Attract"] = true; };
	["EffectsEnabled"]          = true;
	["ReplicationSyncInterval"] = 0.05;
	["ReplicationSmoothTime"]   = 0.1;
	["ReplicationSnapDistance"] = 70;

	["HomingData"] = {
		["Enabled"]   = true;
		["Target"]    = TargetPart;
		["TurnSpeed"] = 10;
	};
}, true);
```

Patchable fields include velocity, acceleration, homing data, control immunity, ignore list, ray settings, time scale, effect policy, effects enabled, and replication correction settings. Patching does not allow identifier, projectile type, template/cache source, or ownership changes. Use `SetOwner` for ownership.

## Delayed True Projectile Patterns

### Orbit Then Launch

```lua
local Identifier = ProjectileCore.SpawnSimulated("OrbitingShuriken", {
	["Position"] = SpawnPosition;
	["Velocity"] = Vector3.zero;
	["Creator"]  = Character;
	["Authority"] = "Server";

	["ControlImmunity"] = {
		["Deflect"] = true;
	};
});

local Controller = ProjectileCore.CreateAreaController({
	["Mode"]   = "Orbit";
	["Center"] = CharacterRoot;

	["Area"] = {
		["Shape"] = "Sphere";
		["Range"] = 18;
	};

	["Motion"] = {
		["Radius"]    = 10;
		["SpinSpeed"] = 8;
		["Cycles"]    = 3;
	};
});

task.delay(5, function()
	Controller:Destroy("Launch");

	if ProjectileCore.IsActive(Identifier) then
		ProjectileCore.SetControlImmunity(Identifier, {
			["Deflect"] = false;
		}, true);

		ProjectileCore.Redirect(Identifier, LaunchVelocity, nil, Vector3.zero, { Character; }, true);
	end;
end);
```

### Blender Bolts

Use `CanPierce` to phase through entities during the blender phase, then switch to terrain-exploding behavior when redirected inward.

```lua
local Identifier = ProjectileCore.SpawnSimulated("MiniBolt", {
	["Position"] = TargetCenter;
	["Velocity"] = InitialOutwardVelocity;
	["Creator"]  = Character;

	["CanPierce"] = function(ProjectileEntry, HitResult : RaycastResult) : boolean
		local HitModel = HitResult["Instance"]:FindFirstAncestorWhichIsA("Model");
		local Humanoid = HitModel and HitModel:FindFirstChildWhichIsA("Humanoid");
		return Humanoid ~= nil;
	end;
});
```

## Network Transport

```lua
ProjectileCore.SetNetworkTransport("Auto");
local NetworkInfo = ProjectileCore.GetNetworkInfo();
```

`Auto` prefers ByteNet Max when available and falls back to SoNET. ByteNet Max is expected at `ReplicatedStorage.ModuleScripts.Networking.ByteNetMax` in the current game layout.

Buffers are for packet/telemetry payloads, not live projectile entries. Live entries contain Instances, callbacks, caches, controller state, and other objects that do not belong in a tiny serialized box pretending life is simple.

## Debug Visualizers

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

Debug visualizers are off by default. Keep them off in production benchmarks unless you want the debug layer itself to become the benchmark.

## Performance Notes

- Prefer server authority for damage, PvP, NPC combat, and real hit handling.
- Use `CreateProjectilePart = false` on server definitions when clients render visuals.
- Warm projectile caches to expected burst counts.
- Prefer spherecasts for high-speed or wide projectiles.
- Keep `AutoAcquireInterval` reasonable under high NPC counts.
- Use grouped `DeflectArea` and `AttractArea` to avoid controller churn.
- Use `ControlImmunity` for special projectiles instead of custom ignore soup.
- Use `FrozenMoment` / `SetTimeScale(0)` for pauses; fully paused projectiles skip movement casts.
- Use close-hit snap for point-blank travel only once the snap patch is present in source.
- Avoid expensive work in `OnPositionChange` because smooth motion means frequent callbacks.
- Use `PatchProjectile`, `Redirect`, and controller phases instead of destroy/respawn churn.
