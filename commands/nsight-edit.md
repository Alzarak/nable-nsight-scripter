---
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
description: Edit an existing N-Sight RMM .amp automation policy file or PowerShell script
argument-hint: "path/to/file.amp or path/to/file.ps1"
disable-model-invocation: false
---

Edit an existing N-Sight RMM `.amp` policy file or PowerShell script.

## Determine File Type

Check the file extension:
- `.amp` → follow the .amp editing workflow
- `.ps1` → follow the PowerShell editing workflow

## For .amp Files

### Step 1: Read and Decode

Read the .amp file specified by the user. If no path given, use Glob to find .amp files:
```bash
# Find .amp files in current directory
```

### Step 2: Decode All Components

**Decode the description:**
```bash
echo "{Description-attribute-value}" | base64 -d
```

**Parse Object Data:** HTML-decode the Data attribute to extract Parameters and GlobalVariables. The Data is HTML-encoded XML:
- `&lt;` → `<`
- `&gt;` → `>`
- `&quot;` → `"`
- `&amp;` → `&`

**Decode embedded PowerShell scripts:**
```bash
echo "{script-attribute-value}" | base64 -d | iconv -f utf-16le -t utf-8
```

### Step 3: Present Structure to User

Show the user:
1. **Policy name** and decoded description
2. **Parameters** (user-configurable inputs)
3. **Global Variables** (internal constants)
4. **Workflow** — list all activities in execution order with their types and key attributes
5. **Embedded scripts** — show decoded PowerShell code
6. **Conditional logic** — explain any IfElse/SwitchObject branches

### Step 4: Ask What Changes Are Needed

Wait for the user to describe what they want to change.

### Step 5: Apply Changes

Common edits:

**Modify embedded PowerShell:**
1. Edit the script content
2. Re-encode: `echo -n 'new script content' | iconv -f utf-8 -t utf-16le | base64 -w0`
3. Replace the `script` attribute value

**Modify parameters:**
1. Update the Object Data (HTML-encoded XML)
2. Update matching Variable declarations with new Default values

**Change description:**
1. Encode new description: `echo -n "new description" | base64`
2. Replace the Description attribute

**Add/remove activities:**
1. Follow the schema for the activity type (read reference docs)
2. Declare all new variables in the appropriate Variables block
3. Use unique GUIDs for new Monikers

### Step 6: Write and Validate

Write the modified .amp file. Validate:
```bash
xmllint --noout modified-policy.amp 2>&1 || echo "XML validation failed"
```

Verify any re-encoded scripts:
```bash
echo "{new-base64}" | base64 -d | iconv -f utf-16le -t utf-8
```

## For PowerShell Scripts

### Step 1: Read the Script

Read the .ps1 file and present its structure:
- Comment-based help
- Configuration/variables
- Functions
- Main logic
- Error handling
- Exit codes

### Step 2: Apply Changes

Follow N-Sight RMM conventions when editing:
- Maintain `$ErrorActionPreference = 'Stop'`
- Keep exit codes > 1000 for failures (never 1-999)
- Keep `Write-Host` for Dashboard output
- Preserve the Write-Log pattern if present
- Keep absolute paths
- Never introduce `$Result` or `$ResultString` if this will be embedded in an .amp later

### Step 3: Write the Modified Script
