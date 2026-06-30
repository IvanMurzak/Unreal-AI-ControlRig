<h1 align="center">Unreal AI ControlRig</h1>

<p align="center">
  An <b>animation rigging (Control Rig)</b> tool extension for
  <a href="https://github.com/IvanMurzak/Unreal-MCP">AI Game Developer (Unreal-MCP)</a>.
  Lets an AI agent list and inspect Control Rig assets — their rig-element hierarchy (bones,
  controls, nulls, curves) — and inspect an actor's Control Rig component, all from inside the
  Unreal Editor.
</p>

---

**Unreal AI ControlRig** is an Unreal Engine **`Type=Editor` plugin** that implements the Unreal-MCP
contract `IUnrealMcpToolProvider` and contributes a focused family of MCP tools wrapping Unreal's
**ControlRig** plugin. Unreal-MCP discovers the provider at boot (and live, when the plugin loads
later) and merges these tools into the advertised set, so an AI agent can drive animation-rigging
workflows against the live editor. Enabling / disabling the extension live-updates what the AI sees.

> Authoring is **C++** (unlike Unity's C# `[McpPluginTool]`). The extension takes a compile-time
> dependency on the engine's `ControlRig` plugin — that dependency **is the gating**: the extension
> won't compile or load unless Control Rig is present in the host project.

## Tools

This extension contributes the following **read-only** Control Rig inspection tools (ids are
kebab-case, prefixed `control-rig-`; handlers run on the game thread and call ControlRig / editor
APIs directly). Every handler validates its inputs and the engine state it touches and returns a
structured error rather than crashing the editor.

| Tool | Kind | What it does |
| --- | --- | --- |
| `control-rig-list` | read-only | List the Control Rig blueprint assets in the project (Asset Registry; no asset loaded). |
| `control-rig-get` | read-only | Inspect a single Control Rig asset's rig element hierarchy: per-type counts + every element's name & type (bones / controls / nulls / curves). |
| `control-rig-list-controls` | read-only | List a Control Rig's control elements, each with its control type (Float / Transform / Bool / …). |
| `control-rig-list-bones` | read-only | List a Control Rig's bone elements, each with its immediate parent bone. |
| `control-rig-get-component` | read-only | Inspect an actor's `UControlRigComponent` in the editor world: its bound Control Rig class. |

> Each tool ships with one UE Automation spec and one E2E `unreal-mcp-cli` check. `extension.json`
> `tools[]` and this table are the source of truth. The first cut is inspection-only: it reads
> Control Rig assets and components but does not author or mutate them.

## Install

Install into any UE project that has the **UnrealMCP core plugin** available (the project path is a
**positional** argument):

```bash
# From the published GitHub Release:
unreal-mcp-cli install-extension com.ivanmurzak.unreal-ai-control-rig <UEProject>

# Offline / from a local checkout (no published release needed):
unreal-mcp-cli install-extension com.ivanmurzak.unreal-ai-control-rig <UEProject> --source <path-to-this-repo>/UnrealAIControlRig
```

The CLI resolves the release source zip
(`releases/download/v<version>/UnrealAIControlRig-<version>.zip`), drops the plugin into
`<UEProject>/Plugins/UnrealAIControlRig/`, enables it **and** the gating `ControlRig` engine plugin
in the `.uproject`, and the editor compiles it from source on next open (or pass `--build` to compile
now via UBT). The same capability backs the AI-Game-Dev desktop app button and the in-editor
Extensions panel.

## Layout

```
UnrealAIControlRig/                                  the UE plugin
├── UnrealAIControlRig.uplugin                        descriptor; Type=Editor; Plugins: [ UnrealMCP, ControlRig ]
└── Source/UnrealAIControlRig/
    ├── UnrealAIControlRig.Build.cs                   deps: UnrealMcpRuntime + UnrealMcpEditor + ControlRig + ControlRigDeveloper
    └── Private/
        ├── UnrealAIControlRigModule.cpp              the IUnrealMcpToolProvider + module; registers the tools
        └── Tests/UnrealAIControlRigSpec.cpp          UE Automation specs (one It(...) per tool)
commands/                                             bump-version / get-version / update-core / init
Tests/e2e/                                            E2E unreal-mcp-cli tool checks (one per tool)
extension.json                                        install-catalog / compatibility manifest
.github/workflows/                                    CI: test_pull_request + release (+ reusable test_unreal_plugin)
```

## Develop locally

The fastest loop is a directory junction into the UE 5.7 testbed (which already has `Plugins/UnrealMCP`):

```powershell
# Junction this plugin into a UE C++ project that has Plugins/UnrealMCP available:
cmd /c mklink /J "<UEProject>\Plugins\UnrealAIControlRig" "<thisRepo>\UnrealAIControlRig"

# Build the editor target with UBT:
& "C:\Program Files\Epic Games\UE_5.7\Engine\Build\BatchFiles\Build.bat" `
  <UEProject>Editor Win64 Development -project="<UEProject>\<UEProject>.uproject" -WaitMutex

# Run this extension's Automation specs (filter = the module name):
& "C:\Program Files\Epic Games\UE_5.7\Engine\Binaries\Win64\UnrealEditor-Cmd.exe" `
  "<UEProject>\<UEProject>.uproject" -nullrhi -nosplash -unattended `
  -ExecCmds="Automation RunTests UnrealAIControlRig; Quit" -ReportExportPath="<dir>" -log
```

Enable both plugins in the project, open the editor, and connect AI Game Developer (the Unreal-MCP
UI / sidecar); `StartupModule` registers the provider as a modular feature, so the Control Rig tools
appear in the tool list immediately. See the
[Unreal-MCP extension author guide](https://github.com/IvanMurzak/Unreal-MCP/blob/main/docs/EXTENSIONS.md).

## Release

Versioning is single-sourced from the `.uplugin` `VersionName`. Bump it in lock-step:

```powershell
./commands/bump-version.ps1 -NewVersion "0.2.0"   # updates .uplugin + GetExtensionVersion() + extension.json
```

Push to `main`. **`release.yml` is version-gated**: when the `VersionName` is a new value with no
existing tag, it runs the full test suite, packages the plugin **source** into a single
`UnrealAIControlRig-<version>.zip`, and creates an **atomic GitHub Release** (tag `v<version>`)
carrying that one zip — the exact asset the installer downloads. The extension ships as source and UE
compiles it on the consumer's next editor open. (Track the core version floor with
`./commands/update-core.ps1`.)

## CI

| Workflow | When | What |
| --- | --- | --- |
| `test_unreal_plugin.yml` | reusable | UBT host-editor build + UE Automation specs for one UE version (5.7) |
| `test_pull_request.yml` | PR | the reusable test (UE 5.7) + E2E `unreal-mcp-cli` tool checks |
| `release.yml` | push to `main` | version-gated → full tests → package source zip `UnrealAIControlRig-<version>.zip` → atomic GitHub Release (tag `v<version>`) |
| `bump_version.yml` | manual | runs `bump-version.ps1`, opens a release PR |

The plugin / E2E jobs run on a **self-hosted Windows UE runner** and are **never red-by-absence** —
they stay *skipped* until a runner is registered and the repo variables are set:

- `UNREAL_RUNNER_READY = true` — enables the UBT build + Automation legs.
- `UNREAL_E2E_READY = true` — enables the E2E `install-extension` + tool-invocation leg.
- `UNREAL_HOST_PROJECT` — absolute path on the runner to a host `.uproject` with UnrealMCP available.

See [`docs/claude/ci.md`](docs/claude/ci.md) and [`docs/claude/release.md`](docs/claude/release.md).

## License

[Apache-2.0](LICENSE).
