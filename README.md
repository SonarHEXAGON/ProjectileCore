# ProjectileCore

> High-speed simulated projectiles, physical projectile control, homing, Bezier travel, area fields, deflection, attraction, and client visuals for CGP:R-style combat.

ProjectileCore is meant to let the server own real projectile truth while clients render something that does not look like it was dragged behind a truck. It supports simple bullets, lobbed blobs, cinematic Beziers, orbiting murder Frisbees, time-stop nonsense, physical debris, and the usual "why is this projectile doing forbidden geometry" edge cases.

## Contributors

- **@BlajahBean** - Main programmer and mathematician. Occasionally fights the formatter and somehow still lands the math.
- **@SonarHEXAGON** - Sub programmer, testing, live implementation, physical/projectile integration work, and helper for GitHub here.
- **@RalziumQUANTUM** - Formatting, comments, and GitHub documentation creator. The grammar police arrived armed.
- **@keklol0401010101010** - Creator of HomingCast.
- **@23sinek345** - Creator of SwiftCast and helped with optimization questions.

## Table Of Contents

- [Features](#features)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Core API](#core-api)
- [Area Controller](#area-controller)
- [Control Immunity](#control-immunity)
- [Close-Hit Snap](#close-hit-snap)
- [Bezier, Homing, And Angles](#bezier-homing-and-angles)
- [Physical Projectiles](#physical-projectiles)
- [Performance Notes](#performance-notes)
- [Full API Reference](#full-api-reference)

## Features

| Feature | Status | Notes |
| --- | --- | --- |
| Simulated projectiles | Supported | Fixed-step SwiftCast-backed simulation for fast/fat bullets. |
| Physical projectiles | Supported | Real BasePart/Model projectiles with movers, autocast, deflect, and lifetime control. |
| Server authority | Supported | Server owns real hit, damage, redirect, deflect, time-scale, and destroy state. |
| Client authority | Supported | Cosmetic/local-only projectiles and benchmark previews. |
| Client visual replicas | Supported | Server simulates while clients render cached visual projectiles. |
| Internal object cache | Supported | Separate caches for projectile templates/presets. |
| Spherecast / shapecast | Supported | More reliable hits for wide or high-speed projectiles. |
| Close-hit snap | Supported | Tiny spawn-time hits resolve before the first visual frame. |
| Transform warmup | Supported | Effects can stay hidden until the initial transform/CFrame is valid. |
| Deflect / DeflectArea | Supported | Direct and area parry/redirect systems. |
| AttractArea | Supported | Blackhole/vortex-style projectile steering. |
| AreaController | Supported | Persistent Orbit, Tornado, and FrozenMoment fields. |
| ControlImmunity | Supported | Projectile-level guardrails against CC-like systems. |
| Homing / auto-homing | Supported | Direct targets or closest humanoid auto-acquisition. |
| Bezier travel | Supported | Linear, QuadraticBezier, and CubicBezier travel. |
| Runtime patching | Supported | Validated live simulated state mutation. |
| Physical autocast | Supported | Ray/sphere/block/shape style checking without relying purely on `.Touched`. |
| AutoLifetime | Supported | Physical projectile cleanup finally does the damn thing. |
| Debug visualizers | Supported | Explicit opt-in cast, area, shape, and homing visualizers. |
| ByteNet transport | Supported when available | Falls back to SoNET when ByteNet is unavailable. |

## Installation

```text
ReplicatedStorage
└── ModuleScripts
    └── ProjectileCoreSystem
        └── ProjectileCore
```

Require it with:

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage");

local ProjectileCore = require(ReplicatedStorage["ModuleScripts"]["ProjectileCoreSystem"]:WaitForChild("ProjectileCore"));
```

Projectile presets usually live under:

```text
ProjectileCore.Objects.Projectiles.<ProjectileName>
```

A preset can be a direct `BasePart`, a `Model` with `PrimaryPart` or a resolvable BasePart, or a `ModuleScript` returning projectile/cache data.

## Quick Start

### Server Authority Projectile

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage");

local ProjectileCore = require(ReplicatedStorage["ModuleScripts"]["ProjectileCoreSystem"]:WaitForChild("ProjectileCore"));

ProjectileCore.RegisterProjectile("Rocket", {
	["CreateProjectilePart"]   = false;
	["ReplicateToClients"]     = true;
	["SpherecastRadius"]       = 4;
	["AllowRaycastDeflection"] = true;

	["ProjectileSettings"] = {
		["MaxFlyTime"]     = 8;
		["MaxFlyDistance"] = 2500;
		["RaysPerMove"]    = 16;
	};
});

local function FireRocket(Player : Player, StartPosition : Vector3, TargetPosition : Vector3) : string?
	local Character = Player["Character"];
	if not Character then
		return nil;
	end;

	local Direction = TargetPosition - StartPosition;
	if Direction["Magnitude"] <= 1e-4 then
		Direction = Vector3.new(0, 0, -1);
	end;

	local Identifier = ProjectileCore.SpawnSimulated("Rocket", {
		["Position"]     = StartPosition;
		["Velocity"]     = Direction.Unit * 360;
		["Acceleration"] = Vector3.zero;
		["Creator"]      = Character;
		["IgnoreList"]   = { Character; };
		["Authority"]    = "Server";

		["OnServerHit"] = function(ProjectileEntry, HitResult : RaycastResult?)
			local HitInstance = HitResult and HitResult["Instance"];
			local HitModel    = HitInstance and HitInstance:FindFirstAncestorWhichIsA("Model");
			local Humanoid    = HitModel and HitModel:FindFirstChildWhichIsA("Humanoid");

			if Humanoid then
				Humanoid:TakeDamage(30);
			end;
		end;
	});

	return Identifier;
end;
```

### Client Visual Registration

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage");
local GameSpace         = game:GetService("Workspace");

local ProjectileCore = require(ReplicatedStorage["ModuleScripts"]["ProjectileCoreSystem"]:WaitForChild("ProjectileCore"));

local VisualFolder = GameSpace:FindFirstChild("ProjectileVisuals") or Instance.new("Folder");
VisualFolder["Name"]   = "ProjectileVisuals";
VisualFolder["Parent"] = GameSpace;

ProjectileCore.RegisterProjectile("Rocket", {
	["ProjectilePartParent"]     = VisualFolder;
	["CreateProjectilePart"]     = true;
	["ReplicateToClients"]       = false;
	["ProjectileCacheWarmCount"] = 32;

	["EffectPolicy"] = {
		["ManagedClasses"] = { "ParticleEmitter"; "Trail"; "Light"; };
		["AutoEnable"]     = true;
	};

	["PrepareProjectilePart"] = function(ProjectileEntry, ProjectilePart : BasePart, Context)
		ProjectilePart["Transparency"] = 0;
	end;
});
```

`PrepareProjectilePart` runs after the initial CFrame/transform is valid and before managed effects are enabled. This avoids the classic "trail starts at 0, 0, 0 then teleports to the gun" nonsense. We have suffered. The docs remember.

## Core API

| Method | Purpose |
| --- | --- |
| `RegisterProjectile(Name, Definition?)` | Registers shared defaults for a projectile type. |
| `SpawnSimulated(Name, SpawnInfo)` | Spawns a fixed-step simulated projectile. |
| `SpawnPhysical(Projectile, SpawnInfo)` | Registers and controls a physical projectile instance. |
| `Deflect(Identifier, Params, ShouldBroadcast?)` | Deflects one active projectile. |
| `Redirect(Identifier, Velocity, Position?, Acceleration?, IgnoreList?, ShouldBroadcast?)` | Recasts a projectile from a new state. |
| `DeflectArea(Params)` | Deflects projectiles inside an area. |
| `AttractArea(Params)` | Applies blackhole-style attraction inside an area. |
| `CreateAreaController(ControllerInformation)` | Creates persistent Orbit, Tornado, or FrozenMoment fields. |
| `SetControlImmunity(Identifier, ImmunityPatch)` | Updates projectile immunity against CC-like control systems. |
| `SetHoming(Identifier, HomingData?, ShouldBroadcast?)` | Adds, changes, or disables homing. |
| `SetOwner(Identifier, Owner?)` | Changes projectile ownership. |
| `GetOwner(Identifier)` | Returns the current owner. |
| `SetTimeScale(Identifier, Scale, ShouldBroadcast?)` | Changes projectile simulation speed. |
| `PatchProjectile(Identifier, PatchData, ShouldBroadcast?)` | Live-patches validated simulated fields. |
| `Cancel(Identifier, Reason?)` | Cancels and cleans up a projectile. |
| `DestroyProjectile(Identifier, Reason?)` | Alias for `Cancel`. |
| `GetData(Identifier)` | Returns one live projectile entry. |
| `GetAll()` | Returns all active entries. |
| `IsActive(Identifier)` | Checks active state. |
| `GetOwnerFromPart(Part)` | Finds owner from a projectile part. |
| `SetNetworkTransport(Mode)` | Sets `Auto`, `ByteNet`, or `SoNET`. |
| `GetNetworkInfo()` | Returns active networking details. |

## Area Controller

`CreateAreaController` creates persistent projectile fields. It affects simulated and physical projectiles by default, scans membership at a bounded rate, and controls captured projectiles with velocity/time-scale behavior instead of raw teleporting.

```lua
local Controller = ProjectileCore.CreateAreaController({
	["Mode"]            = "FrozenMoment";
	["Center"]          = RootPart;
	["AreaPriority"]    = 10;
	["AffectSimulated"] = true;
	["AffectPhysical"]  = true;
	["ShouldBroadcast"] = true;

	["Area"] = {
		["Shape"] = "Sphere";
		["Range"] = 45;
	};

	["Behavior"] = {
		["Lifetime"]              = 2.5;
		["SlowDownMultiplier"]    = 1;
		["ProjectileEntryTime"]   = 0.35;
		["ProjectileExitTime"]    = 0.35;
		["ReleaseMode"]           = "PreserveVelocity";
	};
});
```

Orbit and Tornado use `Motion` and `Shape` data for the actual movement distribution. `Area` is capture membership. `Shape` is where the cool swirly bullshit lives. Do not swap those two unless you enjoy debugging invisible tornado math at 3 AM.

Supported controller modes:

| Mode | Behavior |
| --- | --- |
| `Orbit` | Captured projectiles orbit a position or moving part. |
| `Tornado` | Orbit-style control with vertical swirl and height/hover shaping. |
| `FrozenMoment` | Eases projectiles into slow/frozen time, then restores their previous time scale. |

## Control Immunity

`ControlImmunity` lets important projectiles opt out of ProjectileCore CC systems without disabling normal hits or collision behavior.

```lua
ProjectileCore.RegisterProjectile("BossMeteor", {
	["ControlImmunity"] = {
		["Deflect"]        = true;
		["Attract"]        = true;
		["AreaTime"]       = true;
	};
});

local Identifier = ProjectileCore.SpawnSimulated("BossMeteor", {
	["Position"] = StartPosition;
	["Velocity"] = Direction.Unit * 280;
	["Creator"]  = BossCharacter;

	["ControlImmunity"] = {
		["Deflect"] = false; --// Spawn override removes the default deflect immunity.
	};
});

ProjectileCore.SetControlImmunity(Identifier, {
	["All"] = true;
});
```

Supported keys: `All`, `Deflect`, `DeflectArea`, `Attract`, `AreaController`, `AreaMotion`, `AreaTime`, `Orbit`, `Tornado`, and `FrozenMoment`.

## Close-Hit Snap

Close-hit snap resolves very short spawn-time hits before the projectile gets a rendered travel frame. This removes useless 1-frame bullets and trail jitter when the muzzle is basically kissing a wall.

```lua
ProjectileCore.RegisterProjectile("PointBlankBolt", {
	["SnapHitEnabled"]  = true;
	["SnapHitDistance"] = 12;
	["SnapHitTime"]     = 0.05;
});
```

ProjectileCore enables this by default for simulated definitions. Raw SwiftCast stays conservative unless explicitly configured. Custom `CanPierce` projectiles do not snap by default, because pre-running user pierce callbacks twice would be a wonderful way to create haunted state.

## Bezier, Homing, And Angles

```lua
ProjectileCore.SpawnSimulated("MagicMissile", {
	["Position"]   = StartPosition;
	["Velocity"]   = (TargetPart["Position"] - StartPosition).Unit * 220;
	["Angles"]     = CFrame.Angles(0, math.rad(90), 0);
	["TravelMode"] = "CubicBezier";

	["TravelData"] = {
		["Target"]                = TargetPart;
		["TrackTarget"]           = true;
		["TargetOffset"]          = Vector3.new(0, 1.5, 0);
		["ControlPointA"]         = StartPosition + Vector3.new(-20, 35, 0);
		["ControlPointB"]         = StartPosition + Vector3.new(30, 20, -80);
		["Duration"]              = 1.15;
		["CompletionMode"]        = "Homing";
		["HomingTurnSpeed"]       = 8;
		["HomingMinimumDistance"] = 2;
	};
});
```

`Angles` is an orientation offset applied to the projectile facing direction, including homing visuals. Use it when a mesh's local forward axis is annoying, which is somehow every mesh when the deadline is close.

## Physical Projectiles

Physical projectiles use Roblox physics/movers, but ProjectileCore can still manage ownership, deflection, time scale, lifetime, guided travel, and autocast hit checks.

```lua
local Identifier, ProjectileEntry = ProjectileCore.SpawnPhysical(ProjectilePart, {
	["Creator"]      = Character;
	["Deflectable"]  = true;
	["AutoLifetime"] = 5;

	["AutoCast"] = {
		["Enabled"]          = true;
		["Shape"]            = "Sphere";
		["Radius"]           = 3;
		["CastRate"]         = 60;
		["RespectTimeScale"] = true;
	};

	["OnPhysicalHit"] = function(Entry, HitResult : RaycastResult, Reason : string)
		ProjectileCore.DestroyProjectile(Entry["Identifier"], "Hit");
	end;
});
```

Use autocast for real hit logic when `.Touched` is too chaotic. `.Touched` is still usable for some physical gimmicks, but autocast gives projectile logic the same kind of deterministic treatment simulated projectiles get.

## Delayed True Projectile Pattern

For orbiting shurikens, blender bolts, and other delayed-arm attacks, spawn simulated projectiles as controlled visuals first, use an area/controller phase, then patch/redirect into real projectile behavior when they launch.

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
		});

		ProjectileCore.Redirect(Identifier, LaunchVelocity, nil, Vector3.zero, { Character; }, true);
	end;
end);
```

## Performance Notes

- Use `CreateProjectilePart = false` on server-authority server definitions when clients render visuals.
- Register client visuals with a warm cache sized for expected burst count.
- Prefer `SpherecastRadius` for fast bullets instead of giant overlap loops.
- Use close-hit snap for tiny point-blank travel so short casts do not waste a frame.
- Keep `AutoAcquireInterval` reasonable, usually `0.06` to `0.15` seconds.
- `FrozenMoment` / time-stop skips movement raycasts while fully paused. If it is not moving, casting air every frame is just expensive meditation.
- Tune `ReplicationSyncInterval`: `0.05` is tighter, `0.08` is the default, and `0` disables continuous correction.
- Use `ReplicationSmoothTime` to hide correction jitter.
- Use `ReplicationSnapDistance` as a safety valve for severe drift.
- Keep debug visualizers disabled during real gameplay benchmarks.
- Use `ControlImmunity` for boss/special projectiles instead of writing bespoke ignore checks in five different places.
- Avoid expensive logic in `OnPositionChange`; it runs often.

## Full API Reference

See [API.md](./API.md) for the full configuration reference, extended examples, AreaController fields, ControlImmunity rules, replication fields, Bezier settings, homing settings, patching rules, and the other stuff we keep adding because apparently projectiles are a lifestyle now.
