# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UnLua is a Lua scripting plugin for Unreal Engine (UE4.17–UE5.x) by Tencent. It bridges Lua and UE's C++ reflection system, allowing direct access to UCLASS, UPROPERTY, UFUNCTION, USTRUCT, and UENUM from Lua without glue code.

The repo contains:
- `Plugins/UnLua/` — the main plugin (drop into any UE project)
- `Plugins/UnLuaExtensions/` — optional extensions (LuaProtobuf, LuaRapidjson, LuaSocket)
- `Plugins/UnLuaTestSuite/` — functional test suite using UE's automation system
- `TPSProject.uproject` — the test/demo project that uses the plugin
- `Content/Script/` — Lua scripts for the demo project
- `Docs/` — documentation in Chinese (`CN/`) and English (`EN/`)

## Building

This is a standard UE plugin project. There is no standalone build script.

1. Open `TPSProject.uproject` in Unreal Engine (or copy `Plugins/` into your own project)
2. If using C++: right-click the `.uproject` → "Generate Visual Studio project files", then build via VS or Rider
3. The editor will compile UnLua modules automatically on startup

**Build-time configuration** is read from `Config/DefaultUnLuaEditor.ini` under the section `[/Script/UnLuaEditor.UnLuaEditorSettings]`. Key compile-time macros (all require recompile to take effect):

| Config Key | Macro | Default |
|---|---|---|
| `bAutoStartup` | `AUTO_UNLUA_STARTUP` | true |
| `bEnablePersistentParamBuffer` | `ENABLE_PERSISTENT_PARAM_BUFFER` | true |
| `bEnableTypeChecking` | `ENABLE_TYPE_CHECK` | true |
| `bEnableDebug` | `UNLUA_ENABLE_DEBUG` | false |
| `bWithUE4Namespace` | `WITH_UE4_NAMESPACE` | true |
| `HotReloadMode` | `UNLUA_WITH_HOT_RELOAD` | Manual |
| `LuaVersion` | `UNLUA_LUA_VERSION` | lua-5.4.3 |

## Running Tests

Tests use UE's Session Frontend automation system:
1. Open the demo project in the UE editor
2. `Window → Test Automation` (or Session Frontend → Automation tab)
3. Run tests under the `UnLua` category

The test suite plugin (`UnLuaTestSuite`) must be enabled. Test source is in `Plugins/UnLuaTestSuite/Source/`.

## Architecture

### Module Layout (`Plugins/UnLua/Source/`)

| Module | Phase | Purpose |
|---|---|---|
| `UnLua` | PreDefault | Core runtime: reflection bridge, Lua environment, bindings |
| `UnLuaEditor` | Default | Editor tools: template generation, blueprint Lua export |
| `UnLuaDefaultParamCollector` | Program | Collects UFunction default parameter values |
| `ThirdParty/Lua` | — | Embedded Lua 5.4.3 source |

### Core Runtime (`UnLua` module)

**`FLuaEnv`** (`Public/LuaEnv.h`) is the central class — one per `lua_State`. It owns all registries and listens for UObject deletion. Multiple independent `FLuaEnv` instances can coexist (useful for environment isolation per `GameInstance`).

```
FLuaEnv
  ├─ FClassRegistry     → UStruct/UClass ↔ FClassDesc
  ├─ FPropertyRegistry  → property metadata and type ops
  ├─ FFunctionRegistry  → UFUNCTION metadata
  ├─ FDelegateRegistry  → delegate binding and callbacks
  ├─ FObjectRegistry    → live UObject instance tracking
  ├─ FEnumRegistry      → UENUM metadata
  └─ FContainerRegistry → TArray/TSet/TMap wrappers
```

**`FClassDesc`** (`Private/ReflectionUtils/ClassDesc.h`) caches UStruct reflection data lazily. Each descriptor maintains indexed property/function lists and is shared across all Lua instances of that type.

**`UUnLuaManager`** (`Public/UnLuaManager.h`) manages binding lifecycle — object bind/unbind, input component replacement for enhanced input, and Blueprint-callable override helpers.

### Binding System

Two mechanisms for attaching Lua modules to UObjects:

1. **Static binding**: Blueprint implements `UnLuaInterface` and returns a module path from `GetModuleName()`. Bound automatically at object initialization.

2. **Dynamic binding**: Runtime override of specific functions on demand (used for delegates and event handlers). Entry point: `LuaDynamicBinding.h`.

Lua overrides per class are managed by `FLuaOverridesClass` (`Private/LuaOverrides.h`).

### Type System

`ITypeOps` (defined in `UnLuaBase.h`) is the stack-based interface for marshaling values between C++ and Lua:
- `ReadValue()` / `WriteValue()` — Lua stack push/pop
- `ReadValue_InContainer()` / `WriteValue_InContainer()` — for struct fields and container elements

Container types (`Private/Containers/LuaArray.h`, `LuaMap.h`, `LuaSet.h`) access native TArray/TSet/TMap memory directly — no conversion copies.

### Key Design Points

- **Lazy reflection loading**: UStruct/UClass data is loaded only when first accessed from Lua.
- **Persistent parameter buffer** (`ENABLE_PERSISTENT_PARAM_BUFFER`): Reuses allocation across repeated UFUNCTION calls.
- **Dangling pointer detection** (`LuaDanglingCheck.h`): Disallows caching struct/container references across C++→Lua call boundaries.
- **Dead loop detection** (`LuaDeadLoopCheck.h`): Timeout-based guard against infinite loops in Lua (configurable, off by default).
- **Hot reload**: Triggered via `Alt+L` in editor or toolbar. Mode (Auto/Manual/Never) set in editor settings.
- **`UE` namespace** (formerly `UE4`): All UE types accessed in Lua as `UE.ClassName`. Legacy `UE4` namespace supported via `WITH_UE4_NAMESPACE`.

### Lua Script Conventions

Lua modules live under `Content/Script/`. A typical binding module:

```lua
local M = UnLua.Class()

function M:Initialize(Initializer)
    -- called after object construction
end

function M:ReceiveBeginPlay()
    self.Overridden.ReceiveBeginPlay(self)  -- call original Blueprint/C++ impl
end

return M
```

`self.Overridden` is available when `ENABLE_CALL_OVERRIDDEN_FUNCTION=1` (default).

Return order: out-parameters come after the main return value (legacy reversed order available via `UNLUA_LEGACY_RETURN_ORDER`).

## Documentation

- `Docs/EN/UnLua_Programming_Guide.md` — full API and usage guide
- `Docs/EN/How_To_Implement_Overriding.md` — function override patterns
- `Docs/CN/Settings.md` — all editor/runtime settings explained
- `Docs/CN/FAQ.md` — common issues and solutions
- `Content/Script/Tutorials/` — 13 example Lua scripts
