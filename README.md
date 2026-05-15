# ProjectileCore

> Designed for high-speed bullets, missiles, magic projectiles, area controllers, and etc for CGP:R

## Contributors

- **@BlajahBean** - Main programmer and mathematician. Occasionally fights the formatter and somehow still lands the math.
- **@SonarHEXAGON** - Sub programmer, testing, live implementation, physical/projectile integration work, and helper for GitHub here.
- **@RalziumQUANTUM** - Formatting, comments, and GitHub documentation creator (massive shoutout queen who logged my account to make this but ok)
- **@keklol0401010101010** - Creator of HomingCast.
- **@23sinek345** - Creator of SwiftCast and helped with optimization questions. 

## Table Of Contents

- [Features](#features)
- [Installation](#installation)
- [Recommended Layout](#recommended-layout)
- [Quick Start](#quick-start)
- [Authority Modes](#authority-modes)
- [Core API](#core-api)
- [Projectile Presets And Cache](#projectile-presets-and-cache)
- [Bezier And Homing](#bezier-and-homing)
- [Deflect And Attract Areas](#deflect-and-attract-areas)
- [Debug Visualizers](#debug-visualizers)
- [Performance Notes](#performance-notes)
- [Full API Reference](#full-api-reference)

## Features and Meaning

| Feature | Status | Notes |
| --- | --- | --- |
| Simulated projectiles | Supported | Fixed-step SwiftCast-backed simulation. |
| Physical projectiles | Supported | Real BasePart/Model projectiles with physical controllers. |
| Server authority | Supported | Server owns hits, damage, redirects, deflections, and destroy state. |
| Client authority | Supported | Useful for cosmetic/local-only projectile handling. |
| Client visual replicas | Supported | Server can simulate without server-side visual projectile parts. |
| Internal object cache | Supported | ProjectileCore-only cache for BasePart and Model templates. |
| Spherecast / shapecast | Supported | More reliable hit checks for high-speed or wide projectiles. |
| Deflect / DeflectArea | Supported | Single-projectile and area-based deflection APIs. |
| Homing / auto-homing | Supported | Direct targets or closest humanoid auto-acquisition. |
| Bezier travel | Supported | Linear, QuadraticBezier, and CubicBezier travel. |
| Tracked Bezier targets | Supported | Bezier targets can be moving Instances. |
| AttractArea | Supported | Blackhole-style velocity or acceleration damp pull. |
| Runtime patching | Supported | Validated live simulated state mutation. |
| Debug visualizers | Supported | Explicit opt-in cast, area, and homing visualizers. |
| ByteNet transport | Supported when available | Falls back to SoNET when ByteNet is unavailable. |

## Installation

Technically not necessary because it's already in ModuleScripts unless some1 tampered *cough cough*.

```text
ReplicatedStorage
└── ModuleScripts
    └── ProjectileCoreSystem
        └── ProjectileCore
```

Require it with:

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage");
--// Blame @BlajahBean 4 placing
local ProjectileCore = require(ReplicatedStorage["ModuleScripts"]["ProjectileCoreSystem"]:WaitForChild("ProjectileCore"));
```

## Layout stuff

Projectile presets live under:

```text
ProjectileCore.Objects.Projectiles.<ProjectileName>
```

A preset can be:

- A direct `BasePart` template.
- A `Model` template with `PrimaryPart` or a resolvable BasePart.
- A `ModuleScript` returning projectile/cache data.

## QuickS tart

### Server Authority, Server Handles Hits

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage");

local ProjectileCore = require(ReplicatedStorage["ModuleScripts"]["ProjectileCoreSystem"]:WaitForChild("ProjectileCore"));

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

local function FireRocket(Player : Player, StartPosition : Vector3, TargetPosition : Vector3) : string?
	local Character = Player["Character"];
	if not Character then
		return nil;
	end;

	local FireDirection = TargetPosition - StartPosition;
	if FireDirection.Magnitude <= 1e-4 then
		FireDirection = Vector3.new(0, 0, -1);
	end;

	local Identifier = ProjectileCore.SpawnSimulated("Rocket", {
		["Position"]     = StartPosition;
		["Velocity"]     = FireDirection.Unit * 360;
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

local VisualFolder     = GameSpace:FindFirstChild("ProjectileVisuals") or Instance.new("Folder");
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

## Authority Modes

| Mode | Use Case | Hit Source | Visual Source |
| --- | --- | --- | --- |
| `Server` | Damage, PvP, NPC combat, authoritative weapons | Server | Replicated client visual |
| `Client` | Cosmetic tracers, local previews, benchmark tests | Client | Local client visual |

Server-authority client replicas are visual-only. They do not resolve local hits. They wait for authoritative server spawn, deflect, redirect, patch, state-sync, and destroy packets.

Path-sensitive server projectiles such as homing, Bezier, and attract-controlled shots can use correction packets controlled by:

- `ReplicationSyncInterval`
- `ReplicationSmoothTime`
- `ReplicationSnapDistance`

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
| `SetHoming(Identifier, HomingData?, ShouldBroadcast?)` | Adds or changes homing behavior. |
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

## Projectile Presets And Cache

ProjectileCore includes an internal cache module so each projectile type can own a separate cache instead of sharing unrelated object pools.

```lua
local ProjectileCache = require(script["Parent"]["Parent"]["Parent"]["Shared"]:WaitForChild("ProjectileCache"));

local Projectile = script:FindFirstChild("TemplateBullet");
if not Projectile then
	return nil;
end;

return {
	["Projectile"] = Projectile;
	["Cache"]      = ProjectileCache.new("BenchmarkBullet", Projectile, 64);
};
```

Use `ProjectileCacheWarmCount` or a preset cache sized to the expected burst count. If a weapon/gun thing like Trench Gun (*though Trench only fires 20 Bullets per cast ass of now*) can spawn 48 projectiles at once, warm at least 48 objects. Making the allocator panic mid-fight is how the FPS spike start fuckin the game

## Bezier And Homing

ProjectileCore supports `Linear`, `QuadraticBezier`, and `CubicBezier` travel modes. Bezier targets can be static positions or moving Instances.

```lua
local Vec3 = Vector3.new;

ProjectileCore.SpawnSimulated("MagicMissile", {
	["Position"]   = StartPosition;
	["Velocity"]   = (TargetPart["Position"] - StartPosition).Unit * 220;
	["TravelMode"] = "CubicBezier";

	["TravelData"] = {
		["Target"]                = TargetPart;
		["TrackTarget"]           = true;
		["TargetOffset"]          = Vec3(0, 1.5, 0);
		["ControlPointA"]         = StartPosition + Vec3(-20, 35, 0);
		["ControlPointB"]         = StartPosition + Vec3(30, 20, -80);
		["Duration"]              = 1.15;
		["CompletionMode"]        = "Homing";
		["HomingTurnSpeed"]       = 8;
		["HomingMinimumDistance"] = 2;
	};
});
```

Auto-acquire homing can find the closest valid humanoid target:

```lua
ProjectileCore.SetHoming(Identifier, {
	["Enabled"]              = true;
	["AutoAcquire"]          = true;
	["AutoAcquireRange"]     = 220;
	["AutoAcquireInterval"]  = 0.08;
	["AutoAcquireRetarget"]  = true;
	["RequireLineOfSight"]   = false;
	["TurnSpeed"]            = 8;
	["MinimumDistance"]      = 2;

	["AutoAcquireValidator"] = function(TargetHumanoid : Humanoid, TargetCharacter : Model) : boolean
		return TargetHumanoid["Health"] > 0 and TargetCharacter ~= OwnerCharacter;
	end;
}, true);
```

## Deflect And Attract Areas

Grouped areas prevent overlapping parry/attract effects from repeatedly fighting over the same projectile.

```lua
ProjectileCore.DeflectArea({
	["Center"]           = RootPart;
	["Range"]            = 22;
	["Deflector"]        = Character;
	["ReflectionMode"]   = "Return";
	["SpeedMultiplier"]  = 1.12;
	["DeflectCooldown"]  = 0.12;
	["AreaGroup"]        = "ShieldField"; --// Or whatever
	["AreaPriority"]     = 1;
	["AreaLockDuration"] = 0.16;
	["ShouldBroadcast"]  = true;
});
```

```lua
ProjectileCore.AttractArea({
	["Center"]          = BlackholePart;
	["Range"]           = 60;
	["Strength"]        = 120;
	["Falloff"]         = 1.4;
	["Duration"]        = 0.95;
	["Method"]          = "VelocityDamp";
	["AreaGroup"]       = "Blackhole"; --// Or whatever
	["AreaPriority"]    = 2;
	["AffectSimulated"] = true;
	["AffectPhysical"]  = false;
	["ShouldBroadcast"] = true;
});
```

## Debug Visualizers

Debug visualizers are off by default and should stay disabled in production.

| Setting | Description |
| --- | --- |
| `DebugVisualize` | Global debug enable flag. |
| `DebugVisualizeCasts` | Draws ray/sphere/shapecast sweeps. |
| `DebugVisualizeAreas` | Draws DeflectArea and AttractArea volumes. |
| `DebugVisualizeHoming` | Draws homing range and current target line. |
| `DebugVisualizerLifetime` | Seconds before pooled visualizer release. |
| `DebugVisualizerColor` | Adornment color. |
| `DebugVisualizerTransparency` | Adornment transparency. |
| `DebugVisualizerAlwaysOnTop` | Always-on-top debug rendering. |

## Performance Notes

- Use `CreateProjectilePart = false` on server-authority server definitions when clients render visuals.
- Register client visuals with a warm cache sized for expected burst count.
- Prefer `SpherecastRadius` for fast bullets instead of many overlap checks.
- Keep spherecast radius as small as gameplay allows.
- Tune `ReplicationSyncInterval`: `0.05` is tighter, `0.08` is the default, and `0` disables continuous correction.
- Use `ReplicationSmoothTime` to hide correction jitter.
- Use `ReplicationSnapDistance` as a safety valve for severe drift.
- Keep `AutoAcquireInterval` reasonable, usually `0.06` to `0.15` seconds.
- Keep debug visualizers disabled during real gameplay benchmarks.
- Use grouped `DeflectArea` and `AttractArea` to avoid repeated controller replacement.
- Avoid expensive logic in `OnPositionChange`; it runs frequently.
- Buffers are useful for replication/telemetry payloads, not live projectile Entry tables.

## Full API Reference

See [API.md](./API.md) for the full configuration reference, extended examples, replication fields, Bezier settings, homing settings, patching rules, and etc... because I am too lazy to write them all down here and also because Polar is nagging for organisation so blame the dude.
