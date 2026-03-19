# N-Sight RMM PowerShell Conventions

## PowerShell Embedding in .amp Files

Embedded PowerShell scripts inside `.amp` XML files must be encoded as **UTF-16LE then Base64**.

### Encoding

**PowerShell:**
```powershell
$script = Get-Content -Path "script.ps1" -Raw
$bytes = [System.Text.Encoding]::Unicode.GetBytes($script)
$encoded = [Convert]::ToBase64String($bytes)
```

**Python:**
```python
with open("script.ps1", "r", encoding="utf-8") as f:
    script = f.read()
encoded = base64.b64encode(script.encode("utf-16-le")).decode("ascii")
```

**Bash:**
```bash
iconv -f UTF-8 -t UTF-16LE < script.ps1 | base64 -w 0
```

### Decoding

**PowerShell:**
```powershell
$decoded = [System.Text.Encoding]::Unicode.GetString([Convert]::FromBase64String($encoded))
```

**Python:**
```python
decoded = base64.b64decode(encoded).decode("utf-16-le")
```

**Bash:**
```bash
echo "$encoded" | base64 -d | iconv -f UTF-16LE -t UTF-8
```

### Embedded Script Rules

- **NEVER** use `$Result` or `$ResultString` in embedded scripts. These variables are reserved by the N-Sight PolicyExecutor and will be silently overwritten, causing unpredictable behavior.
- Parameters are injected via InArgs mapping and appear inside the script as `$VariableName`.
- `Write-Host` output is captured into the `OutPut_64` variable in the .amp execution context.
- The exit code is captured into the `Result` variable (type `Double`) in the .amp execution context.
- Maximum script size: **10 MB** (after encoding).
- **Avoid using `Exit`** in embedded scripts. Let PolicyExecutor manage results. Use `If`/`Else` branching with Fail Policy nodes to surface pass/fail status.

---

## N-Sight Exit Code Convention

| Exit Code | Meaning |
|-----------|---------|
| `0` | Success (green check on Dashboard) |
| `1001` | General failure |
| `1002+` | Specific failures (define per script) |
| `1-999` | **RESERVED** by N-Sight system scripts. **NEVER USE.** |

In `.amp` files, exit codes are consumed internally by the policy engine. Use `If`/`Else` decision nodes combined with `Fail Policy` to surface pass/fail status to the Dashboard.

---

## Output Mechanism

| Method | Behavior |
|--------|----------|
| `Write-Host` | Primary output method. Displayed on the N-Sight Dashboard. |
| `Write-Output` | Works for pipeline output. Also captured by PolicyExecutor. |
| `Write-Error` | Writes to stderr. **Not shown on Dashboard.** |

### Size Limits

- **Dashboard display:** max 10,000 characters.
- **Script Manager uploads:** max 65,535 characters.

---

## Standalone Script Template (Script Manager)

Use this template for scripts uploaded directly to the N-Sight Script Manager.

```powershell
<#
.SYNOPSIS
    Brief description of what the script does.

.DESCRIPTION
    Detailed description of the script's purpose, behavior, and any
    prerequisites or dependencies.

.NOTES
    Author  : Your Name
    Date    : YYYY-MM-DD
    Version : 1.0.0
    Target  : Windows 10/11, Server 2016+
    Context : SYSTEM
    PS Ver  : 5.1+
#>

$ErrorActionPreference = 'Stop'
Set-StrictMode -Version Latest

$ScriptName = "ScriptName"
$LogPath = "C:\DatoLogs\$ScriptName"

if (-not (Test-Path $LogPath)) {
    New-Item -Path $LogPath -ItemType Directory -Force | Out-Null
}

$LogFile = Join-Path $LogPath "$ScriptName-$(Get-Date -Format 'yyyyMMdd-HHmmss').log"

function Write-Log {
    param(
        [Parameter(Mandatory)]
        [string]$Message,

        [ValidateSet('INFO', 'SUCCESS', 'WARN', 'ERROR')]
        [string]$Level = 'INFO'
    )
    $timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    $entry = "[$timestamp] [$Level] $Message"
    Add-Content -Path $LogFile -Value $entry
    Write-Host $entry
}

try {
    Write-Log "Starting $ScriptName on $env:COMPUTERNAME"

    # === Main logic here ===

    Write-Log "Completed successfully" -Level SUCCESS
    Exit 0
}
catch {
    Write-Log "FATAL: $_" -Level ERROR
    Write-Log $_.ScriptStackTrace -Level ERROR
    Exit 1001
}
```

---

## Embedded Script Template (.amp)

Use this template for PowerShell scripts embedded inside `.amp` policy files.

```powershell
<#
.SYNOPSIS
    Brief description of what the embedded script does.

.DESCRIPTION
    Detailed description. This script runs inside an .amp policy
    and must not use $Result, $ResultString, or Exit.
#>

$ErrorActionPreference = 'Stop'

try {
    # === Main logic here ===
    # Parameters come from InArgs mapping as $VariableName

    Write-Host "[SUCCESS] Operation completed on $env:COMPUTERNAME"
}
catch {
    Write-Host "[ERROR] $_"
    Write-Host "[ERROR] $($_.ScriptStackTrace)"
}
```

Key points:
- **NO** `$Result` or `$ResultString` (reserved by PolicyExecutor).
- **NO** `Exit` statement (let PolicyExecutor handle result flow).
- Use `Write-Host` for all output.

---

## Key Differences: Standalone vs Embedded

| Aspect | Standalone (Script Manager) | Embedded (.amp) |
|--------|---------------------------|-----------------|
| Exit codes | `Exit 0` / `Exit 1001` / `Exit 1002+` | No `Exit`. PolicyExecutor manages results. |
| Parameters | Hardcoded or script parameters | Injected via InArgs mapping as `$VariableName` |
| Reserved vars | None | `$Result`, `$ResultString` are **off-limits** |
| Output | `Write-Host` + log file | `Write-Host` only (captured to `OutPut_64`) |
| Max size | 65,535 chars (Script Manager) | 10 MB (after Base64 encoding) |
| `Exit` usage | Required (`Exit 0` / `Exit 1001`) | **Never use `Exit`** |
| Error surfacing | Exit code shown on Dashboard | Use If/Else + Fail Policy nodes |

---

## Script Structure Tiers

### Tier 1: Inline

No structure. Direct commands for trivial operations.

```powershell
# DisablePin example
New-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\PassportForWork" `
    -Name "Enabled" -Value 0 -PropertyType DWord -Force
Write-Host "[SUCCESS] Windows Hello PIN disabled on $env:COMPUTERNAME"
```

### Tier 2: Structured

Comment-based help, `try`/`catch`, and `Write-Log` function.

```powershell
<#
.SYNOPSIS
    Run DISM and SFC scans.
#>

$ErrorActionPreference = 'Stop'

function Write-Log {
    param([string]$Message, [string]$Level = 'INFO')
    Write-Host "[$Level] $Message"
}

try {
    Write-Log "Starting DISM/SFC on $env:COMPUTERNAME"

    $dism = Start-Process -FilePath "DISM.exe" `
        -ArgumentList "/Online /Cleanup-Image /RestoreHealth" `
        -Wait -PassThru -NoNewWindow
    Write-Log "DISM exit code: $($dism.ExitCode)"

    $sfc = Start-Process -FilePath "sfc.exe" `
        -ArgumentList "/scannow" `
        -Wait -PassThru -NoNewWindow
    Write-Log "SFC exit code: $($sfc.ExitCode)"

    Write-Log "Scans completed" -Level SUCCESS
    Exit 0
}
catch {
    Write-Log "FATAL: $_" -Level ERROR
    Exit 1001
}
```

### Tier 3: Modular

`Build.ps1` pattern that concatenates helper functions from separate files into a single deployable script.

- Shared helpers (logging, validation, registry operations) live in separate `.ps1` files.
- A `Build.ps1` script reads and concatenates them in dependency order.
- The output is a single self-contained script suitable for Script Manager or .amp embedding.

---

## Best Practices

### Error Handling

- Always set `$ErrorActionPreference = 'Stop'` at the top of the script.
- Always set `Set-StrictMode -Version Latest` in standalone scripts.
- Wrap all logic in `try`/`catch`.
- Log the full error and stack trace in `catch` blocks: `$_` and `$_.ScriptStackTrace`.

### Variable Handling

- **Never** use `$Result` or `$ResultString` as variable names. They are reserved by PolicyExecutor.
- In standalone scripts, hardcode inputs or use `param()` blocks. Do not rely on InArgs.
- In embedded scripts, parameters come from InArgs mapping.

### Output

- Use `Write-Host` as the primary output mechanism.
- Prefix messages with `[INFO]`, `[SUCCESS]`, `[WARN]`, or `[ERROR]` for consistent parsing.
- Always include `$env:COMPUTERNAME` in key messages to identify the device in multi-device reports.
- Keep output under 10,000 characters for Dashboard visibility.

### File and Path Operations

- Always use absolute paths (e.g., `C:\DatoLogs\ScriptName`).
- Check existence with `Test-Path` before reading or writing.
- Build paths with `Join-Path` instead of string concatenation.
- Create directories with `New-Item -Path $dir -ItemType Directory -Force | Out-Null`.

### Process Execution

- Use `Start-Process -Wait -PassThru` to run external executables.
- Capture the exit code: `$proc.ExitCode`.
- Use `-NoNewWindow` to keep output in the current console context.

### Registry

- Use PowerShell provider paths: `HKLM:\SOFTWARE\...`, `HKCU:\SOFTWARE\...`.
- Create keys with `New-Item -Path $path -Force`.
- Set values with `New-ItemProperty -Path $path -Name $name -Value $value -PropertyType $type -Force`.
- Use `-Force` to create or overwrite without errors.

### Service Operations

- Query services with `Get-Service -Name $name -ErrorAction SilentlyContinue`.
- Check for `$null` before operating on the result.
- Use `Set-Service`, `Start-Service`, `Stop-Service` for management.

### SYSTEM Context Gotchas

Scripts deployed via N-Sight run as `NT AUTHORITY\SYSTEM`. Be aware of:

- **No user profile.** `$env:USERPROFILE` points to `C:\Windows\system32\config\systemprofile`, not a real user directory.
- **No HKCU for users.** `HKCU:\` is the SYSTEM account's registry hive. To modify a user's registry, load their `ntuser.dat` via `reg load`.
- **No mapped drives.** Network drives mapped by users are not visible in SYSTEM context. Use UNC paths.
- **winget path resolution.** `winget` is a per-user app. In SYSTEM context, resolve the path explicitly:
  ```powershell
  $wingetPath = Get-ChildItem "C:\Program Files\WindowsApps\Microsoft.DesktopAppInstaller_*_x64__8wekyb3d8bbwe\winget.exe" -ErrorAction SilentlyContinue |
      Sort-Object FullName -Descending |
      Select-Object -First 1 -ExpandProperty FullName
  ```

---

## Logging Patterns

### Simple: Write-Host with Prefix Labels

For embedded scripts and lightweight standalone scripts.

```powershell
Write-Host "[INFO] Starting operation on $env:COMPUTERNAME"
Write-Host "[SUCCESS] Operation completed"
Write-Host "[ERROR] Something failed: $_"
```

### Structured: Write-Log with Dual Output

For standalone scripts that need persistent log files.

```powershell
function Write-Log {
    param(
        [Parameter(Mandatory)]
        [string]$Message,

        [ValidateSet('INFO', 'SUCCESS', 'WARN', 'ERROR')]
        [string]$Level = 'INFO'
    )
    $timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    $entry = "[$timestamp] [$Level] $Message"
    Add-Content -Path $LogFile -Value $entry
    Write-Host $entry
}
```

This writes every log entry to both a file (for post-mortem analysis) and `Write-Host` (for Dashboard display).

---

## Password and Credential Handling

### .amp Files Are Not Safe for Credentials

Embedded scripts in `.amp` files store values as `System.String` in plain text within the XML. Anyone with access to the `.amp` file can decode and read credentials.

### Blocked Characters

The following characters cause issues in N-Sight parameter injection and should be avoided in passwords and string parameters:

| Character | Issue |
|-----------|-------|
| `"` | Breaks XML attribute quoting |
| `'` | Breaks PowerShell string literals |
| `/` | Misinterpreted in path contexts |
| `\` | Escape character conflicts |
| `$` | PowerShell variable expansion |
| `` ` `` | PowerShell escape character |

### Recommendations

- Never store production credentials in `.amp` files.
- For production environments, retrieve secrets at runtime from a vault or secure API.
- If credentials must be passed, use the N-Sight Custom Properties feature with appropriate access controls, and accept the risk that values are not encrypted at rest.
