---
allowed-tools: Read, Write, Bash, Glob, Grep
description: Create a new N-Sight RMM .amp automation policy file or standalone PowerShell script
argument-hint: "[deployment|checker|utility|clienttool|script] description of what to create"
disable-model-invocation: false
---

Create a new N-Sight RMM automation artifact based on the user's description. This can be either an `.amp` Automation Manager policy file or a standalone PowerShell script for Script Manager.

## Determine What to Create

Parse the user's description to determine:

1. **Type**: .amp policy or standalone PowerShell script
   - If the user says "script", "ps1", "powershell script", "Script Manager" → create a standalone .ps1
   - Otherwise → create an .amp policy file

2. **Category** (for .amp files):
   - **deployment** — installing/deploying software
   - **checker** — verifying software is installed, state tracking
   - **utility** — system admin tasks (registry, services, cleanup, diagnostics)
   - **clienttool** — client-facing tools (messages, prompts, file operations)

## For .amp Policy Files

### Step 1: Read Example Files

Read matching example .amp files from `${CLAUDE_PLUGIN_ROOT}/Examples/` for the determined category:

- **Utility**: Read `Examples/Utilities/DisablePin.amp` (simplest reference)
- **Checker**: Read `Examples/Checkers/CoveChecker.amp` (canonical checker pattern)
- **Deployment**: Read `Examples/Deployments/HuntressDeployment2025.amp` (smallest deployment)
- **ClientTool**: Read `Examples/ClientTools/LanMessage.amp` (parameter mapping example)

Also read the skill reference for the schema: `${CLAUDE_PLUGIN_ROOT}/skills/nsight-scripter/SKILL.md`

### Step 2: Generate GUIDs

Generate all needed GUIDs upfront:

```bash
python3 -c "
import uuid
print('Policy ID:', str(uuid.uuid4()))
print('Object ID: {' + str(uuid.uuid4()) + '}')
# Add more for each activity Moniker needed
for i in range(10):
    print(f'Moniker {i+1}:', str(uuid.uuid4()))
"
```

### Step 3: Encode Description

```bash
echo -n "Your policy description here" | base64
```

### Step 4: Write PowerShell Script (if needed)

Write the PowerShell script as a plain .ps1 file first for readability. Follow these rules for embedded scripts:
- Set `$ErrorActionPreference = 'Stop'`
- NEVER use `$Result` or `$ResultString` — reserved by PolicyExecutor
- Use `Write-Host` for output (goes to OutPut_64 variable)
- Avoid `Exit` — let PolicyExecutor manage results
- Parameters come from InArgs mapping as `$VariableName`

### Step 5: Base64-Encode the Script

```bash
cat script.ps1 | iconv -f utf-8 -t utf-16le | base64 -w0
```

Or inline:
```bash
echo -n 'Write-Host "Hello"' | iconv -f utf-8 -t utf-16le | base64 -w0
```

### Step 6: Assemble the .amp XML

Build the complete XML following the schema. Key points:
- Policy ID: lowercase GUID without braces
- Object ID: GUID with braces `{...}`
- Object Type: always `{B6FA6D8B-EEAA-47A6-8463-7F9A4F5BBB6E}`
- Object Data: HTML-encode the parameters XML (`<` → `&lt;`, `>` → `&gt;`, `"` → `&quot;`)
- LinkManager: static boilerplate (copy from examples)
- Diagnostics OriginalVersion: `2.98.2.2`
- Activity namespaces: copy from examples, add `sco` for SwitchObject, `acb` for InputPrompt
- Variables: declare ALL variables used by activities in the nearest containing Variables block
- Variable naming: `{ActivityType}_{OutputName}`, then `_{N}` for subsequent instances

### Step 7: Write and Validate

Write the .amp file, then validate:
```bash
xmllint --noout generated-policy.amp 2>&1 || echo "XML validation failed"
```

Decode and verify embedded scripts:
```bash
# Extract and decode the script attribute to verify
echo "{base64-from-script-attribute}" | base64 -d | iconv -f utf-16le -t utf-8
```

## For Standalone PowerShell Scripts

### Step 1: Determine Script Type

- **Script Check** — monitoring (runs on schedule, Exit 0 = pass, non-zero = fail)
- **Automated Task** — maintenance (runs on demand or schedule)

### Step 2: Generate the Script

Use this template:

```powershell
<#
.SYNOPSIS
    Brief description
.DESCRIPTION
    Detailed description. Designed for N-Sight RMM deployment.
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

$LogPath = "C:\DatoLogs\{ScriptName}"
$LogFile = "$LogPath\$env:COMPUTERNAME.log"

function Write-Log {
    param ([string]$Message, [string]$Level = "INFO")
    $timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    $logEntry = "$timestamp [$Level] $Message"
    Add-Content -Path $LogFile -Value $logEntry -ErrorAction SilentlyContinue
    Write-Host $logEntry
}

try {
    if (-not (Test-Path $LogPath)) {
        New-Item -ItemType Directory -Path $LogPath -Force | Out-Null
    }
    Write-Log "Script started on $env:COMPUTERNAME"

    # --- Main logic here ---

    Write-Log "Operation completed successfully" -Level "SUCCESS"
    Exit 0
}
catch {
    $errorMsg = $_.Exception.Message
    Write-Log "An error occurred: $errorMsg" -Level "ERROR"
    Exit 1001
}
```

**Critical rules:**
- Max 65,535 characters
- Exit 0 = pass (green on Dashboard), Exit 1001+ = fail (red)
- NEVER use exit codes 1-999 (reserved by N-Sight)
- Use Write-Host for Dashboard output (max 10,000 chars)
- No `param()` block — N-Sight Script Manager doesn't support it
- Use absolute paths — runs as SYSTEM from unpredictable working directory
- No user interaction (no Read-Host, no GUI)

### Step 3: Write the .ps1 File

Write the script to the user's specified location or current directory.
