# ProjectileCore

ProjectileCore is the projectile system used in CGP:R. It separates server simulation from client visuals, with support for homing, Bezier paths, deflection, area controls, and registered physical projectiles.

These docs cover the integrated `ProjectileSystem_Test` build checked on **30 October 2026**. The source and dependencies are not distributed in this repository yet. The older standalone export has not been brought up to date with this build.

## Contents

- [Features](#features)
- [Setup And Authority](#setup-and-authority)
- [Server Gameplay Registration](#server-gameplay-registration)
- [Client Visual Registration](#client-visual-registration)
- [Preparation](#preparation)
- [API Map](#api-map)
- [Performance And Security](#performance-and-security)
- [API Reference](./API.md)
- [Contributors](#contributors)

## Features

- Linear and accelerated motion, homing, and quadratic/cubic Bezier travel.
- Ray, sphere, block, and shape casts; optional bouncing and a separate living-humanoid sweep radius.
- Deflection surfaces and area fields for Orbit, Tornado, Attract, Wind, and FrozenMoment, with per-projectile control immunity.
- Burst patterns, projectile groups, phase transitions, and age/distance curves.
- Cached client visuals, managed effects, angular spin, impact snapshots, and embedded impact attachments.
- ByteNet/SoNET delivery with spatial relevance, shared encoding, bounded packets, and paced recovery.

Native collision is the default. DDA/Bolt and the experimental Actor runtime have [specific eligibility limits](./API.md#collision-backends-and-dynamic-occupancy); Actor Auto promotion is currently disabled.

## Setup And Authority

In the integrated build, the root module is located here:

```text
ReplicatedStorage
  ModuleScripts
    ProjectileCoreSystem
      ProjectileCore
        Shared
        Simulated
        Physical
        Simulators
        Network
        ClientRegistrations
        Objects
          Projectiles
```

The examples below assume that build is already installed, including its game-specific networking, utility, and humanoid dependencies.

Require the root on the server and each client. Register the same projectile name independently on both sides:

- **Server:** movement, collision dimensions, ownership, lifetime, and damage callbacks.
- **Client:** templates, cache warming, trails/particles, and impact visuals.

Use `CreateProjectilePart = false` when the server only needs simulation. Collision and gameplay callbacks still run. Set collision dimensions on the server; the client template's size does not determine the server hitbox.

## Server Gameplay Registration

This server ModuleScript registers a projectile and returns a firing function. Call it after the ability's equipped-state, cooldown, and aim checks:

```lua
--// Rocket Gameplay

--// Services

local ReplicatedStorage = game:GetService("ReplicatedStorage");

--// Variables

local Self, Game = script, game;

--// Constants

local LoadedModules = {
	["ProjectileCore"] = require(ReplicatedStorage["ModuleScripts"]["ProjectileCoreSystem"]:WaitForChild("ProjectileCore"));
};

local ProjectileCore = LoadedModules["ProjectileCore"];

--// Primary

ProjectileCore["RegisterProjectile"]("Rocket", {
	["CreateProjectilePart"] = false;
	["SpherecastRadius"]     = 2;
	["ReplicateHitInfo"]     = true;

	["ProjectileSettings"] = {
		["MaxFlyTime"]     = 8;
		["MaxFlyDistance"] = 2500;
	};
});

--//

local function FireRocket(Character : Model, StartingPosition : Vector3, Direction : Vector3) : string
	local Identifier = ProjectileCore["SpawnSimulated"]("Rocket", {
		["Position"]             = StartingPosition;
		["Direction"]            = Direction;
		["Speed"]                = 360;
		["Creator"]              = Character;
		["Authority"]            = "Server";
		["CreateProjectilePart"] = false;
		["IgnoreList"]           = { Character; };
	});

	return Identifier;
end;

--// Return

return FireRocket;
```

Add damage through `OnServerHit`; see the [callback contracts](./API.md#callbacks-and-terminal-lifetime). `IgnoreList` must be a flat array of Instances, including any team characters you want excluded.

## Client Visual Registration

Put this ModuleScript directly under `ProjectileCore.ClientRegistrations`, name it `Rocket`, and add a BasePart or Model child named `Template`. Leave its `Disabled` attribute unset or false. Registration modules are direct children; the loader does not recursively search folders or resolve dependencies between them.

```lua
--// Rocket Visual Registration

--// Variables

local Self, Game = script, game;

--// Constants

local LoadedModules = {
	["Types"] = require(Self["Parent"]["Parent"]["Shared"]["Types"]);
};

local Types    = LoadedModules["Types"];
local Template = Self:WaitForChild("Template");
assert(Template:IsA("BasePart") or Template:IsA("Model"), "Rocket needs a BasePart or Model Template.");

--// Export

type ProjectileDefinition = Types.ProjectileDefinition;

--// Primary

local Definition : ProjectileDefinition = {
	["ProjectileTemplate"]       = Template;
	["ProjectileCacheWarmCount"] = 48;
	["CacheWarmMode"]            = "Background";
};

--// Return

return Definition;
```

A registration module may also return `function(ProjectileCore) -> Definition`. Use `Shared.Types` for its annotations: requiring the root from a registration module would create a circular dependency. Preload after the definition has been registered.

Use `PrepareProjectilePart` for effect setup. It runs after initial placement and before managed effects are restored. Let Core handle movement; a second CFrame loop or an early-enabled pooled trail can introduce visible jumps.

## Preparation

Call `PreloadProjectile` from client loading or ability-preparation code, after registration. It yields, so keep it outside movement and hit callbacks.

```lua
local Preparation = ProjectileCore["PreloadProjectile"]("Rocket", { ["WarmCount"] = 48; ["PreloadAssets"] = true; });

if not Preparation["CacheReady"] then
	warn(Preparation["CacheError"] or "Rocket cache is not ready.");
end;
```

Repeated calls warm toward 48 available objects rather than adding 48 each time. Asset failures are reported separately from cache readiness. Call it on clients to load their assets; server preparation only warms server caches. [Preparation details](./API.md#preparation-and-cache-ownership).

## API Map

| Task | Entry points |
| --- | --- |
| Register, prepare, spawn | `RegisterProjectile`, `PreloadProjectile`, `GetProjectileCacheStats`, `SpawnSimulated`, `SpawnPhysical` |
| Emit a pattern | [`SpawnBurst`](./API.md#bursts-and-patterns) |
| Manage an attack | [`CreateProjectileGroup`](./API.md#projectile-groups) |
| Change its phase | [`TransitionProjectile`, `GetProjectilePhase`](./API.md#safe-phase-transitions) |
| Redirect or guide | `Redirect`, `SetHoming`, `SetTimeScale`, `PatchProjectile` |
| Ownership and control immunity | `SetOwner`, `GetOwner`, `SetControlImmunity` |
| Deflection and fields | `Deflect`, `DeflectArea`, `AttractArea`, [`CreateAreaController`](./API.md#area-controllers), shield configuration |
| Projectile clashes | [`ConfigureProjectileCollisions`](./API.md#projectile-clashes), `ProjectileCollision` settings |
| Impact placement | [`GetImpactContext`, `GetImpactCFrame`, `AttachImpact`](./API.md#impact-context-and-attachments) |
| Age/distance behavior | [`SetProjectileCurves`, `ResetProjectileCurves`, `GetCurveValue`, `GetProjectileCurveState`](./API.md#distance-and-age-curves) |
| Inspect or end | `GetData`, `GetAll`, `IsActive`, `GetOwnerFromPart`, `Cancel`, `DestroyProjectile` |
| Networking | `SetNetworkTransport`, `GetNetworkTransport`, `GetNetworkInfo`, `ReplicationPolicy` |
| Measure/tune | `GetPerformanceStats`, `SetPerformanceProfilingEnabled`, `SaveScriptProfilingData`, `ConfigureVisualBudget`, collision-map/dynamic-collider APIs, `ConfigureParallel` |

## Performance And Security

- Prepare caches before large bursts and keep frequent callbacks small. Burst helpers still create ordinary projectiles with their usual simulation and delivery costs.
- Use a persistent Attract controller for a sustained field instead of repeatedly broadcasting `AttractArea` pulses.
- Keep replicated `UserData` small and non-secret. Leave private ability state in server-owned storage; the [patch caveat](./API.md#userdata) matters here.
- Sample diagnostics at a modest rate and disable debug visuals when measuring. Client visual budgets can reduce particle/light work without changing server collision.

Remote guards handle malformed traffic and rate limits. Weapon code must still validate equipped state, cooldown, ammo, firing origin, aim, and damage permissions on the server.

The eight-local-client test delivered all 48 shots to each client through both transports, but client receive-to-prepare delays exceeded one second on the shared test machine. Delivery passed; latency remains unresolved. See [test results and known limits](./API.md#current-limits-and-verification) before planning a larger server test.

## Contributors

- **@BlajahBean** - Main programmer and mathematician.
- **@SonarHEXAGON** - Sub programmer, testing, live implementation, physical/projectile integration, and GitHub maintenance.
- **@RalziumQUANTUM** - Formatting, comments, and GitHub documentation.
- **@keklol0401010101010** - Creator of HomingCast.
- **@23sinek345** - Creator of SwiftCast; helped with optimization questions.
