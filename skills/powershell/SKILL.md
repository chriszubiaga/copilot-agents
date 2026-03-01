---
name: powershell
description: 'Deep knowledge for writing, reviewing, and debugging PowerShell scripts on Windows (5.1) and cross-platform (PowerShell Core 7+). Covers error handling, cmdlet conventions, pipelines, modules, security, and common automation idioms. Load when the user is working with .ps1, .psm1, or .psd1 files, or shell automation targeting Windows or PowerShell Core.'
---

# PowerShell Scripting Skill

## Scope

This skill applies to **Windows PowerShell 5.1** and **PowerShell Core 7+** (Linux/macOS/Windows). When cross-platform compatibility is required, target PS 7+ and document any 5.1-only or Windows-only constraints explicitly.

## WSL Execution (this workspace)

This workspace runs inside WSL. PowerShell is available via the Windows bridge — always invoke it as:

```bash
powershell.exe -ep bypass <command>
# or for a script file:
powershell.exe -ep bypass -File ./script.ps1
# or for an inline command:
powershell.exe -ep bypass -Command "Get-Service | Where-Object Status -eq Running"
```

- `-ep bypass` sets ExecutionPolicy to Bypass for the session (required when running unsigned scripts).
- Do **not** use `pwsh` or `powershell` as bare commands unless the user confirms a native Linux PS install exists.
- Script paths passed to `powershell.exe` from WSL must use Windows paths (`C:\...`) or WSL-converted paths via `wslpath -w`.

---

## Script Header Requirements

Every generated script should start with:

```powershell
#Requires -Version 7.0
Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'
```

- `#Requires -Version` — enforces the minimum PS version at parse time.
- `Set-StrictMode -Version Latest` — catches uninitialized variables, bad property references, etc.
- `$ErrorActionPreference = 'Stop'` — makes non-terminating errors terminating (equivalent to `set -e`).

For Windows-only scripts that must support PS 5.1, use `#Requires -Version 5.1`.

---

## Error Handling

### try/catch/finally

```powershell
try {
    $result = Invoke-RestMethod -Uri $Uri -Method Get
}
catch [System.Net.WebException] {
    Write-Error "Network error: $_"
    exit 1
}
catch {
    Write-Error "Unexpected error: $_"
    exit 1
}
finally {
    # cleanup always runs
}
```

### Propagating errors from native commands

```powershell
# PS 7+: native command errors throw automatically
$ErrorActionPreference = 'Stop'
git status   # throws if git exits non-zero

# PS 5.1: must check manually
git status
if ($LASTEXITCODE -ne 0) { throw "git status failed ($LASTEXITCODE)" }
```

### Retry logic

```powershell
function Invoke-WithRetry {
    param(
        [ScriptBlock]$Action,
        [int]$Attempts = 3,
        [int]$DelaySeconds = 5
    )
    for ($i = 1; $i -le $Attempts; $i++) {
        try   { & $Action; return }
        catch {
            if ($i -ge $Attempts) { throw }
            Write-Warning "Attempt $i failed. Retrying in ${DelaySeconds}s..."
            Start-Sleep -Seconds $DelaySeconds
        }
    }
}
# Usage: Invoke-WithRetry -Action { Invoke-RestMethod $Uri }
```

---

## Cmdlet / Function Conventions

### CmdletBinding and Parameter Blocks

```powershell
function Invoke-Deployment {
    [CmdletBinding(SupportsShouldProcess, ConfirmImpact='High')]
    param(
        [Parameter(Mandatory, ValueFromPipeline)]
        [string]$TargetPath,

        [Parameter()]
        [ValidateSet('dev','staging','prod')]
        [string]$Environment = 'dev',

        [switch]$Force
    )
    process {
        if ($PSCmdlet.ShouldProcess($TargetPath, "Deploy to $Environment")) {
            # actual work
        }
    }
}
```

- Use `Verb-Noun` naming following [approved verbs](https://learn.microsoft.com/en-us/powershell/scripting/developer/cmdlet/approved-verbs-for-windows-powershell-commands).
- Always add `[CmdletBinding()]` for functions that accept pipeline input or should support `-Verbose`/`-WhatIf`.
- Use `[Parameter(Mandatory)]` over manual `if (-not $Param)` checks.
- Add `[ValidateSet]`, `[ValidateNotNullOrEmpty]`, or `[ValidateRange]` where applicable.

---

## Output: The Right Cmdlet for the Job

| Scenario                        | Use                    | Avoid               |
| ------------------------------- | ---------------------- | ------------------- |
| Structured data for pipeline    | `Write-Output`         | `Write-Host`        |
| User-facing progress/status     | `Write-Host` or `Write-Information` | `Write-Output` |
| Debug info (hidden by default)  | `Write-Debug`          | `Write-Host`        |
| Non-fatal warnings              | `Write-Warning`        | `Write-Host`        |
| Verbose info                    | `Write-Verbose`        | `Write-Host`        |
| Errors                          | `Write-Error` / `throw`| `Write-Host`        |

Never use `Write-Host` for data the caller may need to consume — it bypasses the pipeline.

---

## Pipeline Patterns

```powershell
# Filter and transform
Get-ChildItem -Path $SourceDir -Filter '*.log' -Recurse |
    Where-Object { $_.LastWriteTime -lt (Get-Date).AddDays(-30) } |
    Select-Object Name, LastWriteTime, @{N='SizeKB';E={[math]::Round($_.Length/1KB,1)}} |
    Sort-Object LastWriteTime |
    Export-Csv -Path old_logs.csv -NoTypeInformation

# Parallel (PS 7+ only)
$items | ForEach-Object -Parallel {
    process_item $_
} -ThrottleLimit 4
```

---

## Credential and Secret Handling

```powershell
# Prompt securely — never hardcode
$cred = Get-Credential -Message "Enter service account credentials"

# Read a secret from environment (prefer over in-script literals)
$token = [System.Environment]::GetEnvironmentVariable('MY_API_TOKEN','User')
if (-not $token) { throw 'MY_API_TOKEN environment variable is not set.' }

# Use SecureString for sensitive in-memory values
$secureToken = ConvertTo-SecureString $token -AsPlainText -Force
```

Never store plain-text passwords in scripts, version control, or log output.

---

## Dry-Run (WhatIf) Pattern

```powershell
function Remove-OldFiles {
    [CmdletBinding(SupportsShouldProcess)]
    param([string]$Path)
    Get-ChildItem $Path | ForEach-Object {
        if ($PSCmdlet.ShouldProcess($_.FullName, 'Remove')) {
            Remove-Item $_.FullName -Force
        }
    }
}
# Caller uses: Remove-OldFiles -Path C:\Temp -WhatIf
```

Always add `SupportsShouldProcess` on functions that modify state.

---

## File and Path Handling

```powershell
# Prefer Join-Path over string concatenation
$fullPath = Join-Path $BasePath 'logs' 'app.log'

# Test for existence before operating
if (-not (Test-Path -LiteralPath $fullPath)) {
    throw "File not found: $fullPath"
}

# Get script's own directory (works in PS 5.1 and 7+)
$ScriptDir = $PSScriptRoot   # NOT $MyInvocation.MyCommand.Path

# Resolve to absolute path
$resolved = Resolve-Path -LiteralPath $relativePath
```

---

## Modules

### Module structure

```
MyModule/
  MyModule.psd1      # module manifest
  MyModule.psm1      # root module
  Public/            # exported functions
  Private/           # internal functions
  tests/             # Pester tests
```

### Loading patterns

```powershell
# Explicitly import with version requirement
Import-Module MyModule -MinimumVersion 2.0 -ErrorAction Stop

# Check if module is available before importing
if (-not (Get-Module -ListAvailable -Name MyModule)) {
    Install-Module MyModule -Scope CurrentUser -Force
}
```

---

## Windows vs PowerShell Core (7+) Compatibility

| Feature                         | PS 5.1 (Windows only)          | PS 7+ (cross-platform)         |
| ------------------------------- | ------------------------------ | ------------------------------ |
| `$PSVersionTable.OS`            | Not available                  | Available                      |
| Native cmd error throws         | No (`$LASTEXITCODE` required)  | Yes (`$ErrorActionPreference`) |
| `ForEach-Object -Parallel`      | Not available                  | Available                      |
| `??` and `?.` null operators    | Not available                  | Available                      |
| Windows-only cmdlets            | `Get-WmiObject`, `Get-EventLog`| Use `Get-CimInstance`, `Get-WinEvent` |
| COM objects                     | Available                      | Windows only                   |

### Platform detection

```powershell
if ($IsWindows) { <# Windows-specific #> }
elseif ($IsLinux) { <# Linux-specific #> }
elseif ($IsMacOS) { <# macOS-specific #> }
# $IsWindows / $IsLinux / $IsMacOS exist in PS 6+; in PS 5.1 always Windows
```

---

## Logging Pattern

> The format, levels, and color rules below implement the workspace-wide standard. See `.github/skills/cli-output/SKILL.md` for the canonical spec and ready-to-paste implementation.

```powershell
function Write-Log {
    param(
        [ValidateSet('INFO','WARN','ERROR','DEBUG')][string]$Level = 'INFO',
        [string]$Message
    )
    $ts = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    $entry = "[$ts] [$Level] $Message"
    switch ($Level) {
        'ERROR' { Write-Error $entry }
        'WARN'  { Write-Warning $entry }
        'DEBUG' { Write-Debug $entry }
        default { Write-Information $entry -InformationAction Continue }
    }
}
```

---

## Dependency / Prerequisite Checks

```powershell
function Assert-Command {
    param([string[]]$Commands)
    foreach ($cmd in $Commands) {
        if (-not (Get-Command $cmd -ErrorAction SilentlyContinue)) {
            throw "Required command not found: $cmd"
        }
    }
}
Assert-Command git, dotnet, az
```

---

## Security Checklist

- [ ] No hardcoded credentials, tokens, or connection strings.
- [ ] All user input validated via `[ValidateSet]`, `[ValidateRange]`, or explicit guards.
- [ ] No `Invoke-Expression` with untrusted input.
- [ ] `ExecutionPolicy` noted in documentation — never set to `Bypass` in production.
- [ ] `SupportsShouldProcess` added to any function that modifies state.
- [ ] Secrets sourced from environment variables or credential stores (not literals).
- [ ] No `ConvertTo-SecureString ... -AsPlainText` with hardcoded strings.

---

## Behavioral Conventions

- Run `[CmdletBinding()]` functions with `-WhatIf` first to preview changes; always document this.
- Prefer `-LiteralPath` over `-Path` when paths may contain wildcard characters.
- Use `$PSCmdlet.WriteError()` inside advanced functions instead of bare `Write-Error` for proper stream handling.
- Keep scripts idempotent where possible — re-running should be safe.
- Always test on the minimum required PS version declared in `#Requires`.
