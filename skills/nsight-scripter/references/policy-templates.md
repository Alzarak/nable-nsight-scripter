# N-Sight RMM Policy Categories and Templates

Reference for understanding the four main policy categories, their patterns, and starter skeletons.

---

## Policy Categories

### 1. Deployment Policies

- **Purpose:** Install or deploy software to endpoints
- **Pattern:** Check if installed -> Download -> Install -> Verify -> Log result
- **Complexity:** High -- multiple `RunPowerShellScript` activities, conditional branches via `IfElse` and `SwitchObject`
- **Examples:**
  - `DnsFilterDeployment2025.amp` (1280 lines)
  - `SentinelOneDeployment2025.amp` (1213 lines)
  - `HuntressDeployment2025.amp` (1053 lines)
  - `AugmenttDeployment2025.amp` (1199 lines)
  - `CoveDeploymentFullAgent2025.amp` (1144 lines)
  - `ThreatlockerDeployment2025.amp` (1271 lines)
  - `LiongardDeployment2025.amp` (1215 lines)
- **Typical structure:**
  1. Parameters -- URL, API key, org ID, download URL, software folder/file paths, argument list, log paths
  2. Global variables -- `Software`, `Installed_Application`, `Remove`, `Install`, `BlobURL`, `User_Agent`
  3. `SwitchObject` on Override parameter (Bypass / Uninstall / Reinstall) setting control variables
  4. `IsAppInstalled` check against `Installed_Application`
  5. `RunPowerShellScript` -- download installer (with user agent, TLS setup)
  6. `RunPowerShellScript` -- silent install with argument list
  7. `RunPowerShellScript` -- verify installation succeeded
  8. `SetEnvironmentVariable` -- marker for checker policies to read
  9. `Log` activities throughout for output and result strings

### 2. Checker Policies

- **Purpose:** Verify software is installed and running; manage install/uninstall state tracking via environment variables
- **Pattern:** `IsAppInstalled` -> `GetEnvironmentVariable` -> `IfElse`/`SwitchObject` -> `SetEnvironmentVariable` -> `StopPolicy`
- **Complexity:** Medium
- **Examples:**
  - `CoveChecker.amp` (272 lines)
  - `DnsFilterChecker.amp` (426 lines)
  - `SentinelOneChecker.amp` (272 lines)
  - `HuntressChecker.amp` (321 lines)
  - `AugmenttChecker.amp` (304 lines)
  - `ThreatlockerChecker.amp` (292 lines)
  - `LiongardChecker.amp` (294 lines)
- **Typical structure:**
  1. Parameter `Task_Check` (0 = uninstall, 1 = install)
  2. Global variable `Program` -- the application display name to look up
  3. `IsAppInstalled` using `Program` global variable
  4. `GetEnvironmentVariable` -- reads machine-level env var named after `Program`
  5. `IfElse` -- compares `IsAppInstalled_Conditional` to expected state
     - **Then branch (installed):** `IfObject` checks if env var does not equal `0`, then `SetEnvironmentVariable` to `0`
     - **Else branch (not installed):** `IfObject` checks if env var does not equal `1`, then `SetEnvironmentVariable` to `1`
  6. `SwitchObject` on `Task_Check` -- evaluates whether state matches desired action
     - Case `0` (uninstall requested): if app is still installed, `StopPolicy` with error
     - Case `1` (install requested): if app is not installed, `StopPolicy` with error
  7. Final `Log` activity for result

### 3. Utility Policies

- **Purpose:** System administration tasks -- registry edits, service management, cleanup, diagnostics
- **Pattern:** Usually a single `RunPowerShellScript` plus `Log` activities; sometimes native activities like `RestartService`
- **Complexity:** Low to medium
- **Examples:**
  - `DisablePin.amp` (39 lines) -- registry edit via PowerShell
  - `RestartService.amp` (23 lines) -- native `RestartService` activity with parameter
  - `GPUpdate.amp` (41 lines) -- runs gpupdate via PowerShell
  - `CleanNableFiles.amp` (167 lines) -- multi-step cleanup
  - `DismSfcScanWithLogOutput.amp` (77 lines) -- DISM/SFC with output logging
  - `DisableHibernationAndSleep.amp` (59 lines)
  - `RemoveTools.amp` (629 lines) -- complex multi-tool removal
  - `Windows11Checker.amp` (67 lines)
  - `OneDriveSyncAlerts.amp` (44 lines)
- **Typical structure:**
  1. Optional parameters (e.g., service name, path)
  2. `RunPowerShellScript` with base64-encoded script (or native activity like `RestartService`)
  3. `Log` -- output from the script (`OutPut_64`)
  4. `Log` -- result string (`ResultString`)

### 4. ClientTools Policies

- **Purpose:** Client-facing tools -- display messages, prompts, file operations on the endpoint
- **Pattern:** Parameters for user input -> Execute -> Log
- **Complexity:** Low to medium
- **Examples:**
  - `Message.amp` (41 lines) -- `InputPrompt` with title, body, timeout
  - `LanMessage.amp` (36 lines) -- `RunPowerShellScript` using WScript.Shell popup
  - `AddTextFile.amp` (59 lines) -- writes text file via PowerShell
  - `GoToAssistantInstall.amp` (151 lines)
  - `LongPathsEnabled.amp` (138 lines)
- **Typical structure:**
  1. Parameters (message, title, timeout, file path, etc.)
  2. `InputPrompt` activity (for GUI messages) or `RunPowerShellScript` (for non-GUI operations)
  3. Optional `Log` activities

---

## Example Corpus Reference

Files organized by category with complexity ratings:

### Simple (start here)
| File | Lines | Category | Notes |
|------|-------|----------|-------|
| `RestartService.amp` | 23 | Utility | Native activity, single parameter |
| `LanMessage.amp` | 36 | ClientTools | Single RunPS with InArg |
| `DisablePin.amp` | 39 | Utility | RunPS + 2 Logs, no parameters |
| `Message.amp` | 41 | ClientTools | InputPrompt with 3 parameters |
| `GPUpdate.amp` | 41 | Utility | RunPS + 2 Logs |

### Medium
| File | Lines | Category | Notes |
|------|-------|----------|-------|
| `AddTextFile.amp` | 59 | ClientTools | RunPS with parameters |
| `DisableHibernationAndSleep.amp` | 59 | Utility | RunPS + Logs |
| `Windows11Checker.amp` | 67 | Utility | RunPS with conditional logic |
| `DismSfcScanWithLogOutput.amp` | 77 | Utility | Multi-step diagnostic |
| `LongPathsEnabled.amp` | 138 | ClientTools | Registry + conditional |
| `CleanNableFiles.amp` | 167 | Utility | Multi-step cleanup |
| `CoveChecker.amp` | 272 | Checker | Standard checker pattern |
| `SentinelOneChecker.amp` | 272 | Checker | Standard checker pattern |
| `ThreatlockerChecker.amp` | 292 | Checker | Standard checker pattern |
| `LiongardChecker.amp` | 294 | Checker | Standard checker pattern |
| `AugmenttChecker.amp` | 304 | Checker | Standard checker pattern |
| `HuntressChecker.amp` | 321 | Checker | Standard checker pattern |
| `DnsFilterChecker.amp` | 426 | Checker | Extended checker with extra logic |

### Complex
| File | Lines | Category | Notes |
|------|-------|----------|-------|
| `RemoveTools.amp` | 629 | Utility | Multi-tool removal |
| `HuntressDeployment2025.amp` | 1053 | Deployment | Full deployment pattern |
| `CoveDeploymentFullAgent2025.amp` | 1144 | Deployment | Full deployment pattern |
| `AugmenttDeployment2025.amp` | 1199 | Deployment | Full deployment pattern |
| `SentinelOneDeployment2025.amp` | 1213 | Deployment | Full deployment pattern |
| `LiongardDeployment2025.amp` | 1215 | Deployment | Full deployment pattern |
| `ThreatlockerDeployment2025.amp` | 1271 | Deployment | Full deployment pattern |
| `DnsFilterDeployment2025.amp` | 1280 | Deployment | Full deployment pattern |
| `CoveDeploymentDocuments2025.amp` | 1282 | Deployment | Full deployment pattern |

### Build System
- `Build.ps1` -- structured PowerShell build convention for compiling policies from source scripts

---

## Quick-Start Templates

### 1. Simple Utility Skeleton

Minimal policy: run a PowerShell script and log the output.

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<Policy ID="{NEW-GUID}" Name="POLICY_NAME" Description="BASE64_ENCODED_DESCRIPTION" Version="2.10.0.19" RemoteCategory="0" ExecutionType="Local" MinimumPSVersionRequired="0.0.0">
  <Object ID="{NEW-GUID}" Type="{B6FA6D8B-EEAA-47A6-8463-7F9A4F5BBB6E}" Data="&lt;xml /&gt;" />
  <LinkManager xmlns:i="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://schemas.datacontract.org/2004/07/PolicyExecutor">
    <hashset xmlns:d2p1="http://schemas.datacontract.org/2004/07/System" />
  </LinkManager>
  <Diagnostics OriginalVersion="2.96.1.1" />
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
          AssemblyName="PolicyExecutionEngine, Version=2.96.1.1, Culture=neutral, PublicKeyToken=null"
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
          AssemblyName="PolicyExecutionEngine, Version=2.96.1.1, Culture=neutral, PublicKeyToken=null"
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
          AssemblyName="PolicyExecutionEngine, Version=2.96.1.1, Culture=neutral, PublicKeyToken=null"
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

**Placeholders to fill:**
- `{NEW-GUID}` -- generate a unique GUID for each occurrence
- `POLICY_NAME` -- the policy name (no spaces, use underscores)
- `BASE64_ENCODED_DESCRIPTION` -- base64-encode the plain-text description
- `BASE64_ENCODED_UTF16LE_SCRIPT` -- the PowerShell script encoded as base64 from UTF-16LE bytes

---

### 2. Checker Skeleton

Standard checker: verify app installed state, reconcile environment variable, stop on mismatch.

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<Policy ID="{NEW-GUID}" Name="APP_NAMEChecker" Description="BASE64_ENCODED_DESCRIPTION"
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

        <!-- STEP 3: IfElse -- compare IsAppInstalled result to True -->
        <!--
          If installed (True): check env var != "0", if so set to "0"
          Else (not installed): check env var != "1", if so set to "1"
          See CoveChecker.amp for full IfElse/IfObject/SetEnvironmentVariable nesting.
        -->
        <p:IfElse CaseSensitive_Item="{x:Null}" CaseSensitive_ItemProp="{x:Null}"
          Condition_Item="{x:Null}" Condition_ItemProp="{x:Null}"
          Value_Item="{x:Null}" Value_ItemProp="{x:Null}"
          Variable_Item="{x:Null}" Variable_ItemProp="{x:Null}"
          AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
          CaseSensitive="False" CaseSensitive_DisplayArg="false"
          Condition="equals" Condition_DisplayArg="equals"
          DisplayName="Set Enviroment Variable"
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
            <!-- APP IS INSTALLED: set env var to 0 if not already -->
            <p:SequenceActivity DisplayName="Then" sap:VirtualizedContainerService.HintSize="172,81" Name="SequenceActivity">
              <p:SequenceActivity.Activities>
                <!-- INSERT: IfObject checking env var != "0", then SetEnvironmentVariable to "0" -->
              </p:SequenceActivity.Activities>
            </p:SequenceActivity>
          </p:IfElse.IfOption>
          <p:IfElse.ElseOption>
            <!-- APP IS NOT INSTALLED: set env var to 1 if not already -->
            <p:SequenceActivity DisplayName="Else" sap:VirtualizedContainerService.HintSize="172,81" Name="SequenceActivity">
              <p:SequenceActivity.Activities>
                <!-- INSERT: IfObject checking env var != "1", then SetEnvironmentVariable to "1" -->
              </p:SequenceActivity.Activities>
            </p:SequenceActivity>
          </p:IfElse.ElseOption>
          <p:IfElse.Value>
            <InArgument x:TypeArguments="x:Object">
              <p:ObjectLiteral Value="True" />
            </InArgument>
          </p:IfElse.Value>
        </p:IfElse>

        <!-- STEP 4: SwitchObject on Task_Check (0=uninstall, 1=install) -->
        <!--
          Case 0: if app still installed, StopPolicy with error
          Case 1: if app not installed, StopPolicy with error
          See CoveChecker.amp for full SwitchObject/CaseObject/StopPolicy nesting.
        -->
        <!-- INSERT: SwitchObject with CaseObject branches and StopPolicy -->

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
        <Variable x:TypeArguments="x:Double" Default="1" Name="Task_Check" />
        <Variable x:TypeArguments="x:String" Default="APPLICATION_DISPLAY_NAME" Name="Program" />
      </p:PolicySequence.Variables>
    </p:PolicySequence>
  </Activity>
</Policy>
```

**Placeholders to fill:**
- `{NEW-GUID}` -- unique GUID per element
- `APP_NAME` -- short application identifier (no spaces)
- `APPLICATION_DISPLAY_NAME` -- the name as it appears in Add/Remove Programs
- `BASE64_ENCODED_DESCRIPTION` -- base64-encode the plain-text description
- Fill in the `<!-- INSERT -->` comment blocks following the pattern in `CoveChecker.amp`

---

### 3. Deployment Skeleton

Full deployment: override switch, install check, download, install, verify, set env var.

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<Policy ID="{NEW-GUID}" Name="APP_NAME Deployment - FULL_NAME (2025)"
  Description="BASE64_ENCODED_DESCRIPTION"
  Version="2.98.0.2" MinRequiredVersion="2.98.0.2" RemoteCategory="0"
  ExecutionType="CurrentLoggedOnUser" MinimumPSVersionRequired="3.0">
  <Object ID="{NEW-GUID}" Type="{B6FA6D8B-EEAA-47A6-8463-7F9A4F5BBB6E}"
    Data="&lt;xml&gt;&lt;Parameters&gt;&lt;Parameter ParameterName=&quot;Override&quot; Label=&quot;Override - B = Bypass / U = Uninstall / R = Reinstall&quot; ParameterType=&quot;string&quot; Value=&quot;&quot; /&gt;&lt;/Parameters&gt;&lt;GlobalVariables&gt;&lt;Parameter ParameterName=&quot;APIURL&quot; Label=&quot;API URL&quot; ParameterType=&quot;string&quot; Value=&quot;&quot; /&gt;&lt;Parameter ParameterName=&quot;APIKEY&quot; Label=&quot;API KEY&quot; ParameterType=&quot;string&quot; Value=&quot;&quot; /&gt;&lt;Parameter ParameterName=&quot;Software&quot; Label=&quot;Software&quot; ParameterType=&quot;string&quot; Value=&quot;APP_NAME&quot; /&gt;&lt;Parameter ParameterName=&quot;Download_URL&quot; Label=&quot;Download URL&quot; ParameterType=&quot;string&quot; Value=&quot;&quot; /&gt;&lt;Parameter ParameterName=&quot;Software_Folder&quot; Label=&quot;Software Folder&quot; ParameterType=&quot;string&quot; Value=&quot;C:\DatoSoftware&quot; /&gt;&lt;Parameter ParameterName=&quot;Software_File&quot; Label=&quot;Software File&quot; ParameterType=&quot;string&quot; Value=&quot;C:\DatoSoftware\INSTALLER.exe&quot; /&gt;&lt;Parameter ParameterName=&quot;Argument_List&quot; Label=&quot;Argument List&quot; ParameterType=&quot;string&quot; Value=&quot;/S&quot; /&gt;&lt;Parameter ParameterName=&quot;Log_Folder&quot; Label=&quot;Log Folder&quot; ParameterType=&quot;string&quot; Value=&quot;C:\DatoLogs&quot; /&gt;&lt;Parameter ParameterName=&quot;Installed_Application&quot; Label=&quot;Installed Application&quot; ParameterType=&quot;string&quot; Value=&quot;APPLICATION_DISPLAY_NAME&quot; /&gt;&lt;Parameter ParameterName=&quot;Remove&quot; Label=&quot;Remove&quot; ParameterType=&quot;number&quot; Value=&quot;0&quot; /&gt;&lt;Parameter ParameterName=&quot;Install&quot; Label=&quot;Install&quot; ParameterType=&quot;number&quot; Value=&quot;0&quot; /&gt;&lt;/GlobalVariables&gt;&lt;/xml&gt;" />
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

        <!-- STEP 1: SwitchObject on Override parameter -->
        <!--
          Case "B" (Bypass): set Install=1 to skip install check
          Case "U" (Uninstall): set Remove=1
          Case "R" (Reinstall): set Remove=1 and Install=1
          Default: proceed normally
          See HuntressDeployment2025.amp for full SwitchObject pattern.
        -->
        <!-- INSERT: SwitchObject with CaseObject branches for B/U/R -->

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
        <!--
          If NOT installed AND Install != 1:
            RunPowerShellScript: Download installer
            RunPowerShellScript: Run silent install
            RunPowerShellScript: Verify installation
            SetEnvironmentVariable: Mark as installed
            Log: Output results
          If installed AND Remove == 1:
            RunPowerShellScript: Uninstall
          See HuntressDeployment2025.amp for full branching logic.
        -->
        <!-- INSERT: IfElse with nested RunPowerShellScript activities -->

        <!-- STEP 4: Final Log -->
        <p:Log Message_Item="{x:Null}" Message_ItemProp="{x:Null}"
          AssemblyName="PolicyExecutionEngine, Version=2.96.1.1, Culture=neutral, PublicKeyToken=null"
          DisplayName="Log"
          sap:VirtualizedContainerService.HintSize="479,88"
          LogMessage="[Log_LogMessage]"
          Message="[RunPowerShellScript_ResultString]"
          Message_DisplayArg="Run PowerShell Script.Result String"
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
        <Variable x:TypeArguments="x:String" Name="RunPowerShellScript_OutPut_64" />
        <Variable x:TypeArguments="x:String" Name="RunPowerShellScript_ResultString" />
        <Variable x:TypeArguments="scg:IEnumerable(x:Object)" Name="RunPowerShellScript_Results_x64" />
        <Variable x:TypeArguments="x:Double" Name="RunPowerShellScript_Result" />
        <Variable x:TypeArguments="x:String" Name="Log_LogMessage" />
        <Variable x:TypeArguments="x:String" Name="Log_ResultString" />
        <Variable x:TypeArguments="x:Double" Name="Log_Result" />
        <Variable x:TypeArguments="x:String" Default="" Name="Override" />
        <Variable x:TypeArguments="x:String" Default="" Name="APIURL" />
        <Variable x:TypeArguments="x:String" Default="" Name="APIKEY" />
        <Variable x:TypeArguments="x:String" Default="APP_NAME" Name="Software" />
        <Variable x:TypeArguments="x:String" Default="" Name="Download_URL" />
        <Variable x:TypeArguments="x:String" Default="C:\DatoSoftware" Name="Software_Folder" />
        <Variable x:TypeArguments="x:String" Default="C:\DatoSoftware\INSTALLER.exe" Name="Software_File" />
        <Variable x:TypeArguments="x:String" Default="/S" Name="Argument_List" />
        <Variable x:TypeArguments="x:String" Default="C:\DatoLogs" Name="Log_Folder" />
        <Variable x:TypeArguments="x:String" Default="APPLICATION_DISPLAY_NAME" Name="Installed_Application" />
        <Variable x:TypeArguments="x:Double" Default="0" Name="Remove" />
        <Variable x:TypeArguments="x:Double" Default="0" Name="Install" />
      </p:PolicySequence.Variables>
    </p:PolicySequence>
  </Activity>
</Policy>
```

**Placeholders to fill:**
- `{NEW-GUID}` -- unique GUID per element
- `APP_NAME` -- short software identifier
- `FULL_NAME` -- display name for the policy title
- `APPLICATION_DISPLAY_NAME` -- name as it appears in Add/Remove Programs
- `INSTALLER.exe` -- the installer filename
- `BASE64_ENCODED_DESCRIPTION` -- base64-encode the plain-text description
- Fill in `<!-- INSERT -->` comment blocks following `HuntressDeployment2025.amp`

---

### 4. ClientTools Skeleton

Client-facing tool: parameters for user input, InputPrompt or RunPowerShellScript.

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
  <Diagnostics OriginalVersion="2.96.1.1" />
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

        <!-- OPTION A: InputPrompt (GUI message to user) -->
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

        <!-- OPTION B: RunPowerShellScript (non-GUI, e.g., WScript popup or file operation) -->
        <!--
        <p:RunPowerShellScript genArgEvent="{NEW-GUID}"
          AssemblyName="PolicyExecutionEngine, Version=2.96.1.1, Culture=neutral, PublicKeyToken=null"
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
              ArgType="string" DisplayArg="Input Parameters.Message"
              DisplayName="PSMessage" Name="PSMessage" isRequired="False">
              <p:InArg.Arg>
                <InArgument x:TypeArguments="x:Object">[Message]</InArgument>
              </p:InArg.Arg>
            </p:InArg>
          </p:RunPowerShellScript.InArgs>
          <p:RunPowerShellScript.OutArgs>
            <scg:Dictionary x:TypeArguments="x:String, p:OutArg" />
          </p:RunPowerShellScript.OutArgs>
        </p:RunPowerShellScript>
        -->

      </p:PolicySequence.Activities>
      <p:PolicySequence.Variables>
        <!-- InputPrompt variables -->
        <Variable x:TypeArguments="x:Double" Name="InputPrompt_ButtonSelected" />
        <Variable x:TypeArguments="x:String" Name="InputPrompt_InputOneResult" />
        <Variable x:TypeArguments="x:String" Name="InputPrompt_InputTwoResult" />
        <Variable x:TypeArguments="x:String" Name="InputPrompt_InputThreeResult" />
        <Variable x:TypeArguments="x:String" Name="InputPrompt_InputFourResult" />
        <Variable x:TypeArguments="x:String" Name="InputPrompt_InputFiveResult" />
        <Variable x:TypeArguments="x:String" Name="InputPrompt_ClickResult" />
        <Variable x:TypeArguments="x:String" Name="InputPrompt_ResultString" />
        <Variable x:TypeArguments="x:Double" Name="InputPrompt_Result" />
        <!-- RunPowerShellScript variables (uncomment if using Option B) -->
        <!--
        <Variable x:TypeArguments="x:String" Name="RunPowerShellScript_OutPut_64" />
        <Variable x:TypeArguments="x:String" Name="RunPowerShellScript_ResultString" />
        <Variable x:TypeArguments="scg:IEnumerable(x:Object)" Name="RunPowerShellScript_Results_x64" />
        <Variable x:TypeArguments="x:Double" Name="RunPowerShellScript_Result" />
        -->
        <!-- Parameter variables -->
        <Variable x:TypeArguments="x:String" Default="DEFAULT_MESSAGE" Name="Message" />
        <Variable x:TypeArguments="x:String" Default="DEFAULT_TITLE" Name="Title" />
        <Variable x:TypeArguments="x:Double" Default="3600" Name="TimeOut" />
      </p:PolicySequence.Variables>
    </p:PolicySequence>
  </Activity>
</Policy>
```

**Placeholders to fill:**
- `{NEW-GUID}` -- unique GUID per element
- `POLICY_NAME` -- the policy name
- `DEFAULT_TITLE` / `DEFAULT_MESSAGE` -- default parameter values
- `BASE64_ENCODED_DESCRIPTION` -- base64-encode the plain-text description
- Choose Option A (InputPrompt) or Option B (RunPowerShellScript), delete the unused one

---

## Key Conventions

- **Script encoding:** PowerShell scripts are stored as base64-encoded UTF-16LE strings in the `script` attribute
- **Description encoding:** The `Description` attribute on `<Policy>` is base64-encoded plain text
- **GUIDs:** Every `Policy ID`, `Object ID`, and `Moniker` must be a unique GUID
- **Variable naming:** Activity output variables follow the pattern `ActivityName_PropertyName` with `_1`, `_2` suffixes for duplicates
- **Parameters vs GlobalVariables:** Parameters are per-run inputs shown to the operator; GlobalVariables are shared configuration values
- **Build.ps1:** Use the build system at `Examples/Build.ps1` for structured PowerShell script compilation into policies
