---
allowed-tools: Read, Write, Bash
description: Generate a standalone PowerShell script for N-Sight RMM Script Manager
argument-hint: "description of what the script should do"
disable-model-invocation: false
---

Generate a standalone PowerShell script for direct upload to N-Sight RMM Script Manager (Settings > Script Manager). These scripts run as Script Checks (24x7/Daily monitoring) or Automated Tasks (maintenance jobs).

## Step 1: Determine Script Type

From the user's description, determine:
- **Script Check** — monitoring/verification that runs on a schedule. Exit 0 = pass (green), non-zero = fail (red).
- **Automated Task** — maintenance/remediation that runs on demand or schedule.

If unclear, ask the user.

## Step 2: Generate the Script

Follow these N-Sight RMM conventions strictly:

### Required Elements

1. **Comment-based help** — .SYNOPSIS, .DESCRIPTION, .NOTES (Author, Date, Version, Target, Context, PS Version)
2. **Error preferences** — `$ErrorActionPreference = 'Stop'` and `Set-StrictMode -Version Latest`
3. **Logging** — Write-Log function with dual output (file + Write-Host for Dashboard)
4. **Log path** — `$LogPath = "C:\DatoLogs\{ScriptName}"`
5. **try/catch** — main logic in try block, error handling in catch
6. **Exit codes** — `Exit 0` for success, `Exit 1001` for general failure, `Exit 1002+` for specific failures

### Critical Constraints

| Constraint | Rule |
|-----------|------|
| Max script size | 65,535 characters |
| Max output | 10,000 characters on Dashboard |
| Exit codes 1-999 | NEVER USE — reserved by N-Sight system scripts |
| `param()` block | DO NOT USE — N-Sight Script Manager doesn't support it |
| `Read-Host` | DO NOT USE — runs as SYSTEM, no user interaction |
| Paths | Always absolute — runs from unpredictable working directory |
| Context | Runs as `NT AUTHORITY\SYSTEM` — no user profile, no HKCU, no mapped drives |
| `$env:USERPROFILE` | Points to `C:\Windows\system32\config\systemprofile` in SYSTEM context |
| Network paths | Use UNC paths (`\\server\share`) — no mapped drive letters |

### Script Template

```powershell
<#
.SYNOPSIS
    {Brief description}
.DESCRIPTION
    {Detailed description}. Designed for automated execution via N-Sight RMM.
    Runs as SYSTEM with no user interaction.
.NOTES
    Author:      DatoTech
    Date:        {YYYY-MM-DD}
    Version:     1.0.0
    Target:      N-Sight RMM {Script Check|Automated Task}
    Context:     Runs as SYSTEM
    PS Version:  5.1+
#>

$ErrorActionPreference = 'Stop'
Set-StrictMode -Version Latest

# Configuration
$LogPath = "C:\DatoLogs\{ScriptName}"
$LogFile = "$LogPath\$env:COMPUTERNAME.log"

# Helper Functions
function Write-Log {
    param ([string]$Message, [string]$Level = "INFO")
    $timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    $logEntry = "$timestamp [$Level] $Message"
    Add-Content -Path $LogFile -Value $logEntry -ErrorAction SilentlyContinue
    Write-Host $logEntry
}

# Main Logic
try {
    if (-not (Test-Path $LogPath)) {
        New-Item -ItemType Directory -Path $LogPath -Force | Out-Null
    }
    Write-Log "Script started on $env:COMPUTERNAME"

    # --- Your logic here ---

    Write-Log "Operation completed successfully" -Level "SUCCESS"
    Exit 0
}
catch {
    $errorMsg = $_.Exception.Message
    Write-Log "An error occurred: $errorMsg" -Level "ERROR"
    Write-Log "Stack Trace: $($_.ScriptStackTrace)" -Level "DEBUG"
    Exit 1001
}
```

### Best Practices

**Error handling:**
- Capture `$_.Exception.Message` immediately in catch blocks
- Use `-ErrorAction SilentlyContinue` only on specific cmdlets where failure is expected

**Output:**
- Prefix with `[INFO]`, `[SUCCESS]`, `[ERROR]`, `[WARN]`
- Include `$env:COMPUTERNAME` for multi-device clarity

**File operations:**
- `Test-Path` before operating
- `Join-Path` for path construction
- `New-Item -ItemType Directory -Force | Out-Null` for creating directories

**Process execution:**
- `Start-Process -Wait -PassThru -NoNewWindow` for external executables
- Capture exit codes: `$proc.ExitCode`
- Redirect stdout/stderr to temp files if needed

**Registry:**
- Use `HKLM:\` for machine-wide settings
- `New-Item -Path -Name -Force` for keys
- `New-ItemProperty -PropertyType DWORD|String -Force` for values

**Service management:**
- `Get-Service -Name $svc -ErrorAction SilentlyContinue` to check existence
- `Restart-Service -Name $svc -Force` with appropriate error handling

**SYSTEM context:**
- No `$env:USERPROFILE` (points to systemprofile)
- No HKCU access — use HKLM
- winget needs: `(Resolve-Path "$env:ProgramFiles\WindowsApps\Microsoft.DesktopAppInstaller_*_x64_*\winget.exe").Path`
- quser.exe needs `AllowRemoteRPC` registry key

## Step 3: Write the Script

Write the .ps1 file to the user's specified location. If no location specified, write to the current directory with a descriptive filename in PascalCase (e.g., `CheckServiceStatus.ps1`, `CleanTempFiles.ps1`).
