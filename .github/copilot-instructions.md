# AMP Templates Development Guide

## Project Overview
This repository contains game server configuration templates for [CubeCoders AMP](https://cubecoders.com/AMP) (Application Management Panel). Each template defines how AMP manages, configures, and monitors a specific game server application.

## Template File Structure
Every game server template consists of **three core files** (named in lowercase):
- `<appname>.kvp` - Main configuration defining server metadata, executable paths, ports, console behavior, and update mechanisms
- `<appname>config.json` - Settings manifest defining user-configurable parameters that appear in AMP's UI
- `<appname>metaconfig.json` (optional) - Maps settings to external config files (e.g., mod configs)

Additional files may include:
- `<appname>ports.json` - Port definitions (can be inline in KVP via `@IncludeJson[]`)
- `<appname>updates.json` - Multi-stage update pipeline (can be inline in KVP via `@IncludeJSON[]`)
- Config file examples (`.cfg`, `.ini`, `.json`, etc.)
- Management scripts (`.sh`, `.ps1`) for complex mod/workshop operations

## KVP File Structure
The `.kvp` file uses key-value pairs with sections:
- **Meta.*** - Template metadata (DisplayName, OS support, Docker requirements, etc.)
- **App.*** - Application configuration (executables, working directories, command-line args, ports)
- **Console.*** - Regex patterns for parsing server output (ready state, player joins/leaves, updates)
- **Limits.*** - Sleep mode and retry settings

### Key Conventions
- Use `{{Variable}}` for AMP variable substitution (e.g., `{{ServerName}}`, `{{$ApplicationPort1}}`)
- Reference external JSON with `@IncludeJson[filename]` or `@IncludeJSON[filename]`
- Set `App.ApplicationIPBinding=0.0.0.0` for IP binding, `App.AdminMethod=STDIO` for console-based RCON
- Define regex in `Console.AppReadyRegex` to detect when server is running
- Player tracking uses `Console.UserJoinRegex` and `Console.UserLeaveRegex` with named capture groups `(?<username>.+?)` and `(?<userid>.+?)`

## Config Manifest (config.json)
Defines user-facing settings with:
- **FieldName** - Internal variable name (referenced in KVP via `{{FieldName}}`)
- **InputType** - UI control type: `text`, `number`, `checkbox`, `enum`, `list`, `password`
- **ParamFieldName** - Command-line parameter name when `IncludeInCommandLine=true`
- **Category/Subcategory** - UI organization (format: `DisplayName:icon:order`)
- **Special** - Advanced bindings like `listfile:path/to/file.txt` for managing line-separated lists
- **EnumValues** - Maps display text to actual values for dropdowns
- **DefaultValue** - Initial value when creating new instance

Example pattern for game port parameter:
```json
{
  "DisplayName": "Game Port",
  "FieldName": "$ApplicationPort1",
  "InputType": "number",
  "ParamFieldName": "port",
  "IncludeInCommandLine": true,
  "DefaultValue": "7777"
}
```

## Update Pipeline (updates.json)
Multi-stage update array with common sources:
- **SteamCMD** - `UpdateSourceData` = app ID, `UpdateSourceArgs` = login app ID (or blank for anonymous)
- **GithubRelease** - `UpdateSourceArgs` = `owner/repo`, `UpdateSourceData` = asset filename
- **Executable** - Run scripts during updates (must output to stdout for AMP tracking)
- **FetchURL** - Direct file downloads

Set `UpdateSourceConditionSetting` to only run stages when a specific setting is enabled.

## Common Patterns

### SteamCMD-based Games
Most templates use SteamCMD for downloads. In [valheim.kvp](valheim.kvp):
```plaintext
App.ExecutableLinux=896660/valheim_server.x86_64
App.BaseDirectory=./Valheim/896660/
```
Where `896660` is the Steam app ID for the dedicated server.

### Multi-Port Configuration
Define in [valheimports.json](valheimports.json) with `ChildPorts` array for related ports (e.g., query port offset from game port).

### Mod Management
Complex games like ARK use shell scripts (see [ark-se-minapimanagemods.sh](ark-se-minapimanagemods.sh)) to handle Workshop mod downloads, `.mod` file generation, and installation. Scripts receive input via stdin in JSON format.

### Platform-Specific Behavior
- Use `App.LinuxCommandLineArgs` / `App.WindowsCommandLineArgs` for OS-specific flags
- Set `UpdateSourcePlatform` to `Windows`, `Linux`, or `All` in update stages
- Use Wine/Proton for Windows-only games on Linux (see Space Engineers templates)

## Development Workflow

### Testing Requirements
Before submitting PR, ensure template passes automated tests on **both Windows and Linux**:
1. Load configuration successfully
2. Perform update (download game server files)
3. Start application
4. Reach 'Ready' state (via `Console.AppReadyRegex`)
5. Stop application gracefully
6. Verify 'Stopped' state

### Naming Conventions
- All filenames **must be lowercase**
- Use hyphens for multi-word names: `american-truck-simulator.kvp`
- For variants, append suffix after hyphen: `ark-se-min.kvp`, `counter-strike2.kvp`

### Pull Request Guidelines
- Place all files in repository root (no subdirectories)
- Submit only the 3 core files + any required config examples
- Include "(draft)" in PR title for work-in-progress
- Contact original template author for edits to existing templates
- Applications must not require authentication for downloads (except Steam login)
- Must expose console output that AMP can parse
- Do not invoke shell scripts from command line (only from update stages)

## Key Constraints
- **No shell script execution** in `App.CommandLineArgs` - launch executables directly
- Templates requiring authentication (except Steam) are not accepted
- Games without console output cannot be managed
- Settings manifests are required for applications with configurable parameters

## Online Tools
- Basic configurator: https://config.getamp.sh/
- Advanced configurator (beta): https://iceofwraith.github.io/GenericConfigGen/
- **Note:** Generated templates require manual refinement and will not be accepted as-is

## Branch Information
- Default branch: `main` - stable, production templates
- Current branch: `maint` - maintenance and updates
