# Unreal-AI-ControlRig

This is the **Unreal AI ControlRig** extension — a C++ `Type=Editor` Unreal Engine plugin that
implements the Unreal-MCP contract `IUnrealMcpToolProvider` and contributes a family of MCP tools
wrapping the engine's **ControlRig** (animation rigging) plugin to AI Game Developer (Unreal-MCP).
It was scaffolded from `IvanMurzak/Unreal-AI-Template`; the plugin/module is `UnrealAIControlRig` (UE
module names can't contain `-`, so the repo's `Unreal-AI-ControlRig` becomes `UnrealAIControlRig`).

The dependency on the `ControlRig` engine plugin (a `.Build.cs` module dep + a `.uplugin` `Plugins[]`
entry) **is the gating**: this extension won't compile or load unless Control Rig is present in the
host project.

## The tools

The provider (`FUnrealAIControlRigProvider` in
`UnrealAIControlRig/Source/UnrealAIControlRig/Private/UnrealAIControlRigModule.cpp`) registers a
**read-only** Control Rig inspection family via the fluent `Registry.Tool(...).Handle(...)` builder:

- `control-rig-list` — list Control Rig blueprint assets (Asset Registry, no asset loaded), read-only.
- `control-rig-get` — inspect a single Control Rig asset's rig element hierarchy (per-type counts +
  every element's name & type), read-only.
- `control-rig-list-controls` — list a Control Rig's control elements + each control's type, read-only.
- `control-rig-list-bones` — list a Control Rig's bone elements + each bone's immediate parent, read-only.
- `control-rig-get-component` — inspect an actor's `UControlRigComponent` (its bound rig class), read-only.

`extension.json` `tools[]` + the README table are the source of truth, and each tool ships one UE
Automation spec + one E2E check. Handlers run on the **game thread** and call ControlRig / editor APIs
directly; each validates its inputs + engine state defensively (UE has no C++ exceptions — a crash in a
handler is an editor crash) and returns `FUnrealMcpToolResult::Error(...)`, never an unchecked deref.
The first cut is inspection-only — it reads Control Rig assets/components but does not author or mutate
them (asset creation is an editor-module-heavy, headless-brittle operation deliberately left out).

## The contract (read before editing tools)

- `IUnrealMcpToolProvider` (in Unreal-MCP `UnrealMcpRuntime/Public/IUnrealMcpToolProvider.h`):
  `GetExtensionId()` / `GetDisplayName()` / `GetExtensionVersion()` / `RegisterTools(FUnrealMcpToolRegistry&)`.
- Tools are declared with the fluent builder: `Registry.Tool("kebab-id").Title(...).Param*(...).Handle([](const FUnrealMcpToolCall&){...})`.
- The provider is registered as a **modular feature** in `StartupModule` and unregistered in
  `ShutdownModule`. Unreal-MCP discovers it on boot or live.
- Handlers run on the **game thread** (call editor/engine APIs directly). Tool ids MUST match
  `^[a-z0-9]+(-[a-z0-9]+)*$` or the registry drops them. Do NOT call `.ExtensionId(...)` — it's stamped.

## Commands

```powershell
./commands/bump-version.ps1 -NewVersion "0.2.0"   # .uplugin VersionName + GetExtensionVersion() + extension.json
./commands/get-version.ps1                        # prints the .uplugin VersionName (single source of truth)
./commands/update-core.ps1                        # refreshes extension.json minCoreVersion from Unreal-MCP releases
```

## Build / test (local loop)

Junction `UnrealAIControlRig/` into a UE 5.7 project that has the UnrealMCP core plugin available (the
`engines/unreal/test-project` testbed already junctions `Plugins/UnrealMCP`), then build the editor
target with UBT and run the Automation specs with filter = the module name (`UnrealAIControlRig`). See
`README.md` → "Develop locally". CI does the same on a self-hosted Windows UE runner.

## Conventions

- **Naming:** repo `Unreal-AI-ControlRig` (hyphens); plugin + module `UnrealAIControlRig` (no hyphens
  — UE module names can't contain `-`); C++ prefixes `F*`/`U*`/`I*`; tool ids kebab-case
  `control-rig-<op>`.
- **C++ style:** Unreal — tabs, braces on new lines, UE types. File header: the
  `// Copyright (c) 2026 ...` Apache-2.0 one-liner. The module is **unity-built** (every `.cpp` is
  concatenated into one TU), so give file-local helpers a module-unique name (prefix
  `UnrealAIControlRig_`) — an `anonymous namespace` does NOT make a helper file-private here.
- **Versioning:** the `.uplugin` `VersionName` is the single source of truth; never hand-edit one
  version location alone — use `bump-version.ps1`. Keep `GetExtensionVersion()` == the `VersionName`.
- **Tests:** one UE Automation spec + one E2E `unreal-mcp-cli` check **per tool**.
- **Distribution:** GitHub-Release source zip `UnrealAIControlRig-<version>.zip` at tag `v<version>`;
  UE compiles on the consumer's next editor open. NOT a NuGet package, NOT precompiled binaries.
- **Secrets:** never commit `.env` or tokens.

## Find detail in

- `README.md` — the user-facing tools / install / develop / release / CI walkthrough.
- `docs/claude/architecture.md` — extension shape, the contract, layout.
- `docs/claude/ci.md` — workflows, required repo variables, self-hosted runner gating.
- `docs/claude/release.md` — version gate + atomic release mechanics.
