---
allowed-tools: Read, Bash, Glob, Grep
description: Decode and explain an N-Sight RMM .amp file or PowerShell script
argument-hint: "path/to/file.amp or path/to/file.ps1"
disable-model-invocation: false
---

Decode and explain an N-Sight RMM `.amp` policy file or PowerShell script in human-readable form.

## Determine File Type

Check the file extension:
- `.amp` → follow the .amp reading workflow
- `.ps1` → follow the PowerShell reading workflow

## For .amp Files

### Step 1: Read the File

Read the .amp file specified by the user. If no path given, use Glob to find .amp files and ask which one.

### Step 2: Decode All Components

**Policy metadata:**
- Name: from the `Name` attribute
- Description: decode Base64 → UTF-8
  ```bash
  echo "{Description-value}" | base64 -d
  ```
- Version, ExecutionType, MinimumPSVersionRequired

**Parameters and GlobalVariables:**
HTML-decode the Object Data attribute, then extract:
- Parameters (user inputs at deployment time)
- GlobalVariables (internal constants)

For each parameter/variable, show: name, label, type (string/number), default value.

**Workflow activities:**
Walk the Activity > PolicySequence > PolicySequence.Activities in order. For each activity:
- Activity type (RunPowerShellScript, Log, IfElse, IsAppInstalled, etc.)
- DisplayName
- Key configuration (what it does)
- For IfElse/SwitchObject: explain the branching logic and conditions
- For nested activities in branches: indent and show hierarchy

**Embedded PowerShell scripts:**
For each RunPowerShellScript activity, decode the script:
```bash
echo "{script-attribute-value}" | base64 -d | iconv -f utf-16le -t utf-8
```

Show the decoded script with syntax highlighting context.

If InArgs are present, explain the parameter mapping (policy variable → PowerShell variable).

### Step 3: Present Summary

Format the output as:

```
## Policy: {Name}
**Description:** {decoded description}
**Version:** {version}
**Execution:** {ExecutionType}
**Min PS Version:** {version}

### Parameters (user-configurable)
| Name | Label | Type | Default |
|------|-------|------|---------|
| ... | ... | ... | ... |

### Global Variables (internal)
| Name | Label | Type | Default |
|------|-------|------|---------|
| ... | ... | ... | ... |

### Workflow
1. {Activity type} — {DisplayName}: {what it does}
2. {Activity type} — {DisplayName}: {what it does}
   - If {condition}:
     - Then: {activities}
     - Else: {activities}
3. ...

### Embedded Scripts
#### Script 1: {DisplayName}
{decoded PowerShell code}

**Parameter mappings:**
- $PSVar ← Policy Variable "Label"
```

## For PowerShell Scripts

### Step 1: Read the Script

Read the .ps1 file.

### Step 2: Analyze and Explain

Present:
1. **Purpose** — from comment-based help (.SYNOPSIS, .DESCRIPTION)
2. **Metadata** — author, version, target (from .NOTES)
3. **Configuration** — key variables and their defaults
4. **Functions** — list helper functions and what they do
5. **Main Logic** — step-by-step explanation of what the script does
6. **Error Handling** — how errors are caught and reported
7. **Exit Codes** — what exit codes are used and what they mean
8. **N-Sight Compatibility** — note any issues (reserved vars, exit code range, output method)
