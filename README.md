# ProjectileCore

> High-speed simulated projectiles, physical projectile control, homing, Bezier travel, area fields, deflection, attraction, and client visuals for CGP:R-style combat.

ProjectileCore lets the server own real projectile truth while clients render something that does not look like it was dragged behind a truck. Bullets, lobbed blobs, orbiting murder Frisbees, time-stop nonsense, and the usual "why is this projectile doing forbidden geometry" edge cases live here.

**Documentation baseline: the integrated `ProjectileSystem_Test` implementation inspected on 30 October 2026.** This repository currently contains documentation, not the source modules or an installable release. The separately prepared standalone export is older and must be refreshed and validated before distribution. Reading the README does not summon its dependencies. We checked.

## Contributors

- **@BlajahBean** - Main programmer and mathematician. Occasionally fights the formatter and somehow still lands the math.
- **@SonarHEXAGON** - Sub programmer, testing, live implementation, physical/projectile integration work, and helper for GitHub here.
- **@RalziumQUANTUM** - Formatting, comments, and GitHub documentation creator. The grammar police arrived armed.
- **@keklol0401010101010** - Creator of HomingCast.
- **@23sinek345** - Creator of SwiftCast and helped with optimization questions.

## Contents

- [Features](#features)
- [Setup And Authority](#setup-and-authority)
- [Server Gameplay Registration](#server-gameplay-registration)
- [Client Visual Registration](#client-visual-registration)
- [Prepare Before The Fight](#prepare-before-the-fight)
- [API Map](#api-map)
- [Performance And Security](#performance-and-security)
- [Full Reference](./API.md)

## Features

| System | Current capability |
| --- | --- |
| Simulation | Linear, accelerated, homing, quadratic/cubic Bezier; client cosmetic or server gameplay authority. |
| World collision | Ray, sphere, block, or shapecast; separate living-humanoid sphere radius; close-hit snap; optional bouncing. |
| Control | Direct/area deflection, configurable shields, one-shot attraction, and persistent Orbit, Tornado, Attract, Wind, FrozenMoment fields. |
| Immunity | Per-projectile control immunity, separate from normal world collision and damage. |
| Authoring | Centered Fan, Ring, Cone, and Sphere bursts; runtime-local projectile groups; atomic phase transitions. |
| Curves | Effective-age or committed-distance speed, visual/collision scale, and named numeric values. Damage is still your job. |
| Impacts | Optional authoritative contact snapshots and separate client-only embedded/following impact visuals. |
| Visuals | Cached parts/Models, initial-placement warmup, managed effects, angular spin, optional distant particle/light budgets. |
| Networking | Auto/ByteNet/SoNET, spatial relevance, shared encoding, byte-safe reliable batches/fragmentation, paced recovery, and inbound guards. |
| Collision acceleration | Opt-in DDA/Bolt filtering with conservative native fallbacks and tracked dynamic occupancy. Native remains the default. |
| Parallel runtime | Restricted experimental server worker lane. Auto promotion is currently disabled; this is not a "more Actors = more FPS" button. |
| Diagnostics | Rolling timings, MicroProfiler labels, queues, cache misses, homing acquisition, controls, and cleanup counters. |

See the [API reference](./API.md) for defaults and limits. Features being present does not mean every combination is free, automatic, or certified on your toaster.

## Setup And Authority

For an existing installation of the integrated build, the usual root is:

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

This is a **location convention**, not a complete installer. The integrated build also references game-owned networking, utility, and humanoid dependencies. Copying one root script out of these docs is not installation. The older standalone package has different isolation/configuration rules; do not mix its instructions into this build.

Require the root on the server and each client. Register the same projectile name independently on both sides:

- **Server:** movement, collision dimensions, allegiance/damage decisions, ownership, lifetime, and gameplay callbacks.
- **Client:** templates, cache warming, trails/particles, preparation, impact visuals, and visual retention.

`CreateProjectilePart = false` on the server avoids creating a server visual. It does **not** disable server simulation, spherecasts, damage callbacks, or clashes. Client registration does not set the server's hit radius. A giant local mesh is not a giant server hitbox.

## Server Gameplay Registration

Example server ModuleScript, called by trusted ability code after its own usability/cooldown/aim checks:

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

Add your server `OnServerHit` damage logic using the [callback contracts](./API.md#callbacks-and-terminal-lifetime). Always flatten ignore arrays; putting a team-character array inside another array does not exclude its contents.

## Client Visual Registration

Put this ModuleScript directly under `ProjectileCore.ClientRegistrations`, name it `Rocket`, and give it a separate BasePart/Model child named `Template`. Keep its `Disabled` attribute unset or false. The current loader discovers direct registration children; do not rely on arbitrary nested-folder traversal or load order.

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

The registration can instead return `function(ProjectileCore) -> Definition`. **Do not require the root from its own registration modules:** the loader is already running inside that require. For typed factories, require `Shared.Types`, not the root. Call preloading **after** the returned definition has been registered, not inside the factory before it exists.

Core positions the leased visual before preparation/effect restoration. Keep effect setup in `PrepareProjectilePart`; do not add a second movement loop, parent visuals into gameplay assemblies, or enable a pooled trail before initial placement. Origin snaps are not a feature.

## Prepare Before The Fight

Run this from client loading/ability-preparation code after requiring the root and completing registration. It yields; movement/hit callbacks are the wrong place for it.

```lua
local Preparation = ProjectileCore["PreloadProjectile"]("Rocket", { ["WarmCount"] = 48; ["PreloadAssets"] = true; });

if not Preparation["CacheReady"] then
	warn(Preparation["CacheError"] or "Rocket cache is not ready.");
end;
```

Repeated requests warm **toward** the available target instead of adding another 48 every time. Client asset failures are reported separately from cache/metadata readiness. Server preparation cannot load assets for other machines. See [preparation and cache ownership](./API.md#preparation-and-cache-ownership).

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

- Warm client caches and metadata before the first fight. Cache misses, Model descendants, particles, and callbacks still cost time. `SpawnBurst` simplifies authoring; it does not make 50 blades cost one blade.
- Prefer server-invisible simulated gameplay plus cached client visuals. Do not make a workspace Part solely to give a server raycast an identity.
- Keep world collision volumes small enough for gameplay. Use the separate living-humanoid radius when character generosity should not make a meteor hit a wall ten studs early.
- Use a persistent Attract controller instead of repeatedly broadcasting overlapping attraction pulses. Keep one-shot `AttractArea` for actual pulses.
- Homing acquisition and repeated query/filter work are scheduled/reused, not eliminated. Custom validators and high-frequency callbacks can still become your expensive little gremlins.
- Reliable lifecycle delivery is byte-budgeted and paced; ordinary corrections are disposable/coalesced. Keep replicated user data small and non-secret. Debug visualizers and broad diagnostic snapshots are not per-projectile hot-loop tools.
- Client particle/light distance budgets are opt-in and do not lower server collision accuracy. DDA/Bolt/Actors are measured options with eligibility and fallback rules, not default speed guarantees.
- Inbound guards reject malformed/forged Core traffic, limit calls/bytes and invokes, and throttle recovery. They do not validate your weapon's equipped state, cooldown, ammo, firing origin, or damage permission. Server-authoritative does not mean every surrounding remote is magically safe.

The recent local eight-client checks delivered all 48 projectiles per client through both transports with no reported send/flush failures or terminal queue residue. They also recorded high frame times and client receive-to-prepare delays above one second on the shared test machine. Those are delivery-count passes, **not latency passes or 16-player certification**. Proposed inbox catch-up improvements are not implemented in this documentation baseline.

## Full Reference

Read [API.md](./API.md) for contracts, defaults, handle methods, examples, current limitations, and the distinction between "stopped firing" and "all projectiles have ended". That distinction has saved at least one ability from becoming haunted.
