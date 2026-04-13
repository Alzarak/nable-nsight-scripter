# N-Sight RMM Policy Schema Rules and Templates

Complete reference for .amp policy XML schema rules, category patterns, and ready-to-use templates.

---

## Part 1 — Schema Rules

For complete .amp file structure, encoding rules, and element specifications, see **amp-format-spec.md**. For individual activity XML snippets, see **activity-types.md**. This section covers rules specific to template generation that are not fully detailed elsewhere.

### Conditional Namespaces

Add these only when the corresponding activity types are used (base namespaces are shown in each template):

| Prefix | Declaration | When needed |
|--------|-------------|-------------|
| `sco` | `xmlns:sco="clr-namespace:System.Collections.ObjectModel;assembly=mscorlib"` | SwitchObject activities (empty DefaultOption collections) |
| `acb` | `xmlns:acb="clr-namespace:AutomationManager.Common.Branding;assembly=AutomationManager.Common"` | InputPrompt activities (ButtonData/InputData types) |

### Null Attribute Ordering Convention

`_Item="{x:Null}"` and `_ItemProp="{x:Null}"` attributes always come FIRST on any activity element, before other attributes.

### Variable Scoping Rules

Variables are declared in the nearest containing `.Variables` block:

| Scope | Element | What goes here |
|-------|---------|---------------|
| Policy-level | `p:PolicySequence.Variables` | Parameters, global variables, top-level activity outputs (IsAppInstalled, GetEnvironmentVariable, IfElse, SwitchObject) |
| Branch-level | `p:SequenceActivity.Variables` | Activity outputs inside IfElse/IfObject/CaseObject branches |
| Case-level | `p:CaseSequenceActivity.Variables` | CaseObject Result/ResultString variables |

**Rule:** Activities inside a branch declare their output variables in that branch's `SequenceActivity.Variables`, NOT in the parent `PolicySequence.Variables`.

### DisplayArg Naming Convention

| Source | Pattern | Example |
|--------|---------|---------|
| Input Parameter | `"Input Parameters.{Label}"` | `"Input Parameters.Message"` |
| Global Variable | `"Global Variables.{Label}"` | `"Global Variables.Program"` |
| Activity Output | `"{ActivityDisplayName}.{OutputField}"` | `"Run PowerShell Script.OutPut_64"` |
| Static Value | The literal value itself | `"True"`, `"0"`, `"Machine"` |

### Object Data Encoding

The `<Object>` `Data` attribute uses HTML-entity-encoded XML with `ParameterName` and `Label` attributes (not `Name` and `Variable`):

```xml
Data="&lt;xml&gt;&lt;Parameters&gt;&lt;Parameter ParameterName=&quot;Task_Check&quot; Label=&quot;Uninstall = 0 Install = 1&quot; ParameterType=&quot;number&quot; Value=&quot;1&quot; /&gt;&lt;/Parameters&gt;&lt;GlobalVariables&gt;&lt;Parameter ParameterName=&quot;Program&quot; Label=&quot;Program&quot; ParameterType=&quot;string&quot; Value=&quot;APPLICATION_DISPLAY_NAME&quot; /&gt;&lt;/GlobalVariables&gt;&lt;/xml&gt;"
```

When no parameters exist: `Data="&lt;xml /&gt;"`

---

## Part 2 — Policy Categories

### 1. Deployment Policies

- **Purpose:** Install or deploy software to endpoints
- **Pattern:** Override SwitchObject -> IsAppInstalled check -> Download -> Install -> Verify -> SetEnvironmentVariable
- **Complexity:** High — multiple `RunPowerShellScript` activities, conditional branches via `IfElse` and `SwitchObject`
- **Typical structure:**
  1. Parameters — Override (B/U/R), URL, API key, software folder/file paths, argument list, log paths
  2. Global variables — `Software`, `Installed_Application`, `Remove`, `Install`, `BlobURL`, `User_Agent`
  3. `SwitchObject` on Override parameter (Bypass / Uninstall / Reinstall) setting control variables
  4. `IsAppInstalled` check against `Installed_Application`
  5. `IfElse` — if not installed: download, install, verify; if installed and Remove=1: uninstall
  6. `SetEnvironmentVariable` — marker for checker policies to read
  7. `Log` activities throughout

### 2. Checker Policies

- **Purpose:** Verify software is installed and running; manage install/uninstall state tracking via environment variables
- **Pattern:** `IsAppInstalled` -> `GetEnvironmentVariable` -> `IfElse`/`IfObject`/`SetEnvironmentVariable` -> `SwitchObject`/`StopPolicy`
- **Complexity:** Medium
- **Typical structure:**
  1. Parameter `Task_Check` (0 = uninstall, 1 = install)
  2. Global variable `Program` — the application display name
  3. `IsAppInstalled` using `Program`
  4. `GetEnvironmentVariable` — reads machine-level env var named after `Program`
  5. `IfElse` — reconciles env var with installed state (sets to "0" or "1")
  6. `SwitchObject` on `Task_Check` — verifies state matches desired action, `StopPolicy` on mismatch
  7. Final `Log` activity

### 3. Utility Policies

- **Purpose:** System administration tasks — registry edits, service management, cleanup, diagnostics
- **Pattern:** Usually a single `RunPowerShellScript` plus `Log` activities; sometimes native activities
- **Complexity:** Low to medium
- **Typical structure:**
  1. Optional parameters (e.g., service name, path)
  2. `RunPowerShellScript` with base64-encoded script (or native activity like `RestartService`)
  3. `Log` — output from the script (`OutPut_64`)
  4. `Log` — result string (`ResultString`)

### 4. ClientTools Policies

- **Purpose:** Client-facing tools — display messages, prompts, file operations on the endpoint
- **Pattern:** Parameters for user input -> Execute -> Log
- **Complexity:** Low to medium
- **Typical structure:**
  1. Parameters (message, title, timeout, file path, etc.)
  2. `InputPrompt` activity (for GUI messages) or `RunPowerShellScript` (for non-GUI operations)
  3. Optional `Log` activities

---

## Part 3 — Complete Templates

### Template 1: Simple Utility

Minimal policy: run a PowerShell script and log the output. No parameters needed.

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<Policy ID="{NEW-GUID}" Name="POLICY_NAME" Description="BASE64_ENCODED_DESCRIPTION" Version="2.10.0.19" RemoteCategory="0" ExecutionType="Local" MinimumPSVersionRequired="0.0.0">
  <Object ID="{NEW-GUID}" Type="{B6FA6D8B-EEAA-47A6-8463-7F9A4F5BBB6E}" Data="&lt;xml /&gt;" />
  <LinkManager xmlns:i="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://schemas.datacontract.org/2004/07/PolicyExecutor">
    <hashset xmlns:d2p1="http://schemas.datacontract.org/2004/07/System" />
  </LinkManager>
  <Diagnostics OriginalVersion="2.98.2.2" />
  <Activity mc:Ignorable="sads sap" x:Class="Policy Builder"
    xmlns="http://schemas.microsoft.com/netfx/2009/xaml/activities"
    xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
    xmlns:mva="clr-namespace:Microsoft.VisualBasic.Activities;assembly=System.Activities"
    xmlns:p="clr-namespace:PolicyExecutor;assembly=PolicyExecutionEngine"
    xmlns:sads="http://schemas.microsoft.com/netfx/2010/xaml/activities/debugger"
    xmlns:sap="http://schemas.microsoft.com/netfx/2009/xaml/activities/presentation"
    xmlns:scg="clr-namespace:System.Collections.Generic;assembly=mscorlib"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <x:Members>
      <x:Property Name="PolicyGUID" Type="InArgument(x:String)" />
    </x:Members>
    <sap:VirtualizedContainerService.HintSize>490,827</sap:VirtualizedContainerService.HintSize>
    <mva:VisualBasic.Settings>Assembly references and imported namespaces serialized as XML namespaces</mva:VisualBasic.Settings>
    <p:PolicySequence DisplayName="Policy Builder" sap:VirtualizedContainerService.HintSize="490,827"
      mva:VisualBasic.Settings="Assembly references and imported namespaces serialized as XML namespaces">
      <p:PolicySequence.Activities>

        <!-- ACTION: Run PowerShell Script -->
        <p:RunPowerShellScript genArgEvent="{x:Null}"
          AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
          DisplayName="Run PowerShell Script"
          sap:VirtualizedContainerService.HintSize="454,348"
          Moniker="{NEW-GUID}"
          OutPut_64="[RunPowerShellScript_OutPut_64]"
          Result="[RunPowerShellScript_Result]"
          ResultString="[RunPowerShellScript_ResultString]"
          Results_x64="[RunPowerShellScript_Results_x64]"
          RunAsCurrentLoggedOnUser="False"
          ScriptExecutionMethod="ExecuteDebug"
          TypeName="RunPowerShellScript"
          m_bTextLinkChange="False"
          script="BASE64_ENCODED_UTF16LE_SCRIPT">
          <p:RunPowerShellScript.InArgs>
            <scg:Dictionary x:TypeArguments="x:String, p:InArg" />
          </p:RunPowerShellScript.InArgs>
          <p:RunPowerShellScript.OutArgs>
            <scg:Dictionary x:TypeArguments="x:String, p:OutArg" />
          </p:RunPowerShellScript.OutArgs>
        </p:RunPowerShellScript>

        <!-- LOG: Script output -->
        <p:Log Message_Item="{x:Null}" Message_ItemProp="{x:Null}"
          AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
          DisplayName="Log"
          sap:VirtualizedContainerService.HintSize="454,88"
          LogMessage="[Log_LogMessage]"
          Message="[RunPowerShellScript_OutPut_64]"
          Message_DisplayArg="Run PowerShell Script.OutPut_64"
          Moniker="{NEW-GUID}"
          Result="[Log_Result]"
          ResultString="[Log_ResultString]"
          RunAsCurrentLoggedOnUser="False"
          ScriptExecutionMethod="ExecuteDebug"
          TypeName="Log"
          m_bTextLinkChange="False" />

        <!-- LOG: Result string -->
        <p:Log Message_Item="{x:Null}" Message_ItemProp="{x:Null}"
          AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
          DisplayName="Log"
          sap:VirtualizedContainerService.HintSize="454,88"
          LogMessage="[Log_LogMessage_1]"
          Message="[RunPowerShellScript_ResultString]"
          Message_DisplayArg="Run PowerShell Script.Result String"
          Moniker="{NEW-GUID}"
          Result="[Log_Result_1]"
          ResultString="[Log_ResultString_1]"
          RunAsCurrentLoggedOnUser="False"
          ScriptExecutionMethod="ExecuteDebug"
          TypeName="Log"
          m_bTextLinkChange="False" />

      </p:PolicySequence.Activities>
      <p:PolicySequence.Variables>
        <Variable x:TypeArguments="x:String" Name="RunPowerShellScript_OutPut_64" />
        <Variable x:TypeArguments="x:String" Name="RunPowerShellScript_ResultString" />
        <Variable x:TypeArguments="scg:IEnumerable(x:Object)" Name="RunPowerShellScript_Results_x64" />
        <Variable x:TypeArguments="x:Double" Name="RunPowerShellScript_Result" />
        <Variable x:TypeArguments="x:String" Name="Log_LogMessage" />
        <Variable x:TypeArguments="x:String" Name="Log_ResultString" />
        <Variable x:TypeArguments="x:Double" Name="Log_Result" />
        <Variable x:TypeArguments="x:String" Name="Log_LogMessage_1" />
        <Variable x:TypeArguments="x:String" Name="Log_ResultString_1" />
        <Variable x:TypeArguments="x:Double" Name="Log_Result_1" />
      </p:PolicySequence.Variables>
    </p:PolicySequence>
  </Activity>
</Policy>
```

**Placeholders:**
- `{NEW-GUID}` — generate a unique GUID for each occurrence
- `POLICY_NAME` — the policy name (no spaces, use underscores)
- `BASE64_ENCODED_DESCRIPTION` — base64-encode the plain-text description (UTF-8)
- `BASE64_ENCODED_UTF16LE_SCRIPT` — the PowerShell script encoded as base64 from UTF-16LE bytes

---

### Template 2: Checker

Complete checker: verify app installed state, reconcile environment variable, stop on mismatch with desired action.

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<Policy ID="{NEW-GUID}" Name="GenericAppChecker" Description="BASE64_ENCODED_DESCRIPTION"
  Version="2.19.0.1" MinRequiredVersion="2.19.0.1" RemoteCategory="0"
  ExecutionType="Local" MinimumPSVersionRequired="3.0">
  <Object ID="{NEW-GUID}" Type="{B6FA6D8B-EEAA-47A6-8463-7F9A4F5BBB6E}"
    Data="&lt;xml&gt;&lt;Parameters&gt;&lt;Parameter ParameterName=&quot;Task_Check&quot; Label=&quot;Uninstall = 0 Install = 1&quot; ParameterType=&quot;number&quot; Value=&quot;1&quot; /&gt;&lt;/Parameters&gt;&lt;GlobalVariables&gt;&lt;Parameter ParameterName=&quot;Program&quot; Label=&quot;Program&quot; ParameterType=&quot;string&quot; Value=&quot;APPLICATION_DISPLAY_NAME&quot; /&gt;&lt;/GlobalVariables&gt;&lt;/xml&gt;" />
  <LinkManager xmlns:i="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://schemas.datacontract.org/2004/07/PolicyExecutor">
    <hashset xmlns:d2p1="http://schemas.datacontract.org/2004/07/System" />
  </LinkManager>
  <Diagnostics OriginalVersion="2.98.2.2" />
  <Activity mc:Ignorable="sads sap" x:Class="Policy Builder"
    xmlns="http://schemas.microsoft.com/netfx/2009/xaml/activities"
    xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
    xmlns:mva="clr-namespace:Microsoft.VisualBasic.Activities;assembly=System.Activities"
    xmlns:p="clr-namespace:PolicyExecutor;assembly=PolicyExecutionEngine"
    xmlns:sads="http://schemas.microsoft.com/netfx/2010/xaml/activities/debugger"
    xmlns:sap="http://schemas.microsoft.com/netfx/2009/xaml/activities/presentation"
    xmlns:scg="clr-namespace:System.Collections.Generic;assembly=mscorlib"
    xmlns:sco="clr-namespace:System.Collections.ObjectModel;assembly=mscorlib"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <x:Members>
      <x:Property Name="PolicyGUID" Type="InArgument(x:String)" />
    </x:Members>
    <sap:VirtualizedContainerService.HintSize>341,667</sap:VirtualizedContainerService.HintSize>
    <mva:VisualBasic.Settings>Assembly references and imported namespaces serialized as XML namespaces</mva:VisualBasic.Settings>
    <p:PolicySequence DisplayName="Policy Builder" sap:VirtualizedContainerService.HintSize="341,667"
      MinRequiredVersion="2.19.0.1"
      mva:VisualBasic.Settings="Assembly references and imported namespaces serialized as XML namespaces">
      <p:PolicySequence.Activities>

        <!-- STEP 1: Check if app is installed -->
        <p:IsAppInstalled ApplicationName_Item="{x:Null}" ApplicationName_ItemProp="{x:Null}"
          ApplicationName="[Program]"
          ApplicationName_DisplayArg="Global Variables.Program"
          AssemblyName="PolicyExecutionEngine, Version=2.96.1.1, Culture=neutral, PublicKeyToken=null"
          Conditional="[IsAppInstalled_Conditional]"
          DisplayName="Is Application Installed"
          sap:VirtualizedContainerService.HintSize="305,81"
          MinRequiredVersion="2.10.0.19"
          Moniker="{NEW-GUID}"
          Result="[IsAppInstalled_Result]"
          ResultString="[IsAppInstalled_ResultString]"
          RunAsCurrentLoggedOnUser="False"
          ScriptExecutionMethod="None"
          TypeName="IsAppInstalled"
          m_bTextLinkChange="False" />

        <!-- STEP 2: Read current env var state -->
        <p:GetEnvironmentVariable Type_Item="{x:Null}" Type_ItemProp="{x:Null}"
          Variable_Item="{x:Null}" Variable_ItemProp="{x:Null}"
          AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
          DisplayName="Get Environment Variable"
          sap:VirtualizedContainerService.HintSize="305,81"
          MinRequiredVersion="2.10.0.19"
          Moniker="{NEW-GUID}"
          Result="[GetEnvironmentVariable_Result]"
          ResultString="[GetEnvironmentVariable_ResultString]"
          RunAsCurrentLoggedOnUser="False"
          ScriptExecutionMethod="ExecuteDebug"
          Type="Machine"
          TypeName="GetEnvironmentVariable"
          Type_DisplayArg="Machine"
          Value="[GetEnvironmentVariable_Value]"
          Variable="[Program]"
          Variable_DisplayArg="Global Variables.Program"
          m_bTextLinkChange="False" />

        <!-- STEP 3: IfElse — reconcile env var with installed state -->
        <p:IfElse CaseSensitive_Item="{x:Null}" CaseSensitive_ItemProp="{x:Null}"
          Condition_Item="{x:Null}" Condition_ItemProp="{x:Null}"
          Value_Item="{x:Null}" Value_ItemProp="{x:Null}"
          Variable_Item="{x:Null}" Variable_ItemProp="{x:Null}"
          AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
          CaseSensitive="False" CaseSensitive_DisplayArg="false"
          Condition="equals" Condition_DisplayArg="equals"
          DisplayName="Set Environment Variable"
          sap:VirtualizedContainerService.HintSize="305,81"
          MinRequiredVersion="2.19.0.1"
          Moniker="{NEW-GUID}"
          Result="[IfElse_Result]" ResultString="[IfElse_ResultString]"
          RunAsCurrentLoggedOnUser="False"
          ScriptExecutionMethod="None"
          TypeName="IfElse"
          Value_DisplayArg="True" Value_Type="x:String"
          Variable="[IsAppInstalled_Conditional]"
          Variable_DisplayArg="Is Application Installed.Conditional"
          Variable_Type="x:String"
          m_bTextLinkChange="False">
          <p:IfElse.IfOption>
            <!-- APP IS INSTALLED: set env var to 0 if not already 0 -->
            <p:SequenceActivity DisplayName="Then" sap:VirtualizedContainerService.HintSize="172,81" Name="SequenceActivity">
              <p:SequenceActivity.Activities>
                <p:IfObject CaseSensitive_Item="{x:Null}" CaseSensitive_ItemProp="{x:Null}"
                  Condition_Item="{x:Null}" Condition_ItemProp="{x:Null}"
                  Value_Item="{x:Null}" Value_ItemProp="{x:Null}"
                  Variable_Item="{x:Null}" Variable_ItemProp="{x:Null}"
                  VerboseOutput_Item="{x:Null}" VerboseOutput_ItemProp="{x:Null}"
                  AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
                  CaseSensitive="False" CaseSensitive_DisplayArg="false"
                  Condition="does not equal" Condition_DisplayArg="does not equal"
                  DisplayName="If"
                  sap:VirtualizedContainerService.HintSize="435,81"
                  MinRequiredVersion="2.19.0.1"
                  Moniker="{NEW-GUID}"
                  Result="[IfObject_Result]" ResultString="[IfObject_ResultString]"
                  RunAsCurrentLoggedOnUser="False"
                  ScriptExecutionMethod="None"
                  TypeName="IfObject"
                  Value_DisplayArg="0" Value_Type="x:String"
                  Variable="[GetEnvironmentVariable_Value]"
                  Variable_DisplayArg="Get Environment Variable.Value"
                  Variable_Type="x:String"
                  VerboseOutput="False" VerboseOutput_DisplayArg=""
                  m_bTextLinkChange="False">
                  <p:IfObject.IfOption>
                    <p:SequenceActivity DisplayName="Then" sap:VirtualizedContainerService.HintSize="172,81" Name="SequenceActivity">
                      <p:SequenceActivity.Activities>
                        <p:SetEnvironmentVariable Type_Item="{x:Null}" Type_ItemProp="{x:Null}"
                          UserName="{x:Null}" UserName_DisplayArg="{x:Null}"
                          UserName_Item="{x:Null}" UserName_ItemProp="{x:Null}"
                          Value_Item="{x:Null}" Value_ItemProp="{x:Null}"
                          Variable_Item="{x:Null}" Variable_ItemProp="{x:Null}"
                          AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
                          DisplayName="Set Environment Variable"
                          sap:VirtualizedContainerService.HintSize="365,81"
                          MinRequiredVersion="2.10.0.19"
                          Moniker="{NEW-GUID}"
                          Result="[SetEnvironmentVariable_Result]"
                          ResultString="[SetEnvironmentVariable_ResultString]"
                          RunAsCurrentLoggedOnUser="False"
                          ScriptExecutionMethod="ExecuteDebug"
                          Type="Machine" TypeName="SetEnvironmentVariable"
                          Type_DisplayArg="Machine"
                          Value="0" Value_DisplayArg="0"
                          Variable="[Program]" Variable_DisplayArg="Global Variables.Program"
                          m_bTextLinkChange="False" />
                      </p:SequenceActivity.Activities>
                      <p:SequenceActivity.Variables>
                        <Variable x:TypeArguments="x:Double" Name="SetEnvironmentVariable_Result" />
                        <Variable x:TypeArguments="x:String" Name="SetEnvironmentVariable_ResultString" />
                      </p:SequenceActivity.Variables>
                    </p:SequenceActivity>
                  </p:IfObject.IfOption>
                  <p:IfObject.Value>
                    <InArgument x:TypeArguments="x:Object">
                      <p:ObjectLiteral Value="0" />
                    </InArgument>
                  </p:IfObject.Value>
                </p:IfObject>
              </p:SequenceActivity.Activities>
              <p:SequenceActivity.Variables>
                <Variable x:TypeArguments="x:Double" Name="IfObject_Result" />
                <Variable x:TypeArguments="x:String" Name="IfObject_ResultString" />
              </p:SequenceActivity.Variables>
            </p:SequenceActivity>
          </p:IfElse.IfOption>
          <p:IfElse.ElseOption>
            <!-- APP IS NOT INSTALLED: set env var to 1 if not already 1 -->
            <p:SequenceActivity DisplayName="Else" sap:VirtualizedContainerService.HintSize="172,81" Name="SequenceActivity">
              <p:SequenceActivity.Activities>
                <p:IfObject CaseSensitive_Item="{x:Null}" CaseSensitive_ItemProp="{x:Null}"
                  Condition_Item="{x:Null}" Condition_ItemProp="{x:Null}"
                  Value_Item="{x:Null}" Value_ItemProp="{x:Null}"
                  Variable_Item="{x:Null}" Variable_ItemProp="{x:Null}"
                  VerboseOutput_Item="{x:Null}" VerboseOutput_ItemProp="{x:Null}"
                  AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
                  CaseSensitive="False" CaseSensitive_DisplayArg="false"
                  Condition="does not equal" Condition_DisplayArg="does not equal"
                  DisplayName="If"
                  sap:VirtualizedContainerService.HintSize="435,81"
                  MinRequiredVersion="2.19.0.1"
                  Moniker="{NEW-GUID}"
                  Result="[IfObject_Result_1]" ResultString="[IfObject_ResultString_1]"
                  RunAsCurrentLoggedOnUser="False"
                  ScriptExecutionMethod="None"
                  TypeName="IfObject"
                  Value_DisplayArg="1" Value_Type="x:String"
                  Variable="[GetEnvironmentVariable_Value]"
                  Variable_DisplayArg="Get Environment Variable.Value"
                  Variable_Type="x:String"
                  VerboseOutput="False" VerboseOutput_DisplayArg=""
                  m_bTextLinkChange="False">
                  <p:IfObject.IfOption>
                    <p:SequenceActivity DisplayName="Then" sap:VirtualizedContainerService.HintSize="172,81" Name="SequenceActivity">
                      <p:SequenceActivity.Activities>
                        <p:SetEnvironmentVariable Type_Item="{x:Null}" Type_ItemProp="{x:Null}"
                          UserName="{x:Null}" UserName_DisplayArg="{x:Null}"
                          UserName_Item="{x:Null}" UserName_ItemProp="{x:Null}"
                          Value_Item="{x:Null}" Value_ItemProp="{x:Null}"
                          Variable_Item="{x:Null}" Variable_ItemProp="{x:Null}"
                          AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
                          DisplayName="Set Environment Variable"
                          sap:VirtualizedContainerService.HintSize="365,81"
                          MinRequiredVersion="2.10.0.19"
                          Moniker="{NEW-GUID}"
                          Result="[SetEnvironmentVariable_Result_1]"
                          ResultString="[SetEnvironmentVariable_ResultString_1]"
                          RunAsCurrentLoggedOnUser="False"
                          ScriptExecutionMethod="ExecuteDebug"
                          Type="Machine" TypeName="SetEnvironmentVariable"
                          Type_DisplayArg="Machine"
                          Value="1" Value_DisplayArg="1"
                          Variable="[Program]" Variable_DisplayArg="Global Variables.Program"
                          m_bTextLinkChange="False" />
                      </p:SequenceActivity.Activities>
                      <p:SequenceActivity.Variables>
                        <Variable x:TypeArguments="x:Double" Name="SetEnvironmentVariable_Result_1" />
                        <Variable x:TypeArguments="x:String" Name="SetEnvironmentVariable_ResultString_1" />
                      </p:SequenceActivity.Variables>
                    </p:SequenceActivity>
                  </p:IfObject.IfOption>
                  <p:IfObject.Value>
                    <InArgument x:TypeArguments="x:Object">
                      <p:ObjectLiteral Value="1" />
                    </InArgument>
                  </p:IfObject.Value>
                </p:IfObject>
              </p:SequenceActivity.Activities>
              <p:SequenceActivity.Variables>
                <Variable x:TypeArguments="x:Double" Name="IfObject_Result_1" />
                <Variable x:TypeArguments="x:String" Name="IfObject_ResultString_1" />
              </p:SequenceActivity.Variables>
            </p:SequenceActivity>
          </p:IfElse.ElseOption>
          <p:IfElse.Value>
            <InArgument x:TypeArguments="x:Object">
              <p:ObjectLiteral Value="True" />
            </InArgument>
          </p:IfElse.Value>
        </p:IfElse>

        <!-- STEP 4: SwitchObject — verify state matches desired action -->
        <p:SwitchObject AllowDefault_Item="{x:Null}" AllowDefault_ItemProp="{x:Null}"
          Variable_Item="{x:Null}" Variable_ItemProp="{x:Null}"
          AllowDefault="False" AllowDefault_DisplayArg="true"
          AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
          DisplayName="Output for Install vs Uninstall"
          sap:VirtualizedContainerService.HintSize="305,81"
          MinRequiredVersion="2.10.0.19"
          Moniker="{NEW-GUID}"
          Result="[SwitchObject_Result]" ResultString="[SwitchObject_ResultString]"
          RunAsCurrentLoggedOnUser="False"
          ScriptExecutionMethod="None"
          TypeName="SwitchObject"
          Variable="[Task_Check]"
          Variable_DisplayArg="Input Parameters.Uninstall = 0 Install = 1"
          Variable_Type="x:Double"
          m_bTextLinkChange="False">
          <p:SwitchObject.CaseSequence>
            <p:CaseSequenceActivity DisplayName="" sap:VirtualizedContainerService.HintSize="243,238" Name="CaseSequenceActivity">
              <p:CaseSequenceActivity.Activities>
                <!-- Case 0 (Uninstall): if app still installed, stop with error -->
                <p:CaseObject Value_Item="{x:Null}" Value_ItemProp="{x:Null}"
                  AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
                  DisplayName="Case 0"
                  sap:VirtualizedContainerService.HintSize="237,81"
                  MinRequiredVersion="2.10.0.19"
                  Moniker="{NEW-GUID}"
                  Result="[CaseObject_Result]" ResultString="[CaseObject_ResultString]"
                  RunAsCurrentLoggedOnUser="False" RunCase="False"
                  ScriptExecutionMethod="None" TypeName="CaseObject"
                  ValidationError=""
                  Value_DisplayArg="0" Value_Type="x:String"
                  m_bTextLinkChange="False">
                  <p:CaseObject.ThenOption>
                    <p:SequenceActivity DisplayName="Then" sap:VirtualizedContainerService.HintSize="172,81" Name="SequenceActivity">
                      <p:SequenceActivity.Activities>
                        <p:IfObject CaseSensitive_Item="{x:Null}" CaseSensitive_ItemProp="{x:Null}"
                          Condition_Item="{x:Null}" Condition_ItemProp="{x:Null}"
                          Value_Item="{x:Null}" Value_ItemProp="{x:Null}"
                          Variable_Item="{x:Null}" Variable_ItemProp="{x:Null}"
                          VerboseOutput_Item="{x:Null}" VerboseOutput_ItemProp="{x:Null}"
                          AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
                          CaseSensitive="False" CaseSensitive_DisplayArg="false"
                          Condition="equals" Condition_DisplayArg="equals"
                          DisplayName="If"
                          sap:VirtualizedContainerService.HintSize="435,81"
                          MinRequiredVersion="2.19.0.1"
                          Moniker="{NEW-GUID}"
                          Result="[IfObject_Result_2]" ResultString="[IfObject_ResultString_2]"
                          RunAsCurrentLoggedOnUser="False"
                          ScriptExecutionMethod="None" TypeName="IfObject"
                          Value_DisplayArg="True" Value_Type="x:String"
                          Variable="[IsAppInstalled_Conditional]"
                          Variable_DisplayArg="Is Application Installed.Conditional"
                          Variable_Type="x:String"
                          VerboseOutput="False" VerboseOutput_DisplayArg=""
                          m_bTextLinkChange="False">
                          <p:IfObject.IfOption>
                            <p:SequenceActivity DisplayName="Then" sap:VirtualizedContainerService.HintSize="172,81" Name="SequenceActivity">
                              <p:SequenceActivity.Activities>
                                <p:StopPolicy CompletionResult_Item="{x:Null}" CompletionResult_ItemProp="{x:Null}"
                                  StopReason_Item="{x:Null}" StopReason_ItemProp="{x:Null}"
                                  AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
                                  CompletionResult="1" CompletionResult_DisplayArg="1"
                                  DisplayName="Stop Policy"
                                  sap:VirtualizedContainerService.HintSize="183,81"
                                  MinRequiredVersion="2.16.1.1"
                                  Moniker="{NEW-GUID}"
                                  Result="[StopPolicy_Result]" ResultString="[StopPolicy_ResultString]"
                                  RunAsCurrentLoggedOnUser="False"
                                  ScriptExecutionMethod="None"
                                  StopReason="Application Still Installed"
                                  StopReason_DisplayArg="Application Still Installed"
                                  TypeName="StopPolicy"
                                  m_bTextLinkChange="False" />
                              </p:SequenceActivity.Activities>
                              <p:SequenceActivity.Variables>
                                <Variable x:TypeArguments="x:Double" Name="StopPolicy_Result" />
                                <Variable x:TypeArguments="x:String" Name="StopPolicy_ResultString" />
                              </p:SequenceActivity.Variables>
                            </p:SequenceActivity>
                          </p:IfObject.IfOption>
                          <p:IfObject.Value>
                            <InArgument x:TypeArguments="x:Object">
                              <p:ObjectLiteral Value="True" />
                            </InArgument>
                          </p:IfObject.Value>
                        </p:IfObject>
                      </p:SequenceActivity.Activities>
                      <p:SequenceActivity.Variables>
                        <Variable x:TypeArguments="x:Double" Name="IfObject_Result_2" />
                        <Variable x:TypeArguments="x:String" Name="IfObject_ResultString_2" />
                      </p:SequenceActivity.Variables>
                    </p:SequenceActivity>
                  </p:CaseObject.ThenOption>
                  <p:CaseObject.Value>
                    <InArgument x:TypeArguments="x:Object">
                      <p:ObjectLiteral Value="0" />
                    </InArgument>
                  </p:CaseObject.Value>
                </p:CaseObject>
                <!-- Case 1 (Install): if app NOT installed, stop with error -->
                <p:CaseObject Value_Item="{x:Null}" Value_ItemProp="{x:Null}"
                  AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
                  DisplayName="Case 1"
                  sap:VirtualizedContainerService.HintSize="237,81"
                  MinRequiredVersion="2.10.0.19"
                  Moniker="{NEW-GUID}"
                  Result="[CaseObject_Result_1]" ResultString="[CaseObject_ResultString_1]"
                  RunAsCurrentLoggedOnUser="False" RunCase="False"
                  ScriptExecutionMethod="None" TypeName="CaseObject"
                  ValidationError=""
                  Value_DisplayArg="1" Value_Type="x:String"
                  m_bTextLinkChange="False">
                  <p:CaseObject.ThenOption>
                    <p:SequenceActivity DisplayName="Then" sap:VirtualizedContainerService.HintSize="172,81" Name="SequenceActivity">
                      <p:SequenceActivity.Activities>
                        <p:IfObject CaseSensitive_Item="{x:Null}" CaseSensitive_ItemProp="{x:Null}"
                          Condition_Item="{x:Null}" Condition_ItemProp="{x:Null}"
                          Value_Item="{x:Null}" Value_ItemProp="{x:Null}"
                          Variable_Item="{x:Null}" Variable_ItemProp="{x:Null}"
                          VerboseOutput_Item="{x:Null}" VerboseOutput_ItemProp="{x:Null}"
                          AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
                          CaseSensitive="False" CaseSensitive_DisplayArg="false"
                          Condition="does not equal" Condition_DisplayArg="does not equal"
                          DisplayName="If"
                          sap:VirtualizedContainerService.HintSize="435,81"
                          MinRequiredVersion="2.19.0.1"
                          Moniker="{NEW-GUID}"
                          Result="[IfObject_Result_3]" ResultString="[IfObject_ResultString_3]"
                          RunAsCurrentLoggedOnUser="False"
                          ScriptExecutionMethod="None" TypeName="IfObject"
                          Value_DisplayArg="True" Value_Type="x:String"
                          Variable="[IsAppInstalled_Conditional]"
                          Variable_DisplayArg="Is Application Installed.Conditional"
                          Variable_Type="x:String"
                          VerboseOutput="False" VerboseOutput_DisplayArg=""
                          m_bTextLinkChange="False">
                          <p:IfObject.IfOption>
                            <p:SequenceActivity DisplayName="Then" sap:VirtualizedContainerService.HintSize="172,81" Name="SequenceActivity">
                              <p:SequenceActivity.Activities>
                                <p:StopPolicy CompletionResult_Item="{x:Null}" CompletionResult_ItemProp="{x:Null}"
                                  StopReason_Item="{x:Null}" StopReason_ItemProp="{x:Null}"
                                  AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
                                  CompletionResult="1" CompletionResult_DisplayArg="1"
                                  DisplayName="Stop Policy"
                                  sap:VirtualizedContainerService.HintSize="183,81"
                                  MinRequiredVersion="2.16.1.1"
                                  Moniker="{NEW-GUID}"
                                  Result="[StopPolicy_Result_1]" ResultString="[StopPolicy_ResultString_1]"
                                  RunAsCurrentLoggedOnUser="False"
                                  ScriptExecutionMethod="None"
                                  StopReason="Application Not Installed"
                                  StopReason_DisplayArg="Application Not Installed"
                                  TypeName="StopPolicy"
                                  m_bTextLinkChange="False" />
                              </p:SequenceActivity.Activities>
                              <p:SequenceActivity.Variables>
                                <Variable x:TypeArguments="x:Double" Name="StopPolicy_Result_1" />
                                <Variable x:TypeArguments="x:String" Name="StopPolicy_ResultString_1" />
                              </p:SequenceActivity.Variables>
                            </p:SequenceActivity>
                          </p:IfObject.IfOption>
                          <p:IfObject.Value>
                            <InArgument x:TypeArguments="x:Object">
                              <p:ObjectLiteral Value="True" />
                            </InArgument>
                          </p:IfObject.Value>
                        </p:IfObject>
                      </p:SequenceActivity.Activities>
                      <p:SequenceActivity.Variables>
                        <Variable x:TypeArguments="x:Double" Name="IfObject_Result_3" />
                        <Variable x:TypeArguments="x:String" Name="IfObject_ResultString_3" />
                      </p:SequenceActivity.Variables>
                    </p:SequenceActivity>
                  </p:CaseObject.ThenOption>
                  <p:CaseObject.Value>
                    <InArgument x:TypeArguments="x:Object">
                      <p:ObjectLiteral Value="1" />
                    </InArgument>
                  </p:CaseObject.Value>
                </p:CaseObject>
              </p:CaseSequenceActivity.Activities>
              <p:CaseSequenceActivity.Variables>
                <Variable x:TypeArguments="x:String" Name="CaseObject_ResultString" />
                <Variable x:TypeArguments="x:Double" Name="CaseObject_Result" />
                <Variable x:TypeArguments="x:String" Name="CaseObject_ResultString_1" />
                <Variable x:TypeArguments="x:Double" Name="CaseObject_Result_1" />
              </p:CaseSequenceActivity.Variables>
            </p:CaseSequenceActivity>
          </p:SwitchObject.CaseSequence>
          <p:SwitchObject.DefaultOption>
            <p:SequenceActivity DisplayName="Default" sap:VirtualizedContainerService.HintSize="172,81" Name="SequenceActivity">
              <p:SequenceActivity.Activities>
                <sco:Collection x:TypeArguments="Activity" />
              </p:SequenceActivity.Activities>
              <p:SequenceActivity.Variables>
                <sco:Collection x:TypeArguments="Variable" />
              </p:SequenceActivity.Variables>
            </p:SequenceActivity>
          </p:SwitchObject.DefaultOption>
        </p:SwitchObject>

        <!-- STEP 5: Final Log -->
        <p:Log Message_Item="{x:Null}" Message_ItemProp="{x:Null}"
          AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
          DisplayName="Log"
          sap:VirtualizedContainerService.HintSize="305,88"
          LogMessage="[Log_LogMessage]"
          Message="Application Installed"
          Message_DisplayArg="Application Installed"
          MinRequiredVersion="2.10.0.19"
          Moniker="{NEW-GUID}"
          Result="[Log_Result]"
          ResultString="[Log_ResultString]"
          RunAsCurrentLoggedOnUser="False"
          ScriptExecutionMethod="ExecuteDebug"
          TypeName="Log"
          m_bTextLinkChange="False" />

      </p:PolicySequence.Activities>
      <p:PolicySequence.Variables>
        <Variable x:TypeArguments="x:Boolean" Name="IsAppInstalled_Conditional" />
        <Variable x:TypeArguments="x:Double" Name="IsAppInstalled_Result" />
        <Variable x:TypeArguments="x:String" Name="IsAppInstalled_ResultString" />
        <Variable x:TypeArguments="x:String" Name="GetEnvironmentVariable_Value" />
        <Variable x:TypeArguments="x:Double" Name="GetEnvironmentVariable_Result" />
        <Variable x:TypeArguments="x:String" Name="GetEnvironmentVariable_ResultString" />
        <Variable x:TypeArguments="x:Double" Name="IfElse_Result" />
        <Variable x:TypeArguments="x:String" Name="IfElse_ResultString" />
        <Variable x:TypeArguments="x:Double" Name="SwitchObject_Result" />
        <Variable x:TypeArguments="x:String" Name="SwitchObject_ResultString" />
        <Variable x:TypeArguments="x:String" Name="Log_LogMessage" />
        <Variable x:TypeArguments="x:Double" Name="Log_Result" />
        <Variable x:TypeArguments="x:String" Name="Log_ResultString" />
        <Variable x:TypeArguments="x:Double" Default="1" Name="Task_Check" />
        <Variable x:TypeArguments="x:String" Default="APPLICATION_DISPLAY_NAME" Name="Program" />
      </p:PolicySequence.Variables>
    </p:PolicySequence>
  </Activity>
</Policy>
```

**Placeholders:**
- `{NEW-GUID}` — unique GUID per element
- `APPLICATION_DISPLAY_NAME` — the name as it appears in Add/Remove Programs
- `BASE64_ENCODED_DESCRIPTION` — base64-encode the plain-text description

---

### Template 3: Deployment

Full deployment: override switch, install check, download, install, verify, set env var. Uses generic placeholders for URLs and paths.

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<Policy ID="{NEW-GUID}" Name="GenericApp Deployment - Full (2025)"
  Description="BASE64_ENCODED_DESCRIPTION"
  Version="2.98.0.2" MinRequiredVersion="2.98.0.2" RemoteCategory="0"
  ExecutionType="CurrentLoggedOnUser" MinimumPSVersionRequired="3.0">
  <Object ID="{NEW-GUID}" Type="{B6FA6D8B-EEAA-47A6-8463-7F9A4F5BBB6E}"
    Data="&lt;xml&gt;&lt;Parameters&gt;&lt;Parameter ParameterName=&quot;Override&quot; Label=&quot;Override - B = Bypass / U = Uninstall / R = Reinstall&quot; ParameterType=&quot;string&quot; Value=&quot;&quot; /&gt;&lt;/Parameters&gt;&lt;GlobalVariables&gt;&lt;Parameter ParameterName=&quot;Software&quot; Label=&quot;Software&quot; ParameterType=&quot;string&quot; Value=&quot;GenericApp&quot; /&gt;&lt;Parameter ParameterName=&quot;Download_URL&quot; Label=&quot;Download URL&quot; ParameterType=&quot;string&quot; Value=&quot;&quot; /&gt;&lt;Parameter ParameterName=&quot;Software_Folder&quot; Label=&quot;Software Folder&quot; ParameterType=&quot;string&quot; Value=&quot;C:\DatoSoftware&quot; /&gt;&lt;Parameter ParameterName=&quot;Software_File&quot; Label=&quot;Software File&quot; ParameterType=&quot;string&quot; Value=&quot;C:\DatoSoftware\installer.exe&quot; /&gt;&lt;Parameter ParameterName=&quot;Argument_List&quot; Label=&quot;Argument List&quot; ParameterType=&quot;string&quot; Value=&quot;/S&quot; /&gt;&lt;Parameter ParameterName=&quot;Installed_Application&quot; Label=&quot;Installed Application&quot; ParameterType=&quot;string&quot; Value=&quot;APPLICATION_DISPLAY_NAME&quot; /&gt;&lt;Parameter ParameterName=&quot;Remove&quot; Label=&quot;Remove&quot; ParameterType=&quot;number&quot; Value=&quot;0&quot; /&gt;&lt;Parameter ParameterName=&quot;Install&quot; Label=&quot;Install&quot; ParameterType=&quot;number&quot; Value=&quot;0&quot; /&gt;&lt;/GlobalVariables&gt;&lt;/xml&gt;" />
  <LinkManager xmlns:i="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://schemas.datacontract.org/2004/07/PolicyExecutor">
    <hashset xmlns:d2p1="http://schemas.datacontract.org/2004/07/System" />
  </LinkManager>
  <Diagnostics OriginalVersion="2.98.2.2" />
  <Activity mc:Ignorable="sads sap" x:Class="Policy Builder"
    xmlns="http://schemas.microsoft.com/netfx/2009/xaml/activities"
    xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
    xmlns:mva="clr-namespace:Microsoft.VisualBasic.Activities;assembly=System.Activities"
    xmlns:p="clr-namespace:PolicyExecutor;assembly=PolicyExecutionEngine"
    xmlns:sads="http://schemas.microsoft.com/netfx/2010/xaml/activities/debugger"
    xmlns:sap="http://schemas.microsoft.com/netfx/2009/xaml/activities/presentation"
    xmlns:scg="clr-namespace:System.Collections.Generic;assembly=mscorlib"
    xmlns:sco="clr-namespace:System.Collections.ObjectModel;assembly=mscorlib"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <x:Members>
      <x:Property Name="PolicyGUID" Type="InArgument(x:String)" />
    </x:Members>
    <sap:VirtualizedContainerService.HintSize>515,1496</sap:VirtualizedContainerService.HintSize>
    <mva:VisualBasic.Settings>Assembly references and imported namespaces serialized as XML namespaces</mva:VisualBasic.Settings>
    <p:PolicySequence DisplayName="Policy Builder" sap:VirtualizedContainerService.HintSize="515,1496"
      MinRequiredVersion="2.98.0.2"
      mva:VisualBasic.Settings="Assembly references and imported namespaces serialized as XML namespaces">
      <p:PolicySequence.Activities>

        <!-- STEP 1: SwitchObject on Override parameter (B=Bypass, U=Uninstall, R=Reinstall) -->
        <p:SwitchObject AllowDefault_Item="{x:Null}" AllowDefault_ItemProp="{x:Null}"
          Variable_Item="{x:Null}" Variable_ItemProp="{x:Null}"
          AllowDefault="True" AllowDefault_DisplayArg="true"
          AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
          DisplayName="Override"
          sap:VirtualizedContainerService.HintSize="479,81"
          MinRequiredVersion="2.10.0.19"
          Moniker="{NEW-GUID}"
          Result="[SwitchObject_Result]" ResultString="[SwitchObject_ResultString]"
          RunAsCurrentLoggedOnUser="False"
          ScriptExecutionMethod="None"
          TypeName="SwitchObject"
          Variable="[Override]"
          Variable_DisplayArg="Input Parameters.Override - B = Bypass / U = Uninstall / R = Reinstall"
          Variable_Type="x:String"
          m_bTextLinkChange="False">
          <p:SwitchObject.CaseSequence>
            <p:CaseSequenceActivity DisplayName="" sap:VirtualizedContainerService.HintSize="243,238" Name="CaseSequenceActivity">
              <p:CaseSequenceActivity.Activities>
                <!-- Case B: Bypass — set Install=1 to skip install check -->
                <p:CaseObject Value_Item="{x:Null}" Value_ItemProp="{x:Null}"
                  AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
                  DisplayName="Bypass" sap:VirtualizedContainerService.HintSize="237,81"
                  MinRequiredVersion="2.10.0.19" Moniker="{NEW-GUID}"
                  Result="[CaseObject_Result]" ResultString="[CaseObject_ResultString]"
                  RunAsCurrentLoggedOnUser="False" RunCase="False"
                  ScriptExecutionMethod="None" TypeName="CaseObject" ValidationError=""
                  Value_DisplayArg="B" Value_Type="x:String" m_bTextLinkChange="False">
                  <p:CaseObject.ThenOption>
                    <p:SequenceActivity DisplayName="Then" sap:VirtualizedContainerService.HintSize="172,81" Name="SequenceActivity">
                      <p:SequenceActivity.Activities>
                        <p:SetEnvironmentVariable Type_Item="{x:Null}" Type_ItemProp="{x:Null}"
                          UserName="{x:Null}" UserName_DisplayArg="{x:Null}" UserName_Item="{x:Null}" UserName_ItemProp="{x:Null}"
                          Value_Item="{x:Null}" Value_ItemProp="{x:Null}" Variable_Item="{x:Null}" Variable_ItemProp="{x:Null}"
                          AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
                          DisplayName="Set Environment Variable" sap:VirtualizedContainerService.HintSize="365,81"
                          MinRequiredVersion="2.10.0.19" Moniker="{NEW-GUID}"
                          Result="[SetEnvironmentVariable_Result]" ResultString="[SetEnvironmentVariable_ResultString]"
                          RunAsCurrentLoggedOnUser="False" ScriptExecutionMethod="ExecuteDebug"
                          Type="Machine" TypeName="SetEnvironmentVariable" Type_DisplayArg="Machine"
                          Value="Bypass" Value_DisplayArg="Bypass"
                          Variable="[Software]" Variable_DisplayArg="Global Variables.Software"
                          m_bTextLinkChange="False" />
                      </p:SequenceActivity.Activities>
                      <p:SequenceActivity.Variables>
                        <Variable x:TypeArguments="x:Double" Name="SetEnvironmentVariable_Result" />
                        <Variable x:TypeArguments="x:String" Name="SetEnvironmentVariable_ResultString" />
                      </p:SequenceActivity.Variables>
                    </p:SequenceActivity>
                  </p:CaseObject.ThenOption>
                  <p:CaseObject.Value><InArgument x:TypeArguments="x:Object"><p:ObjectLiteral Value="B" /></InArgument></p:CaseObject.Value>
                </p:CaseObject>
                <!-- Case U: Uninstall — set Remove=1 -->
                <p:CaseObject Value_Item="{x:Null}" Value_ItemProp="{x:Null}"
                  AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
                  DisplayName="Uninstall" sap:VirtualizedContainerService.HintSize="237,81"
                  MinRequiredVersion="2.10.0.19" Moniker="{NEW-GUID}"
                  Result="[CaseObject_Result_1]" ResultString="[CaseObject_ResultString_1]"
                  RunAsCurrentLoggedOnUser="False" RunCase="False"
                  ScriptExecutionMethod="None" TypeName="CaseObject" ValidationError=""
                  Value_DisplayArg="U" Value_Type="x:String" m_bTextLinkChange="False">
                  <p:CaseObject.ThenOption>
                    <p:SequenceActivity DisplayName="Then" sap:VirtualizedContainerService.HintSize="172,81" Name="SequenceActivity">
                      <p:SequenceActivity.Activities>
                        <!-- Set Remove=1 via RunPowerShellScript or direct variable assignment -->
                        <p:Log Message_Item="{x:Null}" Message_ItemProp="{x:Null}"
                          AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
                          DisplayName="Log" sap:VirtualizedContainerService.HintSize="172,81"
                          LogMessage="[Log_LogMessage]"
                          Message="Uninstall Override Activated"
                          Message_DisplayArg="Uninstall Override Activated"
                          Moniker="{NEW-GUID}"
                          Result="[Log_Result]" ResultString="[Log_ResultString]"
                          RunAsCurrentLoggedOnUser="False" ScriptExecutionMethod="ExecuteDebug"
                          TypeName="Log" m_bTextLinkChange="False" />
                      </p:SequenceActivity.Activities>
                      <p:SequenceActivity.Variables>
                        <Variable x:TypeArguments="x:String" Name="Log_LogMessage" />
                        <Variable x:TypeArguments="x:Double" Name="Log_Result" />
                        <Variable x:TypeArguments="x:String" Name="Log_ResultString" />
                      </p:SequenceActivity.Variables>
                    </p:SequenceActivity>
                  </p:CaseObject.ThenOption>
                  <p:CaseObject.Value><InArgument x:TypeArguments="x:Object"><p:ObjectLiteral Value="U" /></InArgument></p:CaseObject.Value>
                </p:CaseObject>
                <!-- Case R: Reinstall — set Remove=1 and Install=1 -->
                <p:CaseObject Value_Item="{x:Null}" Value_ItemProp="{x:Null}"
                  AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
                  DisplayName="Reinstall" sap:VirtualizedContainerService.HintSize="237,81"
                  MinRequiredVersion="2.10.0.19" Moniker="{NEW-GUID}"
                  Result="[CaseObject_Result_2]" ResultString="[CaseObject_ResultString_2]"
                  RunAsCurrentLoggedOnUser="False" RunCase="False"
                  ScriptExecutionMethod="None" TypeName="CaseObject" ValidationError=""
                  Value_DisplayArg="R" Value_Type="x:String" m_bTextLinkChange="False">
                  <p:CaseObject.ThenOption>
                    <p:SequenceActivity DisplayName="Then" sap:VirtualizedContainerService.HintSize="172,81" Name="SequenceActivity">
                      <p:SequenceActivity.Activities>
                        <p:Log Message_Item="{x:Null}" Message_ItemProp="{x:Null}"
                          AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
                          DisplayName="Log" sap:VirtualizedContainerService.HintSize="172,81"
                          LogMessage="[Log_LogMessage_1]"
                          Message="Reinstall Override Activated"
                          Message_DisplayArg="Reinstall Override Activated"
                          Moniker="{NEW-GUID}"
                          Result="[Log_Result_1]" ResultString="[Log_ResultString_1]"
                          RunAsCurrentLoggedOnUser="False" ScriptExecutionMethod="ExecuteDebug"
                          TypeName="Log" m_bTextLinkChange="False" />
                      </p:SequenceActivity.Activities>
                      <p:SequenceActivity.Variables>
                        <Variable x:TypeArguments="x:String" Name="Log_LogMessage_1" />
                        <Variable x:TypeArguments="x:Double" Name="Log_Result_1" />
                        <Variable x:TypeArguments="x:String" Name="Log_ResultString_1" />
                      </p:SequenceActivity.Variables>
                    </p:SequenceActivity>
                  </p:CaseObject.ThenOption>
                  <p:CaseObject.Value><InArgument x:TypeArguments="x:Object"><p:ObjectLiteral Value="R" /></InArgument></p:CaseObject.Value>
                </p:CaseObject>
              </p:CaseSequenceActivity.Activities>
              <p:CaseSequenceActivity.Variables>
                <Variable x:TypeArguments="x:String" Name="CaseObject_ResultString" />
                <Variable x:TypeArguments="x:Double" Name="CaseObject_Result" />
                <Variable x:TypeArguments="x:String" Name="CaseObject_ResultString_1" />
                <Variable x:TypeArguments="x:Double" Name="CaseObject_Result_1" />
                <Variable x:TypeArguments="x:String" Name="CaseObject_ResultString_2" />
                <Variable x:TypeArguments="x:Double" Name="CaseObject_Result_2" />
              </p:CaseSequenceActivity.Variables>
            </p:CaseSequenceActivity>
          </p:SwitchObject.CaseSequence>
          <p:SwitchObject.DefaultOption>
            <p:SequenceActivity DisplayName="Default" sap:VirtualizedContainerService.HintSize="172,81" Name="SequenceActivity">
              <p:SequenceActivity.Activities>
                <sco:Collection x:TypeArguments="Activity" />
              </p:SequenceActivity.Activities>
              <p:SequenceActivity.Variables>
                <sco:Collection x:TypeArguments="Variable" />
              </p:SequenceActivity.Variables>
            </p:SequenceActivity>
          </p:SwitchObject.DefaultOption>
        </p:SwitchObject>

        <!-- STEP 2: IsAppInstalled check -->
        <p:IsAppInstalled ApplicationName_Item="{x:Null}" ApplicationName_ItemProp="{x:Null}"
          ApplicationName="[Installed_Application]"
          ApplicationName_DisplayArg="Global Variables.Installed Application"
          AssemblyName="PolicyExecutionEngine, Version=2.96.1.1, Culture=neutral, PublicKeyToken=null"
          Conditional="[IsAppInstalled_Conditional]"
          DisplayName="Is Application Installed"
          sap:VirtualizedContainerService.HintSize="479,81"
          MinRequiredVersion="2.10.0.19"
          Moniker="{NEW-GUID}"
          Result="[IsAppInstalled_Result]"
          ResultString="[IsAppInstalled_ResultString]"
          RunAsCurrentLoggedOnUser="False"
          ScriptExecutionMethod="None"
          TypeName="IsAppInstalled"
          m_bTextLinkChange="False" />

        <!-- STEP 3: IfElse on IsAppInstalled result -->
        <p:IfElse CaseSensitive_Item="{x:Null}" CaseSensitive_ItemProp="{x:Null}"
          Condition_Item="{x:Null}" Condition_ItemProp="{x:Null}"
          Value_Item="{x:Null}" Value_ItemProp="{x:Null}"
          Variable_Item="{x:Null}" Variable_ItemProp="{x:Null}"
          AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
          CaseSensitive="False" CaseSensitive_DisplayArg="false"
          Condition="does not equal" Condition_DisplayArg="does not equal"
          DisplayName="Install"
          sap:VirtualizedContainerService.HintSize="479,81"
          MinRequiredVersion="2.19.0.1"
          Moniker="{NEW-GUID}"
          Result="[IfElse_Result]" ResultString="[IfElse_ResultString]"
          RunAsCurrentLoggedOnUser="False"
          ScriptExecutionMethod="None"
          TypeName="IfElse"
          Value_DisplayArg="True" Value_Type="x:String"
          Variable="[IsAppInstalled_Conditional]"
          Variable_DisplayArg="Is Application Installed.Conditional"
          Variable_Type="x:String"
          m_bTextLinkChange="False">
          <p:IfElse.IfOption>
            <!-- APP IS NOT INSTALLED: download, install, verify -->
            <p:SequenceActivity DisplayName="Then" sap:VirtualizedContainerService.HintSize="479,900" Name="SequenceActivity">
              <p:SequenceActivity.Activities>
                <!-- Download installer -->
                <p:RunPowerShellScript genArgEvent="{x:Null}"
                  AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
                  DisplayName="Download Installer"
                  sap:VirtualizedContainerService.HintSize="454,348"
                  Moniker="{NEW-GUID}"
                  OutPut_64="[RunPowerShellScript_OutPut_64]"
                  Result="[RunPowerShellScript_Result]"
                  ResultString="[RunPowerShellScript_ResultString]"
                  Results_x64="[RunPowerShellScript_Results_x64]"
                  RunAsCurrentLoggedOnUser="False"
                  ScriptExecutionMethod="ExecuteDebug"
                  TypeName="RunPowerShellScript"
                  m_bTextLinkChange="False"
                  script="BASE64_DOWNLOAD_SCRIPT">
                  <p:RunPowerShellScript.InArgs><scg:Dictionary x:TypeArguments="x:String, p:InArg" /></p:RunPowerShellScript.InArgs>
                  <p:RunPowerShellScript.OutArgs><scg:Dictionary x:TypeArguments="x:String, p:OutArg" /></p:RunPowerShellScript.OutArgs>
                </p:RunPowerShellScript>
                <!-- Run silent install -->
                <p:RunPowerShellScript genArgEvent="{x:Null}"
                  AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
                  DisplayName="Install Software"
                  sap:VirtualizedContainerService.HintSize="454,348"
                  Moniker="{NEW-GUID}"
                  OutPut_64="[RunPowerShellScript_OutPut_64_1]"
                  Result="[RunPowerShellScript_Result_1]"
                  ResultString="[RunPowerShellScript_ResultString_1]"
                  Results_x64="[RunPowerShellScript_Results_x64_1]"
                  RunAsCurrentLoggedOnUser="False"
                  ScriptExecutionMethod="ExecuteDebug"
                  TypeName="RunPowerShellScript"
                  m_bTextLinkChange="False"
                  script="BASE64_INSTALL_SCRIPT">
                  <p:RunPowerShellScript.InArgs><scg:Dictionary x:TypeArguments="x:String, p:InArg" /></p:RunPowerShellScript.InArgs>
                  <p:RunPowerShellScript.OutArgs><scg:Dictionary x:TypeArguments="x:String, p:OutArg" /></p:RunPowerShellScript.OutArgs>
                </p:RunPowerShellScript>
                <!-- Verify installation -->
                <p:RunPowerShellScript genArgEvent="{x:Null}"
                  AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
                  DisplayName="Verify Installation"
                  sap:VirtualizedContainerService.HintSize="454,348"
                  Moniker="{NEW-GUID}"
                  OutPut_64="[RunPowerShellScript_OutPut_64_2]"
                  Result="[RunPowerShellScript_Result_2]"
                  ResultString="[RunPowerShellScript_ResultString_2]"
                  Results_x64="[RunPowerShellScript_Results_x64_2]"
                  RunAsCurrentLoggedOnUser="False"
                  ScriptExecutionMethod="ExecuteDebug"
                  TypeName="RunPowerShellScript"
                  m_bTextLinkChange="False"
                  script="BASE64_VERIFY_SCRIPT">
                  <p:RunPowerShellScript.InArgs><scg:Dictionary x:TypeArguments="x:String, p:InArg" /></p:RunPowerShellScript.InArgs>
                  <p:RunPowerShellScript.OutArgs><scg:Dictionary x:TypeArguments="x:String, p:OutArg" /></p:RunPowerShellScript.OutArgs>
                </p:RunPowerShellScript>
                <!-- Mark as deployed -->
                <p:SetEnvironmentVariable Type_Item="{x:Null}" Type_ItemProp="{x:Null}"
                  UserName="{x:Null}" UserName_DisplayArg="{x:Null}" UserName_Item="{x:Null}" UserName_ItemProp="{x:Null}"
                  Value_Item="{x:Null}" Value_ItemProp="{x:Null}" Variable_Item="{x:Null}" Variable_ItemProp="{x:Null}"
                  AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
                  DisplayName="Set Environment Variable" sap:VirtualizedContainerService.HintSize="365,81"
                  MinRequiredVersion="2.10.0.19" Moniker="{NEW-GUID}"
                  Result="[SetEnvironmentVariable_Result_1]" ResultString="[SetEnvironmentVariable_ResultString_1]"
                  RunAsCurrentLoggedOnUser="False" ScriptExecutionMethod="ExecuteDebug"
                  Type="Machine" TypeName="SetEnvironmentVariable" Type_DisplayArg="Machine"
                  Value="Installed" Value_DisplayArg="Installed"
                  Variable="[Software]" Variable_DisplayArg="Global Variables.Software"
                  m_bTextLinkChange="False" />
              </p:SequenceActivity.Activities>
              <p:SequenceActivity.Variables>
                <Variable x:TypeArguments="x:String" Name="RunPowerShellScript_OutPut_64" />
                <Variable x:TypeArguments="x:String" Name="RunPowerShellScript_ResultString" />
                <Variable x:TypeArguments="scg:IEnumerable(x:Object)" Name="RunPowerShellScript_Results_x64" />
                <Variable x:TypeArguments="x:Double" Name="RunPowerShellScript_Result" />
                <Variable x:TypeArguments="x:String" Name="RunPowerShellScript_OutPut_64_1" />
                <Variable x:TypeArguments="x:String" Name="RunPowerShellScript_ResultString_1" />
                <Variable x:TypeArguments="scg:IEnumerable(x:Object)" Name="RunPowerShellScript_Results_x64_1" />
                <Variable x:TypeArguments="x:Double" Name="RunPowerShellScript_Result_1" />
                <Variable x:TypeArguments="x:String" Name="RunPowerShellScript_OutPut_64_2" />
                <Variable x:TypeArguments="x:String" Name="RunPowerShellScript_ResultString_2" />
                <Variable x:TypeArguments="scg:IEnumerable(x:Object)" Name="RunPowerShellScript_Results_x64_2" />
                <Variable x:TypeArguments="x:Double" Name="RunPowerShellScript_Result_2" />
                <Variable x:TypeArguments="x:Double" Name="SetEnvironmentVariable_Result_1" />
                <Variable x:TypeArguments="x:String" Name="SetEnvironmentVariable_ResultString_1" />
              </p:SequenceActivity.Variables>
            </p:SequenceActivity>
          </p:IfElse.IfOption>
          <p:IfElse.ElseOption>
            <!-- APP IS ALREADY INSTALLED -->
            <p:SequenceActivity DisplayName="Else" sap:VirtualizedContainerService.HintSize="172,81" Name="SequenceActivity">
              <p:SequenceActivity.Activities>
                <p:Log Message_Item="{x:Null}" Message_ItemProp="{x:Null}"
                  AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
                  DisplayName="Log" sap:VirtualizedContainerService.HintSize="172,81"
                  LogMessage="[Log_LogMessage_2]"
                  Message="Application Already Installed"
                  Message_DisplayArg="Application Already Installed"
                  Moniker="{NEW-GUID}"
                  Result="[Log_Result_2]" ResultString="[Log_ResultString_2]"
                  RunAsCurrentLoggedOnUser="False" ScriptExecutionMethod="ExecuteDebug"
                  TypeName="Log" m_bTextLinkChange="False" />
              </p:SequenceActivity.Activities>
              <p:SequenceActivity.Variables>
                <Variable x:TypeArguments="x:String" Name="Log_LogMessage_2" />
                <Variable x:TypeArguments="x:Double" Name="Log_Result_2" />
                <Variable x:TypeArguments="x:String" Name="Log_ResultString_2" />
              </p:SequenceActivity.Variables>
            </p:SequenceActivity>
          </p:IfElse.ElseOption>
          <p:IfElse.Value>
            <InArgument x:TypeArguments="x:Object">
              <p:ObjectLiteral Value="True" />
            </InArgument>
          </p:IfElse.Value>
        </p:IfElse>

      </p:PolicySequence.Activities>
      <p:PolicySequence.Variables>
        <Variable x:TypeArguments="x:Boolean" Name="IsAppInstalled_Conditional" />
        <Variable x:TypeArguments="x:Double" Name="IsAppInstalled_Result" />
        <Variable x:TypeArguments="x:String" Name="IsAppInstalled_ResultString" />
        <Variable x:TypeArguments="x:Double" Name="IfElse_Result" />
        <Variable x:TypeArguments="x:String" Name="IfElse_ResultString" />
        <Variable x:TypeArguments="x:Double" Name="SwitchObject_Result" />
        <Variable x:TypeArguments="x:String" Name="SwitchObject_ResultString" />
        <Variable x:TypeArguments="x:String" Default="" Name="Override" />
        <Variable x:TypeArguments="x:String" Default="GenericApp" Name="Software" />
        <Variable x:TypeArguments="x:String" Default="" Name="Download_URL" />
        <Variable x:TypeArguments="x:String" Default="C:\DatoSoftware" Name="Software_Folder" />
        <Variable x:TypeArguments="x:String" Default="C:\DatoSoftware\installer.exe" Name="Software_File" />
        <Variable x:TypeArguments="x:String" Default="/S" Name="Argument_List" />
        <Variable x:TypeArguments="x:String" Default="APPLICATION_DISPLAY_NAME" Name="Installed_Application" />
        <Variable x:TypeArguments="x:Double" Default="0" Name="Remove" />
        <Variable x:TypeArguments="x:Double" Default="0" Name="Install" />
      </p:PolicySequence.Variables>
    </p:PolicySequence>
  </Activity>
</Policy>
```

**Placeholders:**
- `{NEW-GUID}` — unique GUID per element
- `APPLICATION_DISPLAY_NAME` — name as it appears in Add/Remove Programs
- `BASE64_ENCODED_DESCRIPTION` — base64-encode the plain-text description
- `BASE64_DOWNLOAD_SCRIPT` / `BASE64_INSTALL_SCRIPT` / `BASE64_VERIFY_SCRIPT` — PowerShell scripts encoded as base64 from UTF-16LE bytes

---

### Template 4: ClientTools InputPrompt

Client-facing tool: display a GUI message to the logged-in user with parameters for title, body, and timeout.

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<Policy ID="{NEW-GUID}" Name="POLICY_NAME" Description="BASE64_ENCODED_DESCRIPTION"
  Version="2.50.0.0" MinRequiredVersion="2.50.0.0" RemoteCategory="0"
  ExecutionType="Local" MinimumPSVersionRequired="3.0">
  <Object ID="{NEW-GUID}" Type="{B6FA6D8B-EEAA-47A6-8463-7F9A4F5BBB6E}"
    Data="&lt;xml&gt;&lt;Parameters&gt;&lt;Parameter ParameterName=&quot;Title&quot; Label=&quot;Title&quot; ParameterType=&quot;string&quot; Value=&quot;DEFAULT_TITLE&quot; /&gt;&lt;Parameter ParameterName=&quot;Message&quot; Label=&quot;Message&quot; ParameterType=&quot;string&quot; Value=&quot;DEFAULT_MESSAGE&quot; /&gt;&lt;Parameter ParameterName=&quot;TimeOut&quot; Label=&quot;Time Out in Seconds&quot; ParameterType=&quot;number&quot; Value=&quot;3600&quot; /&gt;&lt;/Parameters&gt;&lt;/xml&gt;" />
  <LinkManager xmlns:i="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://schemas.datacontract.org/2004/07/PolicyExecutor">
    <hashset xmlns:d2p1="http://schemas.datacontract.org/2004/07/System" />
  </LinkManager>
  <Diagnostics OriginalVersion="2.98.2.2" />
  <Activity mc:Ignorable="sads sap" x:Class="Policy Builder"
    xmlns="http://schemas.microsoft.com/netfx/2009/xaml/activities"
    xmlns:acb="clr-namespace:AutomationManager.Common.Branding;assembly=AutomationManager.Common"
    xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
    xmlns:mva="clr-namespace:Microsoft.VisualBasic.Activities;assembly=System.Activities"
    xmlns:p="clr-namespace:PolicyExecutor;assembly=PolicyExecutionEngine"
    xmlns:sads="http://schemas.microsoft.com/netfx/2010/xaml/activities/debugger"
    xmlns:sap="http://schemas.microsoft.com/netfx/2009/xaml/activities/presentation"
    xmlns:scg="clr-namespace:System.Collections.Generic;assembly=mscorlib"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <x:Members>
      <x:Property Name="PolicyGUID" Type="InArgument(x:String)" />
    </x:Members>
    <sap:VirtualizedContainerService.HintSize>521,605</sap:VirtualizedContainerService.HintSize>
    <mva:VisualBasic.Settings>Assembly references and imported namespaces serialized as XML namespaces</mva:VisualBasic.Settings>
    <p:PolicySequence DisplayName="Policy Builder" sap:VirtualizedContainerService.HintSize="521,605"
      MinRequiredVersion="2.50.0.0"
      mva:VisualBasic.Settings="Assembly references and imported namespaces serialized as XML namespaces">
      <p:PolicySequence.Activities>

        <p:InputPrompt
          AlwaysForeground_Item="{x:Null}" AlwaysForeground_ItemProp="{x:Null}"
          Body_Item="{x:Null}" Body_ItemProp="{x:Null}"
          Buttons="{x:Null}" Buttons_DisplayArg="{x:Null}"
          Buttons_Item="{x:Null}" Buttons_ItemProp="{x:Null}"
          Inputs="{x:Null}" Inputs_DisplayArg="{x:Null}"
          Inputs_Item="{x:Null}" Inputs_ItemProp="{x:Null}"
          Timeout_Item="{x:Null}" Timeout_ItemProp="{x:Null}"
          Title_Item="{x:Null}" Title_ItemProp="{x:Null}"
          AlwaysForeground="True" AlwaysForeground_DisplayArg="true"
          AssemblyName="PolicyExecutionEngine, Version=2.96.1.1, Culture=neutral, PublicKeyToken=null"
          Body="[Message]" BodyXaml=""
          Body_DisplayArg="Input Parameters.Message"
          BrandingJson="{}{&quot;ImageBase64&quot;:null,&quot;ImageName&quot;:&quot;&quot;,&quot;FontName&quot;:&quot;Open Sans&quot;,&quot;MainColor&quot;:&quot;#FF363636&quot;,&quot;AccentColor&quot;:&quot;#FF2E96B9&quot;,&quot;ButtonTextColor&quot;:&quot;#FFFFFFFF&quot;}"
          ButtonSelected="[InputPrompt_ButtonSelected]"
          ClickResult="[InputPrompt_ClickResult]"
          DisplayName="Input Prompt"
          sap:VirtualizedContainerService.HintSize="485,382"
          InputFiveResult="[InputPrompt_InputFiveResult]"
          InputFourResult="[InputPrompt_InputFourResult]"
          InputOneResult="[InputPrompt_InputOneResult]"
          InputThreeResult="[InputPrompt_InputThreeResult]"
          InputTwoResult="[InputPrompt_InputTwoResult]"
          MinRequiredVersion="2.50.0.0"
          Moniker="{NEW-GUID}"
          Result="[InputPrompt_Result]"
          ResultString="[InputPrompt_ResultString]"
          RunAsCurrentLoggedOnUser="False"
          ScriptExecutionMethod="None"
          Timeout="[TimeOut]" Timeout_DisplayArg="Input Parameters.Time Out in Seconds"
          Title="[Title]" Title_DisplayArg="Input Parameters.Title"
          TypeName="InputPrompt"
          m_bTextLinkChange="False">
          <p:InputPrompt.ChosenButtons>
            <scg:List x:TypeArguments="acb:ButtonData" Capacity="1">
              <acb:ButtonData Text="Close" Type="Confirmation" />
            </scg:List>
          </p:InputPrompt.ChosenButtons>
          <p:InputPrompt.ChosenInputs>
            <scg:List x:TypeArguments="acb:InputData" Capacity="0" />
          </p:InputPrompt.ChosenInputs>
        </p:InputPrompt>

      </p:PolicySequence.Activities>
      <p:PolicySequence.Variables>
        <Variable x:TypeArguments="x:Double" Name="InputPrompt_ButtonSelected" />
        <Variable x:TypeArguments="x:String" Name="InputPrompt_InputOneResult" />
        <Variable x:TypeArguments="x:String" Name="InputPrompt_InputTwoResult" />
        <Variable x:TypeArguments="x:String" Name="InputPrompt_InputThreeResult" />
        <Variable x:TypeArguments="x:String" Name="InputPrompt_InputFourResult" />
        <Variable x:TypeArguments="x:String" Name="InputPrompt_InputFiveResult" />
        <Variable x:TypeArguments="x:String" Name="InputPrompt_ClickResult" />
        <Variable x:TypeArguments="x:String" Name="InputPrompt_ResultString" />
        <Variable x:TypeArguments="x:Double" Name="InputPrompt_Result" />
        <Variable x:TypeArguments="x:String" Default="DEFAULT_MESSAGE" Name="Message" />
        <Variable x:TypeArguments="x:String" Default="DEFAULT_TITLE" Name="Title" />
        <Variable x:TypeArguments="x:Double" Default="3600" Name="TimeOut" />
      </p:PolicySequence.Variables>
    </p:PolicySequence>
  </Activity>
</Policy>
```

**Placeholders:**
- `{NEW-GUID}` — unique GUID per element
- `POLICY_NAME` — the policy name
- `DEFAULT_TITLE` / `DEFAULT_MESSAGE` — default parameter values
- `BASE64_ENCODED_DESCRIPTION` — base64-encode the plain-text description

---

### Template 5: ClientTools with InArgs (RunPowerShellScript)

Client tool using RunPowerShellScript with InArgs parameter mapping. The `genArgEvent` must be a valid GUID (not null) when InArgs are present.

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<Policy ID="{NEW-GUID}" Name="POLICY_NAME" Description="BASE64_ENCODED_DESCRIPTION"
  Version="2.10.0.19" RemoteCategory="0" ExecutionType="Local" MinimumPSVersionRequired="3.0">
  <Object ID="{NEW-GUID}" Type="{B6FA6D8B-EEAA-47A6-8463-7F9A4F5BBB6E}"
    Data="&lt;xml&gt;&lt;Parameters&gt;&lt;Parameter ParameterName=&quot;Message&quot; Label=&quot;Message&quot; ParameterType=&quot;string&quot; Value=&quot;Hello World&quot; /&gt;&lt;/Parameters&gt;&lt;/xml&gt;" />
  <LinkManager xmlns:i="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://schemas.datacontract.org/2004/07/PolicyExecutor">
    <hashset xmlns:d2p1="http://schemas.datacontract.org/2004/07/System" />
  </LinkManager>
  <Diagnostics OriginalVersion="2.98.2.2" />
  <Activity mc:Ignorable="sads sap" x:Class="Policy Builder"
    xmlns="http://schemas.microsoft.com/netfx/2009/xaml/activities"
    xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
    xmlns:mva="clr-namespace:Microsoft.VisualBasic.Activities;assembly=System.Activities"
    xmlns:p="clr-namespace:PolicyExecutor;assembly=PolicyExecutionEngine"
    xmlns:sads="http://schemas.microsoft.com/netfx/2010/xaml/activities/debugger"
    xmlns:sap="http://schemas.microsoft.com/netfx/2009/xaml/activities/presentation"
    xmlns:scg="clr-namespace:System.Collections.Generic;assembly=mscorlib"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <x:Members>
      <x:Property Name="PolicyGUID" Type="InArgument(x:String)" />
    </x:Members>
    <sap:VirtualizedContainerService.HintSize>490,827</sap:VirtualizedContainerService.HintSize>
    <mva:VisualBasic.Settings>Assembly references and imported namespaces serialized as XML namespaces</mva:VisualBasic.Settings>
    <p:PolicySequence DisplayName="Policy Builder" sap:VirtualizedContainerService.HintSize="490,827"
      mva:VisualBasic.Settings="Assembly references and imported namespaces serialized as XML namespaces">
      <p:PolicySequence.Activities>

        <p:RunPowerShellScript genArgEvent="{GENARG-GUID}"
          AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
          DisplayName="Run PowerShell Script"
          sap:VirtualizedContainerService.HintSize="468,522"
          Moniker="{NEW-GUID}"
          OutPut_64="[RunPowerShellScript_OutPut_64]"
          Result="[RunPowerShellScript_Result]"
          ResultString="[RunPowerShellScript_ResultString]"
          Results_x64="[RunPowerShellScript_Results_x64]"
          RunAsCurrentLoggedOnUser="False"
          ScriptExecutionMethod="ExecuteDebug"
          TypeName="RunPowerShellScript"
          m_bTextLinkChange="False"
          script="BASE64_ENCODED_UTF16LE_SCRIPT">
          <p:RunPowerShellScript.InArgs>
            <p:InArg Item="{x:Null}" ItemProp="{x:Null}" x:Key="PSMessage"
              ArgType="string"
              DisplayArg="Input Parameters.Message"
              DisplayName="PSMessage"
              Name="PSMessage"
              isRequired="False">
              <p:InArg.Arg>
                <InArgument x:TypeArguments="x:Object">[Message]</InArgument>
              </p:InArg.Arg>
            </p:InArg>
          </p:RunPowerShellScript.InArgs>
          <p:RunPowerShellScript.OutArgs>
            <scg:Dictionary x:TypeArguments="x:String, p:OutArg" />
          </p:RunPowerShellScript.OutArgs>
        </p:RunPowerShellScript>

        <p:Log Message_Item="{x:Null}" Message_ItemProp="{x:Null}"
          AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
          DisplayName="Log"
          sap:VirtualizedContainerService.HintSize="454,88"
          LogMessage="[Log_LogMessage]"
          Message="[RunPowerShellScript_OutPut_64]"
          Message_DisplayArg="Run PowerShell Script.OutPut_64"
          Moniker="{NEW-GUID}"
          Result="[Log_Result]"
          ResultString="[Log_ResultString]"
          RunAsCurrentLoggedOnUser="False"
          ScriptExecutionMethod="ExecuteDebug"
          TypeName="Log"
          m_bTextLinkChange="False" />

      </p:PolicySequence.Activities>
      <p:PolicySequence.Variables>
        <Variable x:TypeArguments="x:String" Name="RunPowerShellScript_OutPut_64" />
        <Variable x:TypeArguments="x:String" Name="RunPowerShellScript_ResultString" />
        <Variable x:TypeArguments="scg:IEnumerable(x:Object)" Name="RunPowerShellScript_Results_x64" />
        <Variable x:TypeArguments="x:Double" Name="RunPowerShellScript_Result" />
        <Variable x:TypeArguments="x:String" Name="Log_LogMessage" />
        <Variable x:TypeArguments="x:String" Name="Log_ResultString" />
        <Variable x:TypeArguments="x:Double" Name="Log_Result" />
        <Variable x:TypeArguments="x:String" Default="Hello World" Name="Message" />
      </p:PolicySequence.Variables>
    </p:PolicySequence>
  </Activity>
</Policy>
```

**Placeholders:**
- `{NEW-GUID}` — unique GUID per element
- `{GENARG-GUID}` — a separate GUID for the genArgEvent (required when InArgs are present)
- `POLICY_NAME` — the policy name
- `BASE64_ENCODED_DESCRIPTION` — base64-encode the plain-text description
- `BASE64_ENCODED_UTF16LE_SCRIPT` — PowerShell script using `$PSMessage` parameter, encoded as base64 from UTF-16LE bytes

**InArgs key rules:**
- The `x:Key` must match the PowerShell `$ParameterName` in the script
- `DisplayArg` uses the pattern `"Input Parameters.{Label}"` or `"Global Variables.{Label}"`
- `genArgEvent` must be a valid GUID (not `{x:Null}`) when any InArgs are defined

---

### Template 6: Multi-Step Utility

Utility with multiple steps: check if folder exists, create if missing, then run PowerShell script.

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<Policy ID="{NEW-GUID}" Name="POLICY_NAME" Description="BASE64_ENCODED_DESCRIPTION"
  Version="2.10.0.19" RemoteCategory="0" ExecutionType="Local" MinimumPSVersionRequired="3.0">
  <Object ID="{NEW-GUID}" Type="{B6FA6D8B-EEAA-47A6-8463-7F9A4F5BBB6E}" Data="&lt;xml /&gt;" />
  <LinkManager xmlns:i="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://schemas.datacontract.org/2004/07/PolicyExecutor">
    <hashset xmlns:d2p1="http://schemas.datacontract.org/2004/07/System" />
  </LinkManager>
  <Diagnostics OriginalVersion="2.98.2.2" />
  <Activity mc:Ignorable="sads sap" x:Class="Policy Builder"
    xmlns="http://schemas.microsoft.com/netfx/2009/xaml/activities"
    xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
    xmlns:mva="clr-namespace:Microsoft.VisualBasic.Activities;assembly=System.Activities"
    xmlns:p="clr-namespace:PolicyExecutor;assembly=PolicyExecutionEngine"
    xmlns:sads="http://schemas.microsoft.com/netfx/2010/xaml/activities/debugger"
    xmlns:sap="http://schemas.microsoft.com/netfx/2009/xaml/activities/presentation"
    xmlns:scg="clr-namespace:System.Collections.Generic;assembly=mscorlib"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <x:Members>
      <x:Property Name="PolicyGUID" Type="InArgument(x:String)" />
    </x:Members>
    <sap:VirtualizedContainerService.HintSize>490,827</sap:VirtualizedContainerService.HintSize>
    <mva:VisualBasic.Settings>Assembly references and imported namespaces serialized as XML namespaces</mva:VisualBasic.Settings>
    <p:PolicySequence DisplayName="Policy Builder" sap:VirtualizedContainerService.HintSize="490,827"
      mva:VisualBasic.Settings="Assembly references and imported namespaces serialized as XML namespaces">
      <p:PolicySequence.Activities>

        <!-- STEP 1: Check if working folder exists -->
        <p:FolderExists Folder_Item="{x:Null}" Folder_ItemProp="{x:Null}"
          AssemblyName="PolicyExecutionEngine, Version=2.96.1.1, Culture=neutral, PublicKeyToken=null"
          Conditional="[FolderExists_Conditional]"
          DisplayName="Folder Exists"
          Folder="C:\DatoSoftware"
          Folder_DisplayArg="C:\DatoSoftware"
          sap:VirtualizedContainerService.HintSize="754,88"
          MinRequiredVersion="2.10.0.19"
          Moniker="{NEW-GUID}"
          Result="[FolderExists_Result]"
          ResultString="[FolderExists_ResultString]"
          RunAsCurrentLoggedOnUser="False"
          ScriptExecutionMethod="ExecuteDebug"
          TypeName="FolderExists"
          m_bTextLinkChange="False" />

        <!-- STEP 2: IfElse — create folder if it doesn't exist -->
        <p:IfElse CaseSensitive_Item="{x:Null}" CaseSensitive_ItemProp="{x:Null}"
          Condition_Item="{x:Null}" Condition_ItemProp="{x:Null}"
          Value_Item="{x:Null}" Value_ItemProp="{x:Null}"
          Variable_Item="{x:Null}" Variable_ItemProp="{x:Null}"
          AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
          CaseSensitive="False" CaseSensitive_DisplayArg="false"
          Condition="does not equal" Condition_DisplayArg="does not equal"
          DisplayName="Create Folder If Missing"
          sap:VirtualizedContainerService.HintSize="754,675"
          MinRequiredVersion="2.19.0.1"
          Moniker="{NEW-GUID}"
          Result="[IfElse_Result]" ResultString="[IfElse_ResultString]"
          RunAsCurrentLoggedOnUser="False"
          ScriptExecutionMethod="None"
          TypeName="IfElse"
          Value_DisplayArg="True" Value_Type="x:String"
          Variable="[FolderExists_Conditional]"
          Variable_DisplayArg="Folder Exists.Conditional"
          Variable_Type="x:String"
          m_bTextLinkChange="False">
          <p:IfElse.IfOption>
            <!-- Folder does NOT exist: create it -->
            <p:SequenceActivity DisplayName="Then" sap:VirtualizedContainerService.HintSize="355,274" Name="SequenceActivity">
              <p:SequenceActivity.Activities>
                <p:CreateFolder Folder_Item="{x:Null}" Folder_ItemProp="{x:Null}"
                  AssemblyName="PolicyExecutionEngine, Version=2.96.1.1, Culture=neutral, PublicKeyToken=null"
                  DisplayName="Create Folder"
                  Folder="C:\DatoSoftware"
                  FolderInfo="[CreateFolder_FolderInfo]"
                  Folder_DisplayArg="C:\DatoSoftware"
                  sap:VirtualizedContainerService.HintSize="342,88"
                  MinRequiredVersion="2.10.0.19"
                  Moniker="{NEW-GUID}"
                  Result="[CreateFolder_Result]"
                  ResultString="[CreateFolder_ResultString]"
                  RunAsCurrentLoggedOnUser="False"
                  ScriptExecutionMethod="ExecuteDebug"
                  TypeName="CreateFolder"
                  m_bTextLinkChange="False" />
              </p:SequenceActivity.Activities>
              <p:SequenceActivity.Variables>
                <Variable x:TypeArguments="x:String" Name="CreateFolder_FolderInfo" />
                <Variable x:TypeArguments="x:Double" Name="CreateFolder_Result" />
                <Variable x:TypeArguments="x:String" Name="CreateFolder_ResultString" />
              </p:SequenceActivity.Variables>
            </p:SequenceActivity>
          </p:IfElse.IfOption>
          <p:IfElse.ElseOption>
            <!-- Folder exists: nothing to do -->
            <p:SequenceActivity DisplayName="Else" sap:VirtualizedContainerService.HintSize="355,81" Name="SequenceActivity">
              <p:SequenceActivity.Activities />
              <p:SequenceActivity.Variables />
            </p:SequenceActivity>
          </p:IfElse.ElseOption>
          <p:IfElse.Value>
            <InArgument x:TypeArguments="x:Object">
              <p:ObjectLiteral Value="True" />
            </InArgument>
          </p:IfElse.Value>
        </p:IfElse>

        <!-- STEP 3: Run main PowerShell script -->
        <p:RunPowerShellScript genArgEvent="{x:Null}"
          AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
          DisplayName="Run PowerShell Script"
          sap:VirtualizedContainerService.HintSize="454,348"
          Moniker="{NEW-GUID}"
          OutPut_64="[RunPowerShellScript_OutPut_64]"
          Result="[RunPowerShellScript_Result]"
          ResultString="[RunPowerShellScript_ResultString]"
          Results_x64="[RunPowerShellScript_Results_x64]"
          RunAsCurrentLoggedOnUser="False"
          ScriptExecutionMethod="ExecuteDebug"
          TypeName="RunPowerShellScript"
          m_bTextLinkChange="False"
          script="BASE64_ENCODED_UTF16LE_SCRIPT">
          <p:RunPowerShellScript.InArgs>
            <scg:Dictionary x:TypeArguments="x:String, p:InArg" />
          </p:RunPowerShellScript.InArgs>
          <p:RunPowerShellScript.OutArgs>
            <scg:Dictionary x:TypeArguments="x:String, p:OutArg" />
          </p:RunPowerShellScript.OutArgs>
        </p:RunPowerShellScript>

        <!-- STEP 4: Log output -->
        <p:Log Message_Item="{x:Null}" Message_ItemProp="{x:Null}"
          AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
          DisplayName="Log"
          sap:VirtualizedContainerService.HintSize="454,88"
          LogMessage="[Log_LogMessage]"
          Message="[RunPowerShellScript_OutPut_64]"
          Message_DisplayArg="Run PowerShell Script.OutPut_64"
          Moniker="{NEW-GUID}"
          Result="[Log_Result]"
          ResultString="[Log_ResultString]"
          RunAsCurrentLoggedOnUser="False"
          ScriptExecutionMethod="ExecuteDebug"
          TypeName="Log"
          m_bTextLinkChange="False" />

        <!-- STEP 5: Log result string -->
        <p:Log Message_Item="{x:Null}" Message_ItemProp="{x:Null}"
          AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
          DisplayName="Log"
          sap:VirtualizedContainerService.HintSize="454,88"
          LogMessage="[Log_LogMessage_1]"
          Message="[RunPowerShellScript_ResultString]"
          Message_DisplayArg="Run PowerShell Script.Result String"
          Moniker="{NEW-GUID}"
          Result="[Log_Result_1]"
          ResultString="[Log_ResultString_1]"
          RunAsCurrentLoggedOnUser="False"
          ScriptExecutionMethod="ExecuteDebug"
          TypeName="Log"
          m_bTextLinkChange="False" />

      </p:PolicySequence.Activities>
      <p:PolicySequence.Variables>
        <Variable x:TypeArguments="x:String" Name="FolderExists_Conditional" />
        <Variable x:TypeArguments="x:Double" Name="FolderExists_Result" />
        <Variable x:TypeArguments="x:String" Name="FolderExists_ResultString" />
        <Variable x:TypeArguments="x:Double" Name="IfElse_Result" />
        <Variable x:TypeArguments="x:String" Name="IfElse_ResultString" />
        <Variable x:TypeArguments="x:String" Name="RunPowerShellScript_OutPut_64" />
        <Variable x:TypeArguments="x:String" Name="RunPowerShellScript_ResultString" />
        <Variable x:TypeArguments="scg:IEnumerable(x:Object)" Name="RunPowerShellScript_Results_x64" />
        <Variable x:TypeArguments="x:Double" Name="RunPowerShellScript_Result" />
        <Variable x:TypeArguments="x:String" Name="Log_LogMessage" />
        <Variable x:TypeArguments="x:String" Name="Log_ResultString" />
        <Variable x:TypeArguments="x:Double" Name="Log_Result" />
        <Variable x:TypeArguments="x:String" Name="Log_LogMessage_1" />
        <Variable x:TypeArguments="x:String" Name="Log_ResultString_1" />
        <Variable x:TypeArguments="x:Double" Name="Log_Result_1" />
      </p:PolicySequence.Variables>
    </p:PolicySequence>
  </Activity>
</Policy>
```

**Placeholders:**
- `{NEW-GUID}` — unique GUID per element
- `POLICY_NAME` — the policy name
- `BASE64_ENCODED_DESCRIPTION` — base64-encode the plain-text description
- `BASE64_ENCODED_UTF16LE_SCRIPT` — PowerShell script encoded as base64 from UTF-16LE bytes
- Replace `C:\DatoSoftware` with the appropriate folder path for your use case

---

## Key Conventions

- **Script encoding:** PowerShell scripts are stored as base64-encoded UTF-16LE strings in the `script` attribute
- **Description encoding:** The `Description` attribute on `<Policy>` is base64-encoded plain text (UTF-8)
- **GUIDs:** Every `Policy ID`, `Object ID`, and `Moniker` must be a unique GUID
- **Variable naming:** Activity output variables follow the pattern `ActivityName_PropertyName` with `_1`, `_2` suffixes for duplicates
- **Parameters vs GlobalVariables:** Parameters are per-run inputs shown to the operator; GlobalVariables are shared configuration values
