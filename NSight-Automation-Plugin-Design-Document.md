# N-Sight RMM Automation Plugin — Design Document

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a comprehensive Claude Code plugin that can create, edit, read, and manage N-Able N-Sight RMM `.amp` (Automation Manager Package) files — the XML-based policy format used by N-Sight's Automation Manager.

**Architecture:** The plugin provides a skill (`nsight-scripter`) with deep knowledge of the `.amp` XML schema, N-Sight RMM scripting conventions, and PowerShell best practices specific to N-Sight. It uses the 29 example `.amp` files in `/home/datotech/projects/nable-nsight-scripter/Examples/` as its learned reference corpus. The skill instructs Claude how to generate valid `.amp` XML, embed Base64-encoded PowerShell scripts, wire up parameters/variables, and structure conditional workflows — all without needing the Automation Manager GUI.

**Tech Stack:** Claude Code Plugin (YAML frontmatter + Markdown skill), XML generation, Base64 encoding (UTF-16LE), PowerShell scripting

---

## Table of Contents

1. [N-Sight RMM Platform Context](#1-n-sight-rmm-platform-context)
2. [The .amp File Format — Complete Specification](#2-the-amp-file-format--complete-specification)
3. [Activity Types Reference](#3-activity-types-reference)
4. [PowerShell Embedding Specification](#4-powershell-embedding-specification)
5. [N-Sight RMM Scripting Conventions](#5-n-sight-rmm-scripting-conventions)
6. [Policy Categories & Templates](#6-policy-categories--templates)
7. [Plugin Architecture](#7-plugin-architecture)
8. [Plugin Commands & Skills](#8-plugin-commands--skills)
9. [Example Corpus Reference](#9-example-corpus-reference)
10. [Verification Plan](#10-verification-plan)

---

## 1. N-Sight RMM Platform Context

### What is N-Sight RMM?

N-Able N-Sight RMM (formerly SolarWinds RMM) is a Remote Monitoring and Management platform used by MSPs (Managed Service Providers) to manage client devices. It provides two scripting paths:

1. **Script Manager** — Upload standalone scripts (PowerShell, VBS, Batch, etc.) directly to the Dashboard. Deploy as Script Checks (24x7/Daily monitoring) or Automated Tasks (maintenance). Max 65,535 characters, max 10,000 characters output.

2. **Automation Manager** — Visual drag-and-drop policy builder that generates `.amp` files. Policies are PowerShell-based with 600+ pre-built objects. Upload `.amp` to Dashboard for deployment as Script Checks or Automated Tasks.

### Key Platform Constraints

| Constraint | Value |
|---|---|
| Script size limit (Script Manager) | 65,535 characters |
| Script output limit | 10,000 characters |
| Script size limit (Automation Manager) | 10 MB per embedded PowerShell |
| Reserved exit codes | 1–999 (system scripts) |
| Custom exit codes | Must be > 1000 for text output to display |
| Exit 0 | Pass (green on Dashboard) |
| Exit non-zero | Fail (red on Dashboard) |
| PowerShell version required | 5.x for AMP engine |
| Minimum PS version in policy | Configurable (0.0.0 to 5.0+) |
| Reserved variables | `$Result`, `$ResultString` — NEVER use in embedded scripts |
| Password characters blocked | `" ' / \ $ \`` |
| Execution context | SYSTEM (no user/desktop interaction) |
| Agent scripts path | `C:\Program Files (x86)\Advanced Monitoring Agent\scripts` |

### Exit Code Behavior — Critical Distinction

**Standalone scripts (Script Manager):**
- `Write-Host "message"` + `Exit 0` → Pass with message on Dashboard
- `Write-Host "error"` + `Exit 1001` → Fail with error on Dashboard

**Automation Manager (.amp) embedded PowerShell:**
- Exit codes from `Run PowerShell Script` are **consumed within the module only** — they do NOT propagate to the Dashboard
- To surface failures: wire up `If/Else` Control Flow → `Fail Policy` module
- The `Result` output variable (type `x:Double`) captures the exit code internally

---

## 2. The .amp File Format — Complete Specification

### Overview

An `.amp` file is a **UTF-8 XML 1.0 document** with CRLF line endings. Despite the non-standard extension, it is plain XML — not a ZIP archive, not binary. The schema is based on **Microsoft Windows Workflow Foundation (WF) XAML** with a custom `PolicyExecutor` namespace.

### Root Structure

Every `.amp` file has exactly this structure:

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<Policy
  ID="{GUID}"
  Name="{PolicyName}"
  Description="{Base64EncodedDescription}"
  Version="{Major.Minor.Patch.Build}"
  [MinRequiredVersion="{Version}"]
  RemoteCategory="{0|1|...}"
  ExecutionType="{Local|CurrentLoggedOnUser}"
  MinimumPSVersionRequired="{Version}">

  <Object ... />          <!-- Parameters & Global Variables -->
  <LinkManager ... />     <!-- Link metadata (usually empty) -->
  <Diagnostics ... />     <!-- Version tracking -->
  <Activity ... />        <!-- Main XAML workflow -->

</Policy>
```

### 2.1 Policy Element — Root Attributes

| Attribute | Type | Required | Description |
|---|---|---|---|
| `ID` | GUID | Yes | Unique identifier. Generate with `[guid]::NewGuid()` or `uuidgen`. Format: `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` (lowercase, no braces) |
| `Name` | String | Yes | Human-readable policy name. Use underscores for spaces (e.g., `Disable_Pin`) |
| `Description` | Base64 | Yes | Policy description encoded as **Base64 of UTF-8 text**. Example: `"Disables Windows Hello Pin Requirements"` → `RGlzYWJsZXMgV2luZG93cyBIZWxsbyBQaW4gUmVxdWlyZW1lbnRz` |
| `Version` | String | Yes | Semantic version `Major.Minor.Patch.Build`. Observed range: `2.10.0.19` to `2.50.0.0`. Use `2.10.0.19` as default for simple policies |
| `MinRequiredVersion` | String | No | Minimum Automation Manager version needed. Only include when using newer features (e.g., InputPrompt requires `2.50.0.0`) |
| `RemoteCategory` | Integer | Yes | Category identifier. Always `0` in all examples |
| `ExecutionType` | String | Yes | `Local` (runs as SYSTEM) or `CurrentLoggedOnUser` |
| `MinimumPSVersionRequired` | String | Yes | Minimum PowerShell version. `0.0.0` = any version, `3.0` = PS 3.0+ |

### 2.2 Object Element — Parameters & Global Variables

```xml
<Object
  ID="{GUID-in-braces}"
  Type="{B6FA6D8B-EEAA-47A6-8463-7F9A4F5BBB6E}"
  Data="{HTML-encoded XML}" />
```

- `ID`: GUID **with braces** — `{xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx}`
- `Type`: Always `{B6FA6D8B-EEAA-47A6-8463-7F9A4F5BBB6E}` — this is the metadata container type GUID
- `Data`: HTML-encoded XML string containing Parameters and GlobalVariables

**Data format (decoded):**

```xml
<xml>
  <Parameters>
    <Parameter ParameterName="VarName" Label="Display Label" ParameterType="string|number" Value="default_value" />
    <!-- More parameters... -->
  </Parameters>
  <GlobalVariables>
    <Parameter ParameterName="VarName" Label="Label" ParameterType="string|number" Value="default_value" />
  </GlobalVariables>
</xml>
```

**When no parameters exist:** `Data="&lt;xml /&gt;"`

**HTML encoding rules for Data attribute:**
- `<` → `&lt;`
- `>` → `&gt;`
- `"` → `&quot;`
- `&` → `&amp;`

**Parameter vs GlobalVariable:**
- **Parameters** = user-configurable inputs shown at deployment time (the operator fills these in when deploying)
- **GlobalVariables** = internal constants used within the policy workflow (not shown to operator at deployment)

**ParameterType values:**
- `string` — text value
- `number` — numeric value (stored as `x:Double` in Variables)

**Example (RestartService.amp):**
```xml
Data="&lt;xml&gt;&lt;Parameters&gt;&lt;Parameter ParameterName=&quot;Service_Name&quot; Label=&quot;Service Name&quot; ParameterType=&quot;string&quot; Value=&quot;&quot; /&gt;&lt;/Parameters&gt;&lt;/xml&gt;"
```

**Example (CoveChecker.amp — both Parameters and GlobalVariables):**
```xml
Data="&lt;xml&gt;&lt;Parameters&gt;&lt;Parameter ParameterName=&quot;Task_Check&quot; Label=&quot;Uninstall = 0 Install = 1&quot; ParameterType=&quot;number&quot; Value=&quot;1&quot; /&gt;&lt;/Parameters&gt;&lt;GlobalVariables&gt;&lt;Parameter ParameterName=&quot;Program&quot; Label=&quot;Program&quot; ParameterType=&quot;string&quot; Value=&quot;Backup Manager&quot; /&gt;&lt;/GlobalVariables&gt;&lt;/xml&gt;"
```

### 2.3 LinkManager Element

Always present, always this exact structure:

```xml
<LinkManager xmlns:i="http://www.w3.org/2001/XMLSchema-instance"
             xmlns="http://schemas.datacontract.org/2004/07/PolicyExecutor">
  <hashset xmlns:d2p1="http://schemas.datacontract.org/2004/07/System" />
</LinkManager>
```

This is a static boilerplate element. Never modify it.

### 2.4 Diagnostics Element

```xml
<Diagnostics OriginalVersion="{Version}" />
```

Records the PolicyExecutionEngine version that originally created this policy. Observed values: `2.96.1.1`, `2.98.1.2`, `2.98.2.2`. Use `2.98.2.2` as default for new policies.

### 2.5 Activity Element — XAML Workflow

This is the heart of the `.amp` file. It uses Microsoft WF (Windows Workflow Foundation) XAML format.

**Required namespace declarations:**

```xml
<Activity
  mc:Ignorable="sads sap"
  x:Class="Policy Builder"
  xmlns="http://schemas.microsoft.com/netfx/2009/xaml/activities"
  xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
  xmlns:mva="clr-namespace:Microsoft.VisualBasic.Activities;assembly=System.Activities"
  xmlns:p="clr-namespace:PolicyExecutor;assembly=PolicyExecutionEngine"
  xmlns:sads="http://schemas.microsoft.com/netfx/2010/xaml/activities/debugger"
  xmlns:sap="http://schemas.microsoft.com/netfx/2009/xaml/activities/presentation"
  xmlns:scg="clr-namespace:System.Collections.Generic;assembly=mscorlib"
  xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
```

**Additional namespaces (when needed):**
- `xmlns:sco="clr-namespace:System.Collections.ObjectModel;assembly=mscorlib"` — needed for SwitchObject/CaseObject
- `xmlns:acb="clr-namespace:AutomationManager.Common.Branding;assembly=AutomationManager.Common"` — needed for InputPrompt with branding

**Required boilerplate inside Activity:**

```xml
<x:Members>
  <x:Property Name="PolicyGUID" Type="InArgument(x:String)" />
</x:Members>
<sap:VirtualizedContainerService.HintSize>{Width},{Height}</sap:VirtualizedContainerService.HintSize>
<mva:VisualBasic.Settings>Assembly references and imported namespaces serialized as XML namespaces</mva:VisualBasic.Settings>
```

**HintSize** is a visual layout hint (pixels). Not functionally significant but must be present. Typical values: `490,827` for simple, `341,667` for checkers.

### 2.6 PolicySequence — The Workflow Container

```xml
<p:PolicySequence
  DisplayName="Policy Builder"
  sap:VirtualizedContainerService.HintSize="{W},{H}"
  [MinRequiredVersion="{Version}"]
  mva:VisualBasic.Settings="Assembly references and imported namespaces serialized as XML namespaces">

  <p:PolicySequence.Activities>
    <!-- Activity objects execute in sequence, top to bottom -->
  </p:PolicySequence.Activities>

  <p:PolicySequence.Variables>
    <!-- All variables used by activities must be declared here -->
  </p:PolicySequence.Variables>

</p:PolicySequence>
```

### 2.7 Variable Declarations

Every variable referenced by any activity **must** be declared in the nearest containing `*.Variables` block. Variables follow a naming convention:

```xml
<!-- String variable -->
<Variable x:TypeArguments="x:String" Name="RunPowerShellScript_OutPut_64" />

<!-- String with default value -->
<Variable x:TypeArguments="x:String" Default="some value" Name="MyVar" />

<!-- Double (numeric) variable -->
<Variable x:TypeArguments="x:Double" Name="RunPowerShellScript_Result" />

<!-- Double with default value -->
<Variable x:TypeArguments="x:Double" Default="1" Name="Task_Check" />

<!-- IEnumerable (collection) variable -->
<Variable x:TypeArguments="scg:IEnumerable(x:Object)" Name="RunPowerShellScript_Results_x64" />
```

**Variable naming convention for activity outputs:**
- `{ActivityType}_{OutputName}` for the first instance
- `{ActivityType}_{OutputName}_{N}` for subsequent instances (e.g., `Log_Result_1`, `Log_Result_2`)

**Parameter/GlobalVariable variables:**
- Parameters and GlobalVariables defined in the `<Object>` Data are also declared as Variables with their default values matching the Data values

---

## 3. Activity Types Reference

All activity types live in the `p:` namespace (`PolicyExecutor`). Each has common attributes:

### 3.0 Common Activity Attributes

Every activity element shares these attributes:

| Attribute | Description |
|---|---|
| `AssemblyName` | Always `PolicyExecutionEngine, Version={X}, Culture=neutral, PublicKeyToken=null` |
| `DisplayName` | Human-readable name shown in Automation Manager GUI |
| `sap:VirtualizedContainerService.HintSize` | Layout hint `{W},{H}` |
| `Moniker` | Unique GUID (no braces) identifying this specific activity instance |
| `Result` | Variable reference for numeric result: `[VarName_Result]` |
| `ResultString` | Variable reference for string result: `[VarName_ResultString]` |
| `RunAsCurrentLoggedOnUser` | `True` or `False` |
| `ScriptExecutionMethod` | `ExecuteDebug` or `None` |
| `TypeName` | The activity type name (matches element name without `p:` prefix) |
| `m_bTextLinkChange` | Always `False` |

**Variable references** use bracket syntax: `[VariableName]`

### 3.1 RunPowerShellScript

The most important activity type. Executes embedded PowerShell code.

```xml
<p:RunPowerShellScript
  genArgEvent="{x:Null|GUID}"
  AssemblyName="PolicyExecutionEngine, Version=2.96.1.1, Culture=neutral, PublicKeyToken=null"
  DisplayName="Run PowerShell Script"
  sap:VirtualizedContainerService.HintSize="454,348"
  Moniker="{GUID}"
  OutPut_64="[RunPowerShellScript_OutPut_64]"
  Result="[RunPowerShellScript_Result]"
  ResultString="[RunPowerShellScript_ResultString]"
  Results_x64="[RunPowerShellScript_Results_x64]"
  RunAsCurrentLoggedOnUser="False"
  ScriptExecutionMethod="ExecuteDebug"
  TypeName="RunPowerShellScript"
  m_bTextLinkChange="False"
  script="{BASE64_ENCODED_SCRIPT}">

  <p:RunPowerShellScript.InArgs>
    <!-- Input argument mappings (or empty dictionary) -->
    <scg:Dictionary x:TypeArguments="x:String, p:InArg" />
  </p:RunPowerShellScript.InArgs>

  <p:RunPowerShellScript.OutArgs>
    <!-- Output argument mappings (or empty dictionary) -->
    <scg:Dictionary x:TypeArguments="x:String, p:OutArg" />
  </p:RunPowerShellScript.OutArgs>

</p:RunPowerShellScript>
```

**`genArgEvent` attribute:** Set to `{x:Null}` when no InArgs are mapped. Set to a GUID when InArgs are present (this GUID links the argument event to the parameter mapping).

**Required variables (declare in containing PolicySequence.Variables):**
- `RunPowerShellScript_OutPut_64` — `x:String` — stdout output
- `RunPowerShellScript_Result` — `x:Double` — numeric exit code
- `RunPowerShellScript_ResultString` — `x:String` — result message
- `RunPowerShellScript_Results_x64` — `scg:IEnumerable(x:Object)` — results collection

**For subsequent RunPowerShellScript instances in the same policy:** Append `_1`, `_2`, etc. to variable names (e.g., `RunPowerShellScript_OutPut_64_1`, `RunPowerShellScript_Result_1`).

**InArgs with parameter mapping (when passing input parameters to script):**

```xml
<p:RunPowerShellScript.InArgs>
  <p:InArg
    Item="{x:Null}"
    ItemProp="{x:Null}"
    x:Key="PSVariableName"
    ArgType="string"
    DisplayArg="Input Parameters.Label"
    DisplayName="PSVariableName"
    Name="PSVariableName"
    isRequired="False">
    <p:InArg.Arg>
      <InArgument x:TypeArguments="x:Object">[VariableName]</InArgument>
    </p:InArg.Arg>
  </p:InArg>
</p:RunPowerShellScript.InArgs>
```

This maps the policy variable `[VariableName]` to the PowerShell variable `$PSVariableName` inside the script.

### 3.2 Log

Outputs a message to the policy execution log.

```xml
<p:Log
  Message_Item="{x:Null}"
  Message_ItemProp="{x:Null}"
  AssemblyName="PolicyExecutionEngine, Version=2.96.1.1, Culture=neutral, PublicKeyToken=null"
  DisplayName="Log"
  sap:VirtualizedContainerService.HintSize="454,88"
  LogMessage="[Log_LogMessage]"
  Message="[RunPowerShellScript_OutPut_64]"
  Message_DisplayArg="Run PowerShell Script.OutPut_64"
  Moniker="{GUID}"
  Result="[Log_Result]"
  ResultString="[Log_ResultString]"
  RunAsCurrentLoggedOnUser="False"
  ScriptExecutionMethod="ExecuteDebug"
  TypeName="Log"
  m_bTextLinkChange="False" />
```

**Key attributes:**
- `Message` — the variable reference containing the message: `[SomeVar]` or a literal string
- `Message_DisplayArg` — display label for the message source
- `LogMessage` — variable to store the logged message

### 3.3 IsAppInstalled

Checks if an application is installed on the device.

```xml
<p:IsAppInstalled
  ApplicationName_Item="{x:Null}"
  ApplicationName_ItemProp="{x:Null}"
  ApplicationName="[Program]"
  ApplicationName_DisplayArg="Global Variables.Program"
  AssemblyName="..."
  Conditional="[IsAppInstalled_Conditional]"
  DisplayName="Is Application Installed"
  ...
  TypeName="IsAppInstalled" />
```

**Key output:** `Conditional` — `x:String` variable set to `"True"` or `"False"`

### 3.4 GetEnvironmentVariable

```xml
<p:GetEnvironmentVariable
  Type="Machine"
  TypeName="GetEnvironmentVariable"
  Variable="[Program]"
  Variable_DisplayArg="Global Variables.Program"
  Value="[GetEnvironmentVariable_Value]"
  ... />
```

### 3.5 SetEnvironmentVariable

```xml
<p:SetEnvironmentVariable
  Type="Machine"
  TypeName="SetEnvironmentVariable"
  Variable="[Program]"
  Variable_DisplayArg="Global Variables.Program"
  Value="1"
  Value_DisplayArg="1"
  ... />
```

### 3.6 IfElse — Conditional Branching

```xml
<p:IfElse
  CaseSensitive="False"
  Condition="equals"
  Condition_DisplayArg="equals"
  DisplayName="Descriptive Name"
  TypeName="IfElse"
  Value_DisplayArg="True"
  Value_Type="x:String"
  Variable="[SomeVar]"
  Variable_DisplayArg="Source.Property"
  Variable_Type="x:String"
  ...>

  <p:IfElse.IfOption>
    <p:SequenceActivity DisplayName="Then" Name="SequenceActivity">
      <p:SequenceActivity.Activities>
        <!-- Activities when condition is TRUE -->
      </p:SequenceActivity.Activities>
      <p:SequenceActivity.Variables>
        <!-- Variables scoped to this branch -->
      </p:SequenceActivity.Variables>
    </p:SequenceActivity>
  </p:IfElse.IfOption>

  <p:IfElse.ElseOption>
    <p:SequenceActivity DisplayName="Else" Name="SequenceActivity">
      <p:SequenceActivity.Activities>
        <!-- Activities when condition is FALSE -->
      </p:SequenceActivity.Activities>
      <p:SequenceActivity.Variables>
        <!-- Variables scoped to this branch -->
      </p:SequenceActivity.Variables>
    </p:SequenceActivity>
  </p:IfElse.ElseOption>

  <p:IfElse.Value>
    <InArgument x:TypeArguments="x:Object">
      <p:ObjectLiteral Value="True" />
    </InArgument>
  </p:IfElse.Value>

</p:IfElse>
```

**Condition operators:** `equals`, `does not equal`, `contains`, `does not contain`, `greater than`, `less than`

### 3.7 IfObject — Simple Conditional

Similar to IfElse but simpler — only has an `IfOption` (no else).

```xml
<p:IfObject
  CaseSensitive="False"
  Condition="equals"
  DisplayName="If"
  TypeName="IfObject"
  Value_DisplayArg="True"
  Value_Type="x:String"
  Variable="[SomeVar]"
  Variable_DisplayArg="Source.Property"
  Variable_Type="x:String"
  VerboseOutput="False"
  ...>

  <p:IfObject.IfOption>
    <p:SequenceActivity ...>
      <!-- Activities when condition matches -->
    </p:SequenceActivity>
  </p:IfObject.IfOption>

  <p:IfObject.Value>
    <InArgument x:TypeArguments="x:Object">
      <p:ObjectLiteral Value="True" />
    </InArgument>
  </p:IfObject.Value>

</p:IfObject>
```

### 3.8 SwitchObject / CaseObject — Multi-Way Branching

```xml
<p:SwitchObject
  AllowDefault="False"
  DisplayName="Label"
  TypeName="SwitchObject"
  Variable="[Task_Check]"
  Variable_DisplayArg="Input Parameters.Label"
  Variable_Type="x:Double"
  ...>

  <p:SwitchObject.CaseSequence>
    <p:CaseSequenceActivity DisplayName="" Name="CaseSequenceActivity">
      <p:CaseSequenceActivity.Activities>

        <p:CaseObject DisplayName="Case Label" TypeName="CaseObject" Value_DisplayArg="0" RunCase="False" ...>
          <p:CaseObject.ThenOption>
            <p:SequenceActivity ...>
              <!-- Activities for this case -->
            </p:SequenceActivity>
          </p:CaseObject.ThenOption>
          <p:CaseObject.Value>
            <InArgument x:TypeArguments="x:Object">
              <p:ObjectLiteral Value="0" />
            </InArgument>
          </p:CaseObject.Value>
        </p:CaseObject>

        <!-- More CaseObject elements... -->

      </p:CaseSequenceActivity.Activities>
      <p:CaseSequenceActivity.Variables>
        <!-- Variables for all cases -->
      </p:CaseSequenceActivity.Variables>
    </p:CaseSequenceActivity>
  </p:SwitchObject.CaseSequence>

  <p:SwitchObject.DefaultOption>
    <p:SequenceActivity DisplayName="Default" Name="SequenceActivity">
      <p:SequenceActivity.Activities>
        <sco:Collection x:TypeArguments="Activity" />
      </p:SequenceActivity.Activities>
      <p:SequenceActivity.Variables>
        <sco:Collection x:TypeArguments="Variable" />
      </p:SequenceActivity.Variables>
    </p:SequenceActivity>
  </p:SwitchObject.DefaultOption>

</p:SwitchObject>
```

### 3.9 StopPolicy — Halt Execution

```xml
<p:StopPolicy
  CompletionResult="1"
  CompletionResult_DisplayArg="1"
  DisplayName="Stop Policy"
  StopReason="Application Not Installed"
  StopReason_DisplayArg="Application Not Installed"
  TypeName="StopPolicy"
  ... />
```

`CompletionResult`: `0` = success, `1` = failure

### 3.10 RestartService

```xml
<p:RestartService
  DisplayName="Restart Service"
  Force="False"
  Service="[Service_Name]"
  Service_DisplayArg="Input Parameters.Service Name"
  TypeName="RestartService"
  ... />
```

### 3.11 InputPrompt — User Dialog

```xml
<p:InputPrompt
  AlwaysForeground="True"
  Body="[Message]"
  Body_DisplayArg="Input Parameters.Message"
  BrandingJson="{}{&quot;ImageBase64&quot;:null,...}"
  Title="[Title]"
  Title_DisplayArg="Input Parameters.Title"
  Timeout="[TimeOut]"
  TypeName="InputPrompt"
  MinRequiredVersion="2.50.0.0"
  ...>
  <p:InputPrompt.ChosenButtons>
    <scg:List x:TypeArguments="acb:ButtonData" Capacity="1">
      <acb:ButtonData Text="Close" Type="Confirmation" />
    </scg:List>
  </p:InputPrompt.ChosenButtons>
  <p:InputPrompt.ChosenInputs>
    <scg:List x:TypeArguments="acb:InputData" Capacity="0" />
  </p:InputPrompt.ChosenInputs>
</p:InputPrompt>
```

Requires: `xmlns:acb="clr-namespace:AutomationManager.Common.Branding;assembly=AutomationManager.Common"` namespace and `MinRequiredVersion="2.50.0.0"` on Policy root.

### 3.12 FolderExists

```xml
<p:FolderExists
  Folder="[LogPath]"
  Folder_DisplayArg="Input Parameters.Log Path"
  Conditional="[FolderExists_Conditional]"
  DisplayName="Check for Log Folder"
  TypeName="FolderExists"
  ... />
```

**Key output:** `Conditional` — `x:String` set to `"True"` or `"False"` (note: lowercase `"true"`/`"false"` also observed)

### 3.13 CreateFolder

```xml
<p:CreateFolder
  Folder="[LogPath]"
  Folder_DisplayArg="Input Parameters.Log Path"
  FolderInfo="[CreateFolder_FolderInfo]"
  DisplayName="Create Folder"
  TypeName="CreateFolder"
  ... />
```

**Required variables:** `CreateFolder_FolderInfo` (x:String), `CreateFolder_ResultString`, `CreateFolder_Result`

### 3.14 CreateFile

```xml
<p:CreateFile
  FileName="[File_Path]"
  FileName_DisplayArg="Global Variables.File Path"
  Overwrite="True"
  Overwrite_DisplayArg="true"
  DisplayName="Create File"
  TypeName="CreateFile"
  ... />
```

### 3.15 SequenceActivity — Grouping Container

Used inside IfElse/IfObject branches to group multiple activities:

```xml
<p:SequenceActivity DisplayName="Then" sap:VirtualizedContainerService.HintSize="W,H" Name="SequenceActivity">
  <p:SequenceActivity.Activities>
    <!-- Child activities -->
  </p:SequenceActivity.Activities>
  <p:SequenceActivity.Variables>
    <!-- Variables scoped to this sequence -->
  </p:SequenceActivity.Variables>
</p:SequenceActivity>
```

**Important:** Variables declared in a SequenceActivity are scoped to that branch only. Activities in the Then branch cannot reference variables from the Else branch and vice versa. Shared variables must be in the parent PolicySequence.Variables.

### 3.16 ViewState — Collapse/Expand Hints

Some activities include view state metadata for the GUI:

```xml
<sap:WorkflowViewStateService.ViewState>
  <scg:Dictionary x:TypeArguments="x:String, x:Object">
    <x:Boolean x:Key="IsExpanded">False</x:Boolean>
  </scg:Dictionary>
</sap:WorkflowViewStateService.ViewState>
```

This is optional visual metadata. Include it for compatibility with Automation Manager GUI.

At the Activity level, a `ShouldCollapseAll` key can be set:
```xml
<sap:WorkflowViewStateService.ViewState>
  <scg:Dictionary x:TypeArguments="x:String, x:Object">
    <x:Boolean x:Key="ShouldCollapseAll">True</x:Boolean>
  </scg:Dictionary>
</sap:WorkflowViewStateService.ViewState>
```

---

## 4. PowerShell Embedding Specification

### Encoding Process

PowerShell scripts are embedded in the `script` attribute of `p:RunPowerShellScript` using this encoding:

1. **Write the PowerShell script** as plain text
2. **Encode as UTF-16LE** (Unicode, little-endian — this is PowerShell's native string encoding)
3. **Base64-encode** the UTF-16LE byte array
4. Place the resulting Base64 string in the `script` attribute

**Encoding command (PowerShell):**
```powershell
$script = @'
# Your PowerShell script here
Write-Host "Hello World"
Exit 0
'@
$bytes = [System.Text.Encoding]::Unicode.GetBytes($script)
$encoded = [Convert]::ToBase64String($bytes)
```

**Encoding command (Python):**
```python
script = "# Your PowerShell script here\nWrite-Host 'Hello World'\nExit 0"
encoded = base64.b64encode(script.encode('utf-16-le')).decode('ascii')
```

**Encoding command (Bash):**
```bash
echo -n "Write-Host 'Hello'" | iconv -f utf-8 -t utf-16le | base64 -w0
```

### Decoding Process (for reading .amp files)

```powershell
$decoded = [System.Text.Encoding]::Unicode.GetString([Convert]::FromBase64String($encoded))
```

```python
decoded = base64.b64decode(encoded).decode('utf-16-le')
```

### Script Restrictions Inside .amp

- **NEVER** use `$Result` or `$ResultString` — reserved by PolicyExecutor
- Parameters are passed via `$VariableName` syntax (injected by InArgs mapping)
- No user interaction (no `Read-Host`, no GUI popups from script context)
- `Write-Host` output goes to `OutPut_64` variable
- Exit code goes to `Result` variable (as Double)
- Max script size: 10 MB

---

## 5. N-Sight RMM Scripting Conventions

### 5.1 Exit Code Convention (N-Sight Specific)

| Code | Meaning | Usage |
|---|---|---|
| 0 | Success | Script Check = green on Dashboard, Automated Task = completed |
| 1001 | General failure | Default error exit for custom scripts |
| 1002+ | Specific failures | Use for distinguishable error conditions |
| 1-999 | RESERVED | **Never use** — conflicts with N-Sight system scripts |

**Critical rule:** Exit codes 1–999 are reserved by N-Sight system scripts. Using them causes your text output to NOT display on the Dashboard. Always use 1001+.

### 5.2 Output Mechanism

- **`Write-Host`** is the primary output mechanism — it writes to stdout which N-Sight captures and displays on the Dashboard
- **`Write-Output`** also works for RMM output capture (used in some advanced scripts for pipeline-compatible logging)
- **`Write-Error`** writes to stderr — visible in logs but does NOT display as Dashboard output
- **Maximum output:** 10,000 characters displayed on Dashboard (truncated beyond that)
- **Maximum script size:** 65,535 characters for Script Manager uploads

### 5.3 Standalone Scripts — Full Template (Script Manager Upload)

This is the comprehensive template for scripts uploaded directly via Settings > Script Manager. These run as Script Checks (24x7/Daily monitoring) or Automated Tasks (maintenance jobs).

```powershell
<#
.SYNOPSIS
    Brief description of what this script does

.DESCRIPTION
    Detailed description of the script's purpose, behavior, and any
    prerequisites. Mention if it requires admin/SYSTEM context.
    Designed for automated execution via N-Sight RMM.

.PARAMETER None
    This script takes no parameters (N-Sight injects via Script Manager).

.NOTES
    Author:      [Name/Organization]
    Date:        [YYYY-MM-DD]
    Version:     1.0.0
    Target:      N-Sight RMM Script Check / Automated Task
    Context:     Runs as SYSTEM (no user desktop interaction)
    PS Version:  5.1+

.EXAMPLE
    # Deployed via N-Sight RMM Dashboard as Automated Task
    # No manual execution required
#>

# ============================================================
# Configuration
# ============================================================
$ErrorActionPreference = 'Stop'
Set-StrictMode -Version Latest

# Script-level variables (hardcoded — N-Sight does not support param() for Script Manager)
$LogPath = "C:\DatoLogs\ScriptName"
$LogFile = "$LogPath\$env:COMPUTERNAME.log"

# ============================================================
# Helper Functions
# ============================================================
function Write-Log {
    param (
        [string]$Message,
        [string]$Level = "INFO"
    )
    $timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    $logEntry = "$timestamp [$Level] $Message"
    # Write to local log file
    Add-Content -Path $LogFile -Value $logEntry -ErrorAction SilentlyContinue
    # Write to RMM stdout (captured by N-Sight Dashboard)
    Write-Host $logEntry
}

# ============================================================
# Main Logic
# ============================================================
try {
    # Ensure log directory exists
    if (-not (Test-Path $LogPath)) {
        New-Item -ItemType Directory -Path $LogPath -Force | Out-Null
    }

    Write-Log "Script started on $env:COMPUTERNAME"

    # --- Your logic here ---
    # Example: Check a service, install software, run diagnostics, etc.

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

### 5.4 Embedded PowerShell in .amp — Full Template

Scripts embedded inside `.amp` files via the `Run PowerShell Script` object have **different rules** than standalone scripts:

```powershell
<#
.SYNOPSIS
    Brief description — embedded in Automation Manager policy
.DESCRIPTION
    This script runs inside an N-Sight Automation Manager policy.
    Exit codes are consumed by the Run PowerShell Script module
    and do NOT propagate to the Dashboard directly.
    Use Write-Host for output captured by OutPut_64 variable.
#>

# IMPORTANT: Do NOT use $Result or $ResultString — reserved by PolicyExecutor
$ErrorActionPreference = 'Stop'

try {
    # Input parameters are injected via InArgs mapping
    # Access them as $ParameterName (matches InArg x:Key)

    # --- Your logic here ---
    # Example: $logPath is injected from policy Input Parameters

    # Output goes to OutPut_64 variable in the policy
    Write-Host "Operation completed successfully"
}
catch {
    Write-Host "Error: $($_.Exception.Message)"
    # Do NOT use Exit here — the PolicyExecutor handles result codes
    # Wire up If/Else + Fail Policy in the .amp workflow to surface failures
}
```

**Key differences from standalone scripts:**

| Aspect | Standalone (Script Manager) | Embedded (.amp) |
|---|---|---|
| Exit codes | `Exit 0` / `Exit 1001` directly control Dashboard | Exit codes consumed internally — use `If/Else` + `Fail Policy` to surface |
| Parameters | Hardcoded variables (no `param()`) | Injected via `InArgs` mapping from policy variables |
| Reserved vars | None | `$Result`, `$ResultString` — **NEVER use** |
| Output | `Write-Host` → Dashboard display | `Write-Host` → `OutPut_64` variable |
| Max size | 65,535 characters | 10 MB |
| `Exit` usage | Required (`Exit 0`, `Exit 1001`) | **Avoid** — let PolicyExecutor manage result |

### 5.5 Script Structure Patterns (from Example Corpus)

The example `.amp` files demonstrate three tiers of PowerShell complexity:

**Tier 1 — Inline (no structure):**
Used in simple utilities like DisablePin.amp. No comment-based help, no error handling, no functions. Just direct commands:
```powershell
#Disable pin requirement
$path = "HKLM:\SOFTWARE\Policies\Microsoft"
$key = "PassportForWork"
New-Item -Path $path -Name $key -Force
New-ItemProperty -Path $path\$key -Name "Enabled" -Value "0" -PropertyType DWORD -Force
```

**Tier 2 — Structured (comment-based help + try/catch):**
Used in utilities like DismSfcScan.amp. Includes comment-based help, admin check, logging function, try/catch with exit codes:
```powershell
<#
.SYNOPSIS
    Runs system file checker with enhanced logging for RMM deployment.
.DESCRIPTION
    Designed for automated execution via RMM with no user interaction.
    Requires administrative privileges to run.
#>

# Admin check (useful even in SYSTEM context for validation)
$isAdmin = ([Security.Principal.WindowsPrincipal] [Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole(
    [Security.Principal.WindowsBuiltInRole]::Administrator)
if (-not $isAdmin) {
    Write-Error "This script must be run as Administrator."
    exit 1
}

# Logging setup
$mainLog = "$logPath\SystemRepair.log"

function Write-Log {
    param ([string]$Message, [string]$LogLevel = "INFO")
    $logMessage = "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss') [$LogLevel] $Message"
    Add-Content -Path $mainLog -Value $logMessage -NoNewline:$false
    if ($Host.Name -eq 'Default Host') {
        Write-Output $logMessage
    }
}

try {
    $exitCode = 0
    Write-Log "Starting DISM repair process..."
    # ... main logic ...
    exit $exitCode
}
catch {
    Write-Log "An error occurred: $($_.Exception.Message)" -LogLevel "ERROR"
    exit 2
}
```

**Tier 3 — Modular (Build.ps1 pattern):**
Used for complex deployments. The project's `Build.ps1` demonstrates concatenating helper functions from `src/helpers/` into a single deployable `.ps1`. This pattern:
- Separate files per function (`Write-Log.ps1`, `Invoke-GpUpdate.ps1`, etc.)
- Build script concatenates in dependency order
- `param()` block at top of output (valid for direct execution, not Script Manager)
- `#region`/`#endregion` markers for each helper
- Main entry point with nested try/catch and multiple fallback methods

### 5.6 PowerShell Best Practices for N-Sight RMM

**Error Handling:**
- Always set `$ErrorActionPreference = 'Stop'` — converts non-terminating errors into catchable exceptions
- Use `Set-StrictMode -Version Latest` to catch uninitialized variables and property access errors
- Wrap risky operations in `try/catch` — but only where you need to add context, attempt recovery, or continue after non-critical failure
- Capture `$_` or `$_.Exception.Message` immediately in catch blocks — subsequent commands can overwrite `$_`
- Use `-ErrorAction SilentlyContinue` on specific cmdlets where failure is expected and handled

**Variable Handling:**
- **NEVER** use `$Result` or `$ResultString` in embedded scripts — reserved by PolicyExecutor
- Parameters in embedded scripts come from `InArgs` mapping — use `$ParameterName` matching the `x:Key`
- For standalone scripts: hardcode all inputs as variables (N-Sight Script Manager does not support `param()` blocks or `$args`)
- Use `$env:COMPUTERNAME`, `$env:TEMP`, `$env:USERNAME` for system properties

**Output Best Practices:**
- Use `Write-Host` for all output that should appear on the N-Sight Dashboard
- Prefix output with status labels: `[INFO]`, `[SUCCESS]`, `[ERROR]`, `[WARN]` for parseable output
- Keep output under 10,000 characters — N-Sight truncates beyond this
- Include the computer name in output for multi-device deployment clarity: `"[$env:COMPUTERNAME] Operation completed"`

**File & Path Operations:**
- Always use absolute paths — scripts run as SYSTEM from an unpredictable working directory
- Test paths before operating: `if (Test-Path $path) { ... }`
- Use `Join-Path` for path construction: `$logFile = Join-Path $logDir "$env:COMPUTERNAME.log"`
- Create directories with `-Force` flag: `New-Item -ItemType Directory -Path $dir -Force | Out-Null`

**Process Execution:**
- Use `Start-Process` with `-Wait -PassThru` for external executables to capture exit codes
- Capture exit codes: `$process = Start-Process ... -PassThru; $process.ExitCode`
- Use `-NoNewWindow` to prevent UI windows in SYSTEM context
- Redirect stdout/stderr to temp files: `-RedirectStandardOutput $outFile -RedirectStandardError $errFile`

**Registry Operations:**
- Use `HKLM:\` (HKEY_LOCAL_MACHINE) for machine-wide settings
- Use `New-Item -Path $path -Name $key -Force` to create keys
- Use `New-ItemProperty` with `-PropertyType` (DWORD, String, etc.) and `-Force`
- Always specify `-Force` to overwrite existing values without errors

**Service Operations:**
- Check service existence: `Get-Service -Name $serviceName -ErrorAction SilentlyContinue`
- Restart with timeout: `Restart-Service -Name $serviceName -Force`
- For the Automation Manager `RestartService` object: pass service name via Input Parameters

**SYSTEM Context Gotchas:**
- Scripts run as `NT AUTHORITY\SYSTEM` — no user profile, no desktop, no mapped drives
- `$env:USERPROFILE` points to `C:\Windows\system32\config\systemprofile`
- No access to HKCU (current user registry hive) — use HKLM
- `quser.exe` requires `AllowRemoteRPC` registry key to enumerate sessions from SYSTEM
- Network paths require explicit UNC paths (`\\server\share`) — no mapped drive letters
- Some tools like `winget` need special path resolution in SYSTEM context:
  ```powershell
  $wingetExe = (Resolve-Path "$env:ProgramFiles\WindowsApps\Microsoft.DesktopAppInstaller_*_x64_*\winget.exe").Path
  & $wingetExe install PackageId --silent --accept-source-agreements
  ```

### 5.7 Logging Patterns

The example corpus demonstrates two logging approaches:

**Simple (Write-Host only):**
```powershell
Write-Host "[INFO] Starting operation on $env:COMPUTERNAME"
Write-Host "[SUCCESS] Operation completed"
Write-Host "[ERROR] Failed: $($_.Exception.Message)"
```

**Structured (function + file + stdout):**
```powershell
function Write-Log {
    param (
        [string]$Message,
        [string]$LogLevel = "INFO"
    )
    $logMessage = "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss') [$LogLevel] $Message"
    Add-Content -Path $script:LogFile -Value $logMessage
    Write-Host $logMessage
}
```

The structured pattern writes to both a local log file (for persistent debugging) and stdout (for N-Sight Dashboard capture). The Build.ps1 project uses this dual-output pattern extensively.

### 5.8 Password and Credential Handling

- **Password parameters in Automation Manager are NOT safe** — treated as plain `System.String`
- N-Sight validates passwords to block these characters: `" ' / \ $ \``
- For API keys and secrets: pass them as Input Parameters, but understand they are stored in the `.amp` XML in clear text (within the HTML-encoded Object Data)
- For production deployments: consider retrieving secrets from a secure vault at runtime rather than embedding in the policy

---

## 6. Policy Categories & Templates

Based on the 29 example `.amp` files, policies fall into 4 categories:

### 6.1 Deployment Policies

**Purpose:** Install/deploy software to endpoints.
**Pattern:** Check if installed → Download → Install → Verify → Log result
**Complexity:** High — multiple RunPowerShellScript activities, conditional branches, error handling
**Examples:** DnsFilterDeployment2025.amp, SentinelOneDeployment2025.amp, HuntressDeployment2025.amp

**Typical structure:**
1. Parameters: installer URL, API key, organization ID, etc.
2. Check if already installed (IsAppInstalled or registry check via PS)
3. Download installer
4. Run installer silently
5. Verify installation
6. Set environment variable as installation marker
7. Log results

### 6.2 Checker Policies

**Purpose:** Verify software is installed and running, manage install/uninstall state tracking.
**Pattern:** IsAppInstalled → GetEnvironmentVariable → IfElse/Switch → SetEnvironmentVariable → StopPolicy on failure
**Complexity:** Medium — uses conditional logic and environment variables for state tracking
**Examples:** CoveChecker.amp, DnsFilterChecker.amp, SentinelOneChecker.amp

**Typical structure:**
1. Parameters: `Task_Check` (0=uninstall, 1=install)
2. GlobalVariables: `Program` (application name)
3. IsAppInstalled check
4. Get environment variable (state marker)
5. IfElse: compare state with expected
6. Update environment variable
7. SwitchObject on Task_Check for install/uninstall outcome
8. StopPolicy with failure reason if mismatch

### 6.3 Utility Policies

**Purpose:** System administration tasks (registry edits, service management, cleanup, diagnostics).
**Pattern:** Usually a single RunPowerShellScript + Log activities
**Complexity:** Low to medium
**Examples:** DisablePin.amp, RestartService.amp, GPUpdate.amp, CleanNableFiles.amp

**Typical structure:**
1. Optional parameters (e.g., service name)
2. Run PowerShell Script with the utility logic
3. Log the output
4. Log the result string

### 6.4 ClientTools Policies

**Purpose:** Client-facing tools (messages, prompts, file operations).
**Pattern:** Parameters for user input → Execute action → Log
**Complexity:** Low to medium
**Examples:** Message.amp, LanMessage.amp, AddTextFile.amp

**Typical structure:**
1. Parameters: message text, title, timeout
2. InputPrompt or RunPowerShellScript
3. Log results

---

## 7. Plugin Architecture

### Target Plugin Structure

```
nable-nsight-scripter/
├── .claude-plugin/
│   └── plugin.json
├── README.md
├── LICENSE
├── skills/
│   └── nsight-scripter/
│       ├── SKILL.md                    # Main skill — comprehensive .amp knowledge
│       └── references/
│           ├── amp-format-spec.md      # Extracted from this design doc §2-4
│           ├── activity-types.md       # Extracted from this design doc §3
│           ├── powershell-conventions.md # Extracted from this design doc §5
│           └── policy-templates.md     # Extracted from this design doc §6
├── commands/
│   ├── nsight-create.md               # /nsight-create — create new .amp from description
│   ├── nsight-edit.md                 # /nsight-edit — modify existing .amp
│   ├── nsight-read.md                 # /nsight-read — decode and explain an .amp file
│   └── nsight-script.md              # /nsight-script — generate standalone PS script for N-Sight
└── Examples/                          # Symlink or copy of example .amp files
    ├── Deployments/
    ├── Checkers/
    ├── Utilities/
    └── ClientTools/
```

### plugin.json

```json
{
  "name": "nsight-scripter",
  "description": "Create, edit, read, and manage N-Able N-Sight RMM Automation Manager (.amp) policy files and PowerShell scripts",
  "author": {
    "name": "DatoTech"
  }
}
```

### SKILL.md — The Core Knowledge

The SKILL.md file is the primary knowledge source. When loaded, it gives Claude complete understanding of:
- The `.amp` XML schema (sections 2-4 of this document)
- All activity types and their attributes
- PowerShell embedding/encoding process
- N-Sight RMM constraints and conventions
- How to read the example files for patterns

**Frontmatter:**
```yaml
---
name: nsight-scripter
description: "Expert knowledge for creating and editing N-Able N-Sight RMM .amp policy files and PowerShell automation scripts"
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
user-invocable: true
disable-model-invocation: false
---
```

### Commands

Each command is a focused entry point:

**`/nsight-create`** — "Create a new .amp policy from a natural language description"
- Allowed tools: Read, Write, Bash, Glob
- Flow: Ask what the policy should do → determine category → generate XML with proper schema → encode any PowerShell → write .amp file

**`/nsight-edit`** — "Edit an existing .amp file"
- Allowed tools: Read, Write, Edit, Bash, Glob
- Flow: Read .amp → decode embedded scripts → show user what exists → apply changes → re-encode → write

**`/nsight-read`** — "Decode and explain an .amp file"
- Allowed tools: Read, Bash, Glob
- Flow: Read .amp → parse XML → decode Base64 scripts → explain structure, parameters, workflow logic

**`/nsight-script`** — "Generate a standalone PowerShell script for N-Sight Script Manager"
- Allowed tools: Read, Write, Bash
- Flow: Ask what the script should do → generate with proper N-Sight conventions (exit codes > 1000, Write-Host output, comment-based help)

---

## 8. Plugin Commands & Skills — Detailed Specifications

### /nsight-create Command

```yaml
---
allowed-tools: Read, Write, Bash, Glob, Grep
description: Create a new N-Sight RMM .amp automation policy file
argument-hint: "[deployment|checker|utility|clienttool] description of what the policy should do"
disable-model-invocation: false
---
```

**Behavior:**
1. Parse the user's description to determine policy category
2. Read matching example .amp files from `Examples/` for reference patterns
3. Generate unique GUIDs for Policy ID, Object ID, and activity Monikers
4. Base64-encode the description (UTF-8)
5. If PowerShell is needed: write the script, then Base64-encode it (UTF-16LE)
6. Assemble the complete XML using the schema from this document
7. Write the .amp file
8. Offer to decode and review it

**GUID Generation (via Bash):**
```bash
python3 -c "import uuid; print(str(uuid.uuid4()))"
```

**Base64 Encoding (via Bash):**
```bash
# Description encoding (UTF-8)
echo -n "My Policy Description" | base64

# PowerShell script encoding (UTF-16LE)
echo -n 'Write-Host "Hello"' | iconv -f utf-8 -t utf-16le | base64 -w0
```

### /nsight-edit Command

```yaml
---
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
description: Edit an existing N-Sight RMM .amp automation policy file
argument-hint: "path/to/file.amp"
disable-model-invocation: false
---
```

**Behavior:**
1. Read the .amp file
2. Parse the XML structure
3. Decode any Base64 PowerShell scripts
4. Present the policy structure to the user (name, description, parameters, workflow, scripts)
5. Ask what changes are needed
6. Apply changes (re-encode scripts if modified)
7. Write the modified .amp file

### /nsight-read Command

```yaml
---
allowed-tools: Read, Bash, Glob, Grep
description: Decode and explain an N-Sight RMM .amp file
argument-hint: "path/to/file.amp"
disable-model-invocation: false
---
```

**Behavior:**
1. Read the .amp file
2. Decode description (Base64 → UTF-8)
3. Parse Object Data (HTML-decode → extract Parameters and GlobalVariables)
4. Walk the Activity workflow and list all activities in execution order
5. Decode any embedded PowerShell scripts (Base64 → UTF-16LE → text)
6. Present a human-readable summary

### /nsight-script Command

```yaml
---
allowed-tools: Read, Write, Bash
description: Generate a standalone PowerShell script for N-Sight RMM Script Manager
argument-hint: "description of what the script should do"
disable-model-invocation: false
---
```

**Behavior:**
1. Ask what the script should do
2. Determine if it's a Script Check (monitoring) or Automated Task (maintenance)
3. Generate PowerShell with:
   - Comment-based help (.SYNOPSIS, .DESCRIPTION, .NOTES)
   - `$ErrorActionPreference = 'Stop'`
   - try/catch with `Exit 0` (success) and `Exit 1001` (failure)
   - `Write-Host` for Dashboard output
   - Under 65,535 characters
4. Write the .ps1 file

---

## 9. Example Corpus Reference

The plugin should reference these example files in `/home/datotech/projects/nable-nsight-scripter/Examples/` when generating new policies:

### Simple (start here for reference):
- `Utilities/DisablePin.amp` — Single RunPowerShellScript + 2 Logs (40 lines)
- `Utilities/RestartService.amp` — Single RestartService with parameter (24 lines)
- `ClientTools/LanMessage.amp` — RunPowerShellScript with InArgs parameter mapping (35 lines)
- `ClientTools/Message.amp` — InputPrompt with branding and parameters (42 lines)

### Medium (conditional logic):
- `Checkers/CoveChecker.amp` — IsAppInstalled + IfElse + SwitchObject + StopPolicy (273 lines)
- All Checkers follow this same pattern

### Complex (full deployment workflows):
- `Deployments/HuntressDeployment2025.amp` — Smallest deployment (1053 lines)
- `Deployments/SentinelOneDeployment2025.amp` — Largest deployment (1213 lines)

### Build System:
- `Build.ps1` — PowerShell build script that concatenates helper functions into deployable scripts. Demonstrates the project's convention for structured PowerShell (error handling, logging, modular functions)

---

## 10. Verification Plan

### After Plugin Implementation

- [ ] **Create a simple utility .amp** — Use `/nsight-create utility disable Windows hibernation`. Verify the XML is valid, Base64 decodes correctly, all GUIDs are unique, all required elements present.

- [ ] **Create a checker .amp** — Use `/nsight-create checker verify if Huntress agent is installed`. Verify IfElse/SwitchObject structure matches CoveChecker.amp pattern.

- [ ] **Read an existing .amp** — Use `/nsight-read Examples/Utilities/DisablePin.amp`. Verify it correctly decodes the PowerShell script and identifies all activities.

- [ ] **Edit an existing .amp** — Use `/nsight-edit Examples/ClientTools/Message.amp` to change the default message. Verify only the intended changes are made.

- [ ] **Generate a standalone script** — Use `/nsight-script check if a Windows service is running`. Verify it has comment-based help, try/catch, Exit 0/1001, Write-Host output.

- [ ] **Validate Base64 encoding** — For any generated .amp, extract the `script` attribute and decode it:
  ```bash
  echo "{base64string}" | base64 -d | iconv -f utf-16le -t utf-8
  ```
  Verify the decoded PowerShell is correct.

- [ ] **Validate XML structure** — For any generated .amp:
  ```bash
  xmllint --noout generated-policy.amp
  ```
  Verify no XML parsing errors.

- [ ] **Cross-reference with examples** — Compare generated .amp files against the example corpus to ensure structural consistency (same namespace declarations, same boilerplate elements, same variable naming conventions).
