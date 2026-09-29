# Zenix

Zenix is the shared runtime and UI framework used by the San Aurie game
project. This guide covers both loading the built script as an end user and
building Zenix from source as a maintainer.

> Use Zenix only where you have permission to run custom code, and follow the
> rules of the target experience and the runtime you use.

## End-user quick start

### Prerequisites

You need:

- Access to a supported Roblox-compatible runtime that provides the file and
  request functions required by the loader.
- The current Zenix loader artifact from the project maintainer.
- A valid Zenix key when the active key policy requires one.

The loader expects runtime support for functions such as `isfile`, `readfile`,
and `loadstring`. If one is missing, the loader stops with an explicit runtime
error rather than loading partially.

### Load Zenix

1. Join the supported game.
2. Run the current loader artifact supplied by the maintainer.
3. Complete the key-system flow if prompted.
4. Wait for the `Project San Zenix` window and its tabs to finish loading.
5. If the script reports that the game version is unsupported, update the
   script or leave the session rather than assuming every feature will work.

The key system verifies access before requesting the game-specific script. It
stores runtime data below the `Zenix/KeySystem` directory and diagnostic files
below `Zenix/Logs`. Do not share keys or generated log files publicly.

### UI overview

The San Aurie build normally provides these tabs:

- **Home**: compatibility status, server actions, server information,
  localization, credits, and support links.
- **LocalPlayer**: player movement, stamina, detection, protection, and
  automation controls.
- **Vehicle**: vehicle selection, spawning, movement, protection, and vehicle
  stat controls.
- **Gun Mods**: weapon values, reload/equip behavior, aim options, and item
  modifiers.
- **Game Rules**: game-rule toggles, timers, cooldowns, world settings, and
  vehicle/physics settings.
- **Shop**: item selection, quantity settings, and purchase actions.
- **ESP**: player, team, vehicle, drawing, and visibility options.
- **Teleports**: players, areas, missions, map locations, and saved locations.
- **Theme**: UI appearance settings exposed by the shared UI.
- **Keybinds**: session-only shortcuts for toggles that support keybinds.
- **Config Tab**: saved configuration, in-game notification mode, and language.

Feature names and availability can change with the game version and the active
access policy. A toggle reports success or failure through the notification
system when in-game notifications are enabled.

### Configuration and keybinds

Use **Config Tab** to save or load the settings supported by the UI. The
project stores configuration under the `Zenix` project directory; do not move
or rename that directory while the script is running.

Use **Keybinds** to assign shortcuts to existing toggles. Keybinds are
session-only and reset when the session ends. Use **Reset Keybinds** to clear
all assignments. A bind set to `None` is disabled.

Use the **Language** selector in **Config Tab** to switch among the bundled
translations. The UI refreshes localized text after a successful change.

### Reloading and cleanup

For a clean reload, stop the current instance before executing a new copy. The
source loader also attempts to clean up a previous registry before rebuilding
the combined runtime. If a reload reports cleanup warnings, leave and rejoin
the game before trying again.

Do not load the game component by itself. Split source loading requires the
shared core component first; the game component intentionally fails when the
core registry is absent.

### Troubleshooting

**The loader says a runtime function is unavailable**

Use a supported runtime and verify that file access, compilation, and request
functions are enabled. The loader cannot continue without them.

**The UI does not appear**

Wait for startup to finish, then check the runtime console for a module or UI
initialization error. If the problem persists, restart the game and run one
fresh loader instance only.

**The game is reported as unsupported**

The Home tab checks the current game and version against the server-managed
compatibility manifest. Use a current release and report the displayed game
version to the maintainer.

**A feature reports an error**

Disable the feature, rejoin the game, and try again after confirming that the
game version is supported. Preserve the relevant console/log message when
reporting the issue, but remove keys and other private data first.

**A key or script request fails**

Check connectivity, confirm that the key is current, and verify that the
maintainer's loader channel is available. Do not repeatedly submit a key when
the server has returned a clear rejection.

## Developer and maintainer workflow

### Repository layout

```text
Zenix/
  CoreAutoLoad/       Shared infrastructure, key system, runtime services
  Games/SanAurie/     San Aurie modules, tabs, localization, and adapters
  Games/JailBird/     Initial JailBird scaffold
  UI/                 Shared Luna UI integration
Production/           Generated readable/protected build artifacts
main.lua              Source-folder development loader
key.lua               Key-system compatibility bootstrap
build_zenix.py        Build and upload entry point
ZENIX_BUILD.md        Detailed build-system reference
```

Shared infrastructure belongs under `Zenix/CoreAutoLoad`. San Aurie-specific
behavior belongs under `Zenix/Games/SanAurie`. The shared UI is included by the
core artifact; do not add a second game-local UI unless a target explicitly
uses a different `ui_path`.

### Shared teleport API

Zenix owns the low-level `TeleportService` module. Use
`TeleportService:Teleport(target, destinationCFrame)` for an instant move of a
`BasePart` or `Model` target. Models are moved with `PivotTo`, while parts are
moved with `CFrame`. Pass a positive third argument to `Teleport` to tween
instead, or call
`TeleportService:TweenTeleport(target, destinationCFrame, duration, easingStyle, easingDirection)`
directly. Tweening defaults to linear easing with an `InOut` direction. Both
methods return `true` on success or `false, errorMessage` when arguments are
invalid; tweening also returns the started Tween as the second result. Game
modules remain responsible for destination lookup, offsets, collision/floor
handling, and any game-specific vehicle policy.

San Aurie teleport modules use `TeleportByDistance` by default through the
`DestinationTeleport:TeleportToDestination(targetCFrame)` wrapper. Passing an
explicit duration still uses a fixed-duration tween:
`DestinationTeleport:TeleportToDestination(targetCFrame, duration)`.

For distance-aware movement, use
`TeleportService:TeleportByDistance(target, destinationCFrame, options)`.
With the defaults, targets within 100 studs move instantly; a 1,000-stud
distance takes 3 seconds; and longer distances continue on the same linear
curve. Configure the curve with `InstantDistance`, `FullTweenDistance`, and
`FullTweenDuration`, and optionally provide `EasingStyle` and
`EasingDirection`. The method returns
`true, tweenOrNil, duration, distance` on success or
`false, errorMessage, duration, distance` on failure.
Tween calls yield until the tween completes. If another tween is already
running for the same target, a new tween request waits for that active tween
to finish before starting.
When the tween target is a character model, the service leaves its
`BasePart` descendants unanchored and checks their linear and angular velocity
each heartbeat. A velocity is reset only when its magnitude exceeds 50, which
prevents extreme spinning and accumulated physics velocity without changing
the character's anchored state.

### Local builds

Run commands from the repository root:

```powershell
# Detect changed sources and rebuild affected artifacts.
python .\build_zenix.py

# Preserve the legacy combined artifact.
python .\build_zenix.py --component combined

# Build a shared core artifact.
python .\build_zenix.py --component core

# Build the selected game artifact.
python .\build_zenix.py --component game

# Build the standalone key-system loader.
python .\build_zenix.py --component key_system

# Select the JailBird target.
python .\build_zenix.py --game jailbird --component game
```

The default target is `sanaurie`. Generated artifacts are written under
`Production/`. The default/combined flow retains the legacy filenames. Split
core and game artifacts use `core_` and `game_` prefixes.

The standalone `key_system` build produces exactly one local
`Production/loader.lua`. It includes only the key-system dependency closure,
does not include game modules, and does not upload or publish anything.

### Split component loading

For source-folder development loading, set the component before executing
`main.lua`:

```lua
getgenv().SANZENIX_COMPONENT = "core"
-- Execute main.lua, then execute it again with:
getgenv().SANZENIX_COMPONENT = "game"
```

Load the core readable chunk and protected chunk before the game readable and
protected chunks. The core creates the shared `ZenixRegistry`; the game
component registers against that existing registry. Leave
`SANZENIX_COMPONENT` unset to use the legacy combined loader.

### Function and module usage

After loading, the shared registry is available as `_G.ZenixRegistry`. The
loader also exposes `RunModule` through `getgenv()`:

```lua
local module = getgenv().RunModule("CarFly")
if module == false then
    warn("CarFly could not be loaded or executed")
end

-- Arguments are forwarded to the module's Toggle, Execute, or function entry
-- point, depending on the module type.
getgenv().RunModule("VehicleTeleport", targetVehicle)
```

`RunModule(name, ...)` loads a registered module on demand and dispatches in
this order:

1. `module:Toggle(...)` when the module exposes `Toggle`.
2. `module(...)` when the module itself is a function.
3. `module:Execute(...)` when the module exposes `Execute`.

It returns the module object on success and `false` when the module cannot be
loaded or its protected entry point fails. Use the module's documented
arguments; most feature toggles expect a boolean, while action modules may
expect a player, vehicle, location, or other game object.

The registry also provides lifecycle and diagnostics helpers:

```lua
local registry = _G.ZenixRegistry

registry.LoadModule("CarFly")
local health = registry.GetModuleHealth("CarFly")
local loaded = registry.GetLoadedModules()
local path = registry.GetModulePath("CarFly")

-- Stop and release one module, or clean up the complete runtime.
registry.CleanupModule("CarFly")
registry.CleanupAll()
```

For feature modules, prefer the registry helpers over calling private fields
directly. Modules should return a table with a `Toggle` or `Execute` method,
declare their dependencies through the module context, and release
connections/resources from `Stop` or the registered cleanup callback.

New source modules should use a factory boundary:

```lua
return function(context)
    local registry = context:required(context.registry, "registry is required")
    local Players = context:required(context:getService("Players"), "Players is required")
    local EventConnections = context:getDependency("EventConnectionManager")

    local module = {
        enabled = false,
    }

    function module:Toggle(enabled)
        self.enabled = enabled == true
        -- Install or remove feature behavior here.
        return self.enabled
    end

    function module:Stop()
        self.enabled = false
        -- Disconnect events and release resources here.
    end

    return module
end
```

The factory is called after declared dependencies are resolved. Use
`context:required(value, message)` for mandatory values,
`context:getService(name)` for Roblox services, and
`context:getDependency(name)` for other Zenix modules. Keep per-instance state
in the returned table or local variables; do not publish setup state directly
through the global registry.

### Core service API

Core services are initialized by the core component, but they are not all
intended to be fetched the same way:

- Use `_G.ZenixRegistry` for services explicitly published there, such as
  `FileSystemManager`, `InstanceManager`, `GameModuleManager`, `RemoteManager`,
  `FunctionHookManager`, `FeaturePolicy`, `Localization`, and
  `GameCompatibility`.
- Inside a module factory, prefer `context:getDependency("ServiceName")`.
  This is required for services such as `EventConnectionManager` and is safer
  for load order and testing.
- Always validate optional services before calling them.

#### FileSystemManager

`FileSystemManager` centralizes runtime file access and returns
`success, valueOrError`:

```lua
local fs = _G.ZenixRegistry.FileSystemManager

local ready, errorMessage = fs:EnsureFolders("Zenix/Cache")
if not ready then
    error(errorMessage)
end

local exists, kind = fs:Exists("Zenix/Cache/state.json")
local readOk, contents = fs:Read("Zenix/Cache/state.json")
if readOk then
    print(contents)
end

fs:Write("Zenix/Cache/state.json", "{}")
fs:Append("Zenix/Logs/custom.log", "started\n")
```

Available operations include `IsAvailable`, `Exists`, `EnsureFolder`,
`EnsureFolders`, `Read`, `Write`, `Append`, `Delete`, `List`, and `Path`.
Prefer this service over calling executor file functions directly.

#### EventConnectionManager

Modules receive `EventConnectionManager` through
`context:getDependency("EventConnectionManager")`. Use managed subscriptions
so callbacks and periodic tasks can be disconnected when a module stops:

```lua
local events = context:getDependency("EventConnectionManager")

local subscription = events:InstallRunService("Heartbeat", function(deltaTime)
    updateFeature(deltaTime)
end)

-- In the module's Stop or cleanup function:
events:Disconnect(subscription)
```

Other helpers are `InstallUserInputService`,
`InstallInstanceSignal(instance, eventName, callback)`, and
`InstallPeriodic(name, interval, callback)`. Periodic intervals must meet the
manager's minimum interval. `GetConnections()` and `Cleanup()` are useful when
diagnosing or shutting down a module.

#### InstanceManager and GameModuleManager

Use `InstanceManager` to track created or cloned Roblox instances:

```lua
local instances = _G.ZenixRegistry.InstanceManager
local folder = instances:Create("Folder", workspace)
instances:Track(folder)

-- Releases tracked instances during module cleanup.
instances:Cleanup()
```

`GameModuleManager` provides game-module access through `Get(modulePath)`,
`Require(modulePath)`, `Clear(modulePath)`, `GetRoot()`, and `Cleanup()`.
Prefer it over creating a second module-loading mechanism.

#### Localization and compatibility

```lua
local localization = _G.ZenixRegistry.Localization
local text = localization:T("Home")
local languageOk, languageError = localization:SetLanguage("en", true)

local compatibility = _G.ZenixRegistry.GameCompatibility
local refreshed, refreshError = compatibility:Refresh(game.GameId, game.PlaceId)
local status = compatibility:GetStatus(game.GameId, game.PlaceVersion, game.PlaceId)
```

`Localization` also supports `Register`, `RegisterText`,
`RegisterLanguageChangedListener`, `RemoveLanguageChangedListener`,
`GetLanguageOptions`, and `ExportEnglishTemplate`. `GameCompatibility` also
supports `LoadCached`; check the returned status before enabling
version-sensitive features.

#### FeaturePolicy

Define a feature policy before checking it, then use the active tier to gate
work:

```lua
local policy = _G.ZenixRegistry.FeaturePolicy

policy:Define("VehicleStats", {
    free = {enabled = true, maxSpeed = 100},
    premium = {enabled = true, maxSpeed = 250},
})

if policy:CanUse("VehicleStats") then
    local limit = policy:GetLimit("VehicleStats", "maxSpeed", 100)
    applySpeedLimit(limit)
end
```

Other policy methods include `Get`, `GetPolicy`, `Define`, `SetEnabled`,
`SetLimit`, `Include`, `Lock`, `IsPremium`, and `GetSliderMaximum`.

#### RemoteManager and FunctionHookManager

These managers coordinate shared interception and hook lifecycle. Use their
named add/remove methods and always clean up:

```lua
local remoteManager = _G.ZenixRegistry.RemoteManager
remoteManager:EnsureInstalled()
if remoteManager:IsHooked() then
    remoteManager:AddFilter("MyFeatureFilter", filterFunction)
end

local hooks = _G.ZenixRegistry.FunctionHookManager
local scanCriteria = {
    functions = {targetFunction},
}
local installed = hooks:AddHook(
    "MyFeatureHook",
    scanCriteria,
    function(...)
        return handleCall(...)
    end,
    {required = false, chain = false}
)
if installed then
    hooks:DisableHook("MyFeatureHook")
    hooks:EnableHook("MyFeatureHook")
end

-- Remove filters and hooks when the feature stops.
remoteManager:RemoveFilter("MyFeatureFilter")
hooks:RemoveHook("MyFeatureHook")
```

`RemoteManager` also exposes modifier, invoke-filter, callback-filter,
caller-guard, profile, and inspection methods. `FunctionHookManager` exposes
`AddHook`, `RemoveHook`, `EnableHook`, `DisableHook`, `ToggleHook`, `GetHook`,
`GetHooks`, `IsHooked`, `GetHookedFunctionCount`, `StartAutoScan`,
`StopAutoScan`, and `Cleanup`. `RefreshHook` and `RefreshAllHooks` are
diagnostic stubs that return `false`; use `RemoveHook` followed by `AddHook` to
rebuild a hook. These APIs are shared infrastructure; avoid global hooks when
a scoped module or event subscription is sufficient.

Common shared services are registered on `_G.ZenixRegistry`:

```lua
local registry = _G.ZenixRegistry

registry.NotificationManager:Success("Zenix", "Feature enabled")
registry.NotificationManager:Warning("Zenix", "Compatibility is unknown")

local translated = registry.Localization:T("Some English label")
local status = registry.GameCompatibility:GetStatus(
    game.GameId,
    game.PlaceVersion,
    game.PlaceId
)
```

`NotificationManager` also exposes `Info`, `Success`, `Warning`, and `Error`.
`Localization` supports `T`, `Localize`, `SetLanguage`, and language
registration helpers. `GameCompatibility:GetStatus` returns the current
compatibility result after checking the server manifest and cache. Check that
each service exists before using it when writing a reusable module.

### Debugging Zenix

#### Enable debug output

Set `ZENIX_DEBUG_TARGET` before running the loader:

```lua
-- Key-system diagnostics only.
_G.ZENIX_DEBUG_TARGET = "key"

-- Game/script diagnostics only.
_G.ZENIX_DEBUG_TARGET = "script"

-- Both key-system and game/script diagnostics.
_G.ZENIX_DEBUG_TARGET = "all"

-- Default: no persistent debug log.
_G.ZENIX_DEBUG_TARGET = "none"
```

Set the value before executing the loader, not after startup. Key diagnostics
are written to `Zenix/Logs/Debug_<session>.log`; game/script diagnostics are
written to `Zenix/Logs/Script_<session>.log` when enabled. Messages are also
printed to the runtime console when available. Do not upload or share logs
containing keys, tokens, player identifiers, or private server data.

The shared debug service is available as `_G.ZenixRegistry.Debug`:

```lua
local Debug = _G.ZenixRegistry.Debug

Debug.Info("Loaded module %s", "CarFly")
Debug.Warn("Optional dependency is unavailable")
Debug.Error("Feature failed: %s", tostring(errorMessage))
Debug.Debug("Detailed value: %s", tostring(value))
Debug.Success("Feature enabled")

Debug.SetLogLevel(Debug.LOG_LEVELS.DEBUG)
local history = Debug.GetHistory()
```

The default log level is `INFO`; use `Debug.LOG_LEVELS.DEBUG` when tracing
detailed control flow. `SetDevelopmentMode()` enables logging and
`SetProductionMode()` disables it. `Debug.ClearHistory()` clears the in-memory
history without deleting existing log files.

#### Protect and trace failing calls

Use the protected helpers for optional or failure-prone module work. They
return a success boolean and the callback result or error:

```lua
local Debug = _G.ZenixRegistry.Debug

local ok, result = Debug.ProtectedCall("MyFeature.refresh", function()
    return refreshFeature()
end)

local methodOk, methodResult = Debug.ProtectedMethodCall(
    "MyFeature.toggle",
    featureModule,
    "Toggle",
    true
)
```

Failures are printed with a traceback and recorded through the debug service.
Always check the returned boolean; do not silently continue after a required
initialization call fails.

#### Inspect module health

Use the registry diagnostics when a feature does not load or repeatedly stops:

```lua
local registry = _G.ZenixRegistry

local health = registry.GetModuleHealth("CarFly")
local report = registry.GetModuleHealthReport()

-- Retry a failed module after correcting its dependency or configuration.
local retryOk = registry.RetryModule("CarFly")
```

If a retry still fails, capture the module name, health result, runtime console
message, game version, and the relevant `Zenix/Logs` file. Remove keys and
other sensitive values before sharing the report.

#### Send authorized diagnostic logs

When the server-side diagnostic endpoint is configured, the debug service can
collect recent logs:

```lua
local sent, sendError = _G.ZenixRegistry.Debug:SendLogs({
    mode = "recent", -- "recent", "all", or "timeframe"
    source = "game", -- "game", "key", or "all"
    limit = 200,
})
if not sent then
    warn("Log upload failed: " .. tostring(sendError))
end
```

For `mode = "timeframe"`, provide either `minutes` or a Unix timestamp in
`since`. Use log submission only with an authorized maintainer endpoint.

#### Debugging workflow

1. Reproduce the issue with one loader instance and record the game version.
2. Retry with `ZENIX_DEBUG_TARGET = "script"` or `"all"` as appropriate.
3. Check the runtime console and the matching `Debug_` or `Script_` log.
4. Use `GetModuleHealth` and `RetryModule` for module-specific failures.
5. Disable the failing feature and confirm whether a clean reload resolves it.
6. Share a redacted error, module name, game version, and relevant log excerpt
   with the maintainer.

### Beta and release uploads

Automatic beta uploads occur only when a local `beta_upload.json` exists beside
`build_zenix.py`:

```json
{
  "url": "https://example.invalid/script/",
  "key": "USE-A-LOCAL-AUTHORIZED-KEY",
  "gameId": "YOUR-GAME-ID"
}
```

Use the real maintainer-provided values locally. Do not commit this file, its
key, or environment variables containing upload credentials.

The builder uploads game artifacts for the configured game and shared core
artifacts under the shared core scope. Release snapshot uploads additionally
require the admin URL and credentials described in
[ZENIX_BUILD.md](../ZENIX_BUILD.md); publication remains an explicit admin
workflow.

### Compatibility metadata

The Home tab reads the server-managed compatibility status for the current
universe/game version. The default endpoint is:

```text
GET /api/game_compatibility.php?gameId=<universeId>
```

The last valid manifest is cached at `Zenix/Cache/GameCompatibility.json`.
Maintain one compatibility record per game in the server database, and test
both supported and unsupported version responses before publishing a build.

## Further reference

See [ZENIX_BUILD.md](../ZENIX_BUILD.md) for the complete build state model,
upload behavior, key-system contracts, module annotations, artifact layout,
and server-side compatibility details. Treat generated files under
`Production/` as build outputs rather than source documentation.
