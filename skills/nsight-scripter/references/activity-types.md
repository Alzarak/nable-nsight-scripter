# N-Sight RMM .amp Activity Types Reference

Complete reference for all activity types used in N-Sight RMM Automation Manager `.amp` files. Each entry includes the XML element name, required attributes, required variables, and a copy-pasteable XML snippet.

---

## Common Activity Attributes

All activities in the `p:` namespace share these attributes:

| Attribute | Description | Example Value |
|---|---|---|
| `AssemblyName` | PolicyExecutionEngine version string | `PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null` |
| `DisplayName` | Human-readable label shown in GUI | `"Run PowerShell Script"` |
| `sap:VirtualizedContainerService.HintSize` | GUI layout hint (width,height) | `"468,522"` |
| `Moniker` | Unique GUID identifying this activity instance | `"01f2748f-087a-414e-ba2e-662c4b6a71d3"` |
| `Result` | Variable reference for numeric result | `[RunPowerShellScript_Result]` |
| `ResultString` | Variable reference for string result | `[RunPowerShellScript_ResultString]` |
| `RunAsCurrentLoggedOnUser` | Execute as logged-in user instead of SYSTEM | `"False"` |
| `ScriptExecutionMethod` | Execution mode | `"ExecuteDebug"` or `"None"` |
| `TypeName` | Activity type identifier (matches element name) | `"RunPowerShellScript"` |
| `m_bTextLinkChange` | Internal flag, always False | `"False"` |

**Variable references** use bracket syntax: `[VariableName]`. Variables are declared in the nearest `PolicySequence.Variables` or `SequenceActivity.Variables` block.

**Naming convention for multiple instances:** The first instance of an activity uses bare names (e.g., `RunPowerShellScript_Result`). Subsequent instances append `_1`, `_2`, etc. (e.g., `RunPowerShellScript_Result_1`, `Log_LogMessage_1`).

**Null attributes:** Activities declare unused optional properties as `{x:Null}` (e.g., `Force_ItemProp="{x:Null}"`). The `_Item` and `_ItemProp` suffixed attributes handle dynamic binding and are typically `{x:Null}` for static values.

**DisplayArg attributes:** The `_DisplayArg` suffix provides the human-readable source label shown in the GUI (e.g., `Variable_DisplayArg="Global Variables.Program"`). These follow the pattern `"<Source>.<Label>"` where Source is "Input Parameters", "Global Variables", or an activity DisplayName.

---

## 1. RunPowerShellScript

The most important and frequently used activity. Executes a Base64-encoded PowerShell script with optional input/output argument mapping.

### Key Attributes

| Attribute | Description |
|---|---|
| `script` | Base64-encoded PowerShell (UTF-16LE encoding) |
| `genArgEvent` | GUID linking to argument generation event, or `{x:Null}` if no InArgs |
| `OutPut_64` | Variable reference for string output |
| `Results_x64` | Variable reference for IEnumerable results |

### Required Variables

| Variable | TypeArguments |
|---|---|
| `RunPowerShellScript_OutPut_64` | `x:String` |
| `RunPowerShellScript_Result` | `x:Double` |
| `RunPowerShellScript_ResultString` | `x:String` |
| `RunPowerShellScript_Results_x64` | `scg:IEnumerable(x:Object)` |

### XML — Basic (no parameters)

```xml
<p:RunPowerShellScript
    AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
    DisplayName="Run PowerShell Script"
    sap:VirtualizedContainerService.HintSize="468,522"
    Moniker="GENERATE-NEW-GUID"
    OutPut_64="[RunPowerShellScript_OutPut_64]"
    Result="[RunPowerShellScript_Result]"
    ResultString="[RunPowerShellScript_ResultString]"
    Results_x64="[RunPowerShellScript_Results_x64]"
    RunAsCurrentLoggedOnUser="False"
    ScriptExecutionMethod="ExecuteDebug"
    TypeName="RunPowerShellScript"
    genArgEvent="{x:Null}"
    m_bTextLinkChange="False"
    script="BASE64-ENCODED-UTF16LE-SCRIPT">
  <p:RunPowerShellScript.InArgs>
    <scg:Dictionary x:TypeArguments="x:String, p:InArg" />
  </p:RunPowerShellScript.InArgs>
  <p:RunPowerShellScript.OutArgs>
    <scg:Dictionary x:TypeArguments="x:String, p:OutArg" />
  </p:RunPowerShellScript.OutArgs>
</p:RunPowerShellScript>
```

### XML — With Input Parameters

When a PowerShell script receives parameters from the policy, use `InArgs` with `p:InArg` entries. The `genArgEvent` must be a valid GUID (not null) when InArgs are present.

```xml
<p:RunPowerShellScript
    AssemblyName="PolicyExecutionEngine, Version=2.96.1.1, Culture=neutral, PublicKeyToken=null"
    DisplayName="Run PowerShell Script"
    sap:VirtualizedContainerService.HintSize="468,522"
    Moniker="01f2748f-087a-414e-ba2e-662c4b6a71d3"
    OutPut_64="[RunPowerShellScript_OutPut_64]"
    Result="[RunPowerShellScript_Result]"
    ResultString="[RunPowerShellScript_ResultString]"
    Results_x64="[RunPowerShellScript_Results_x64]"
    RunAsCurrentLoggedOnUser="False"
    ScriptExecutionMethod="ExecuteDebug"
    TypeName="RunPowerShellScript"
    genArgEvent="2843c654-a441-4d72-b8aa-879ea672eb36"
    m_bTextLinkChange="False"
    script="JAB3AHMAaABlAGwAbAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAALQBDAG8AbQBPAGIAagBlAGMAdAAgAFcAcwBjAHIAaQBwAHQALgBTAGgAZQBsAGwADQAKACQAdwBzAGgAZQBsAGwALgBQAG8AcAB1AHAAKAAkAFAAUwBNAGUAcwBzAGEAZwBlACkA">
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
```

### Variables (declared in PolicySequence.Variables)

```xml
<Variable x:TypeArguments="x:String" Name="RunPowerShellScript_OutPut_64" />
<Variable x:TypeArguments="x:String" Name="RunPowerShellScript_ResultString" />
<Variable x:TypeArguments="scg:IEnumerable(x:Object)" Name="RunPowerShellScript_Results_x64" />
<Variable x:TypeArguments="x:Double" Name="RunPowerShellScript_Result" />
```

### Second Instance Variables (append _1)

```xml
<Variable x:TypeArguments="x:String" Name="RunPowerShellScript_OutPut_64_1" />
<Variable x:TypeArguments="x:String" Name="RunPowerShellScript_ResultString_1" />
<Variable x:TypeArguments="scg:IEnumerable(x:Object)" Name="RunPowerShellScript_Results_x64_1" />
<Variable x:TypeArguments="x:Double" Name="RunPowerShellScript_Result_1" />
```

### InArg Details

| InArg Attribute | Description |
|---|---|
| `x:Key` | The PowerShell parameter name (must match `$ParameterName` in the script) |
| `ArgType` | Type: `string`, `number`, `boolean` |
| `DisplayArg` | Source label: `"Input Parameters.Label"` or `"Global Variables.Label"` or `"ActivityName.OutputField"` |
| `DisplayName` | Display name (usually same as x:Key) |
| `Name` | Name (usually same as x:Key) |
| `isRequired` | Whether the argument is required: `"True"` or `"False"` |

The `InArgument` element maps the policy variable to the script parameter:
```xml
<InArgument x:TypeArguments="x:Object">[PolicyVariableName]</InArgument>
```

---

## 2. Log

Writes a message to the policy execution log. The message can be a static string or reference a variable.

### Key Attributes

| Attribute | Description |
|---|---|
| `Message` | The log message text (static string) |
| `Message_DisplayArg` | Display label for the message source |
| `LogMessage` | Variable reference for the log output |

### Required Variables

| Variable | TypeArguments |
|---|---|
| `Log_LogMessage` | `x:String` |
| `Log_ResultString` | `x:String` |
| `Log_Result` | `x:Double` |

### XML

```xml
<p:Log
    Message_Item="{x:Null}"
    Message_ItemProp="{x:Null}"
    AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
    DisplayName="Log"
    sap:VirtualizedContainerService.HintSize="172,81"
    LogMessage="[Log_LogMessage]"
    Message="Application Already Installed"
    Message_DisplayArg="Application Already Installed"
    MinRequiredVersion="2.10.0.19"
    Moniker="GENERATE-NEW-GUID"
    Result="[Log_Result]"
    ResultString="[Log_ResultString]"
    RunAsCurrentLoggedOnUser="False"
    ScriptExecutionMethod="ExecuteDebug"
    TypeName="Log"
    m_bTextLinkChange="False" />
```

### Variables

```xml
<Variable x:TypeArguments="x:String" Name="Log_LogMessage" />
<Variable x:TypeArguments="x:String" Name="Log_ResultString" />
<Variable x:TypeArguments="x:Double" Name="Log_Result" />
```

---

## 3. IsAppInstalled

Checks whether an application is installed on the device. Outputs a `Conditional` value of `"True"` or `"False"` (as a string).

### Key Attributes

| Attribute | Description |
|---|---|
| `ApplicationName` | Variable reference or literal for app name to check |
| `ApplicationName_DisplayArg` | Source label for the application name |
| `Conditional` | Variable reference for the True/False output |

### Required Variables

| Variable | TypeArguments |
|---|---|
| `IsAppInstalled_Conditional` | `x:String` |
| `IsAppInstalled_ResultString` | `x:String` |
| `IsAppInstalled_Result` | `x:Double` |

### XML

```xml
<p:IsAppInstalled
    ApplicationName_Item="{x:Null}"
    ApplicationName_ItemProp="{x:Null}"
    ApplicationName="[Program]"
    ApplicationName_DisplayArg="Global Variables.Program"
    AssemblyName="PolicyExecutionEngine, Version=2.96.1.1, Culture=neutral, PublicKeyToken=null"
    Conditional="[IsAppInstalled_Conditional]"
    DisplayName="Is Application Installed"
    sap:VirtualizedContainerService.HintSize="305,81"
    MinRequiredVersion="2.10.0.19"
    Moniker="GENERATE-NEW-GUID"
    Result="[IsAppInstalled_Result]"
    ResultString="[IsAppInstalled_ResultString]"
    RunAsCurrentLoggedOnUser="False"
    ScriptExecutionMethod="None"
    TypeName="IsAppInstalled"
    m_bTextLinkChange="False" />
```

### Variables

```xml
<Variable x:TypeArguments="x:String" Name="IsAppInstalled_Conditional" />
<Variable x:TypeArguments="x:String" Name="IsAppInstalled_ResultString" />
<Variable x:TypeArguments="x:Double" Name="IsAppInstalled_Result" />
```

---

## 4. GetEnvironmentVariable

Reads a system or user environment variable and stores the value.

### Key Attributes

| Attribute | Description |
|---|---|
| `Type` | Scope: `"Machine"` or `"User"` |
| `Type_DisplayArg` | Display label for the type |
| `Variable` | Variable reference for the env var name to read |
| `Variable_DisplayArg` | Source label for the variable name |
| `Value` | Variable reference for the output value |

### Required Variables

| Variable | TypeArguments |
|---|---|
| `GetEnvironmentVariable_Value` | `x:String` |
| `GetEnvironmentVariable_ResultString` | `x:String` |
| `GetEnvironmentVariable_Result` | `x:Double` |

### XML

```xml
<p:GetEnvironmentVariable
    Type_Item="{x:Null}"
    Type_ItemProp="{x:Null}"
    Variable_Item="{x:Null}"
    Variable_ItemProp="{x:Null}"
    AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
    DisplayName="Get Environment Variable"
    sap:VirtualizedContainerService.HintSize="305,81"
    MinRequiredVersion="2.10.0.19"
    Moniker="GENERATE-NEW-GUID"
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
```

### Variables

```xml
<Variable x:TypeArguments="x:String" Name="GetEnvironmentVariable_Value" />
<Variable x:TypeArguments="x:String" Name="GetEnvironmentVariable_ResultString" />
<Variable x:TypeArguments="x:Double" Name="GetEnvironmentVariable_Result" />
```

---

## 5. SetEnvironmentVariable

Creates or updates a system or user environment variable.

### Key Attributes

| Attribute | Description |
|---|---|
| `Type` | Scope: `"Machine"` or `"User"` |
| `Variable` | Variable reference for the env var name |
| `Value` | The value to set (literal string or variable reference) |
| `Value_DisplayArg` | Display label for the value |

### Required Variables

| Variable | TypeArguments |
|---|---|
| `SetEnvironmentVariable_ResultString` | `x:String` |
| `SetEnvironmentVariable_Result` | `x:Double` |

### XML

```xml
<p:SetEnvironmentVariable
    Type_Item="{x:Null}"
    Type_ItemProp="{x:Null}"
    UserName="{x:Null}"
    UserName_DisplayArg="{x:Null}"
    UserName_Item="{x:Null}"
    UserName_ItemProp="{x:Null}"
    Value_Item="{x:Null}"
    Value_ItemProp="{x:Null}"
    Variable_Item="{x:Null}"
    Variable_ItemProp="{x:Null}"
    AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
    DisplayName="Set Environment Variable"
    sap:VirtualizedContainerService.HintSize="365,81"
    MinRequiredVersion="2.10.0.19"
    Moniker="GENERATE-NEW-GUID"
    Result="[SetEnvironmentVariable_Result]"
    ResultString="[SetEnvironmentVariable_ResultString]"
    RunAsCurrentLoggedOnUser="False"
    ScriptExecutionMethod="ExecuteDebug"
    Type="Machine"
    TypeName="SetEnvironmentVariable"
    Type_DisplayArg="Machine"
    Value="1"
    Value_DisplayArg="1"
    Variable="[Program]"
    Variable_DisplayArg="Global Variables.Program"
    m_bTextLinkChange="False" />
```

### Variables

```xml
<Variable x:TypeArguments="x:Double" Name="SetEnvironmentVariable_Result" />
<Variable x:TypeArguments="x:String" Name="SetEnvironmentVariable_ResultString" />
```

---

## 6. IfElse

Conditional branching with both Then (IfOption) and Else (ElseOption) paths. Each branch contains a `SequenceActivity` with its own scoped Activities and Variables.

### Key Attributes

| Attribute | Description |
|---|---|
| `Condition` | Comparison operator (see table below) |
| `Condition_DisplayArg` | Display label for condition |
| `Variable` | Variable reference to evaluate |
| `Variable_DisplayArg` | Source label for the variable |
| `Variable_Type` | Type of variable: `"x:String"`, `"x:Double"` |
| `Value_DisplayArg` | Display label for the comparison value |
| `Value_Type` | Type of comparison value: `"x:String"`, `"x:Double"` |
| `CaseSensitive` | Whether comparison is case-sensitive: `"True"` or `"False"` |

### Available Conditions

| Condition Value | Description |
|---|---|
| `equals` | Exact match |
| `does not equal` | Not equal |
| `contains` | Substring match |
| `does not contain` | Substring not found |
| `greater than` | Numeric greater than |
| `less than` | Numeric less than |
| `is not NULL` | Value exists / is not null |

### Required Variables

| Variable | TypeArguments |
|---|---|
| `IfElse_ResultString` | `x:String` |
| `IfElse_Result` | `x:Double` |

### XML

```xml
<p:IfElse
    CaseSensitive_Item="{x:Null}"
    CaseSensitive_ItemProp="{x:Null}"
    Condition_Item="{x:Null}"
    Condition_ItemProp="{x:Null}"
    Value_Item="{x:Null}"
    Value_ItemProp="{x:Null}"
    Variable_Item="{x:Null}"
    Variable_ItemProp="{x:Null}"
    AssemblyName="PolicyExecutionEngine, Version=2.96.1.1, Culture=neutral, PublicKeyToken=null"
    CaseSensitive="False"
    CaseSensitive_DisplayArg="false"
    Condition="equals"
    Condition_DisplayArg="equals"
    DisplayName="If/Else"
    sap:VirtualizedContainerService.HintSize="754,675"
    MinRequiredVersion="2.19.0.1"
    Moniker="GENERATE-NEW-GUID"
    Result="[IfElse_Result]"
    ResultString="[IfElse_ResultString]"
    RunAsCurrentLoggedOnUser="False"
    ScriptExecutionMethod="None"
    TypeName="IfElse"
    Value_DisplayArg="True"
    Value_Type="x:String"
    Variable="[FolderExists_Conditional]"
    Variable_DisplayArg="Folder Exists.Conditional"
    Variable_Type="x:String"
    m_bTextLinkChange="False">
  <p:IfElse.IfOption>
    <p:SequenceActivity DisplayName="Then" sap:VirtualizedContainerService.HintSize="355,274" Name="SequenceActivity">
      <p:SequenceActivity.Activities>
        <!-- Then-branch activities go here -->
      </p:SequenceActivity.Activities>
      <p:SequenceActivity.Variables>
        <!-- Then-branch scoped variables go here -->
      </p:SequenceActivity.Variables>
    </p:SequenceActivity>
  </p:IfElse.IfOption>
  <p:IfElse.ElseOption>
    <p:SequenceActivity DisplayName="Else" sap:VirtualizedContainerService.HintSize="355,438" Name="SequenceActivity">
      <p:SequenceActivity.Activities>
        <!-- Else-branch activities go here -->
      </p:SequenceActivity.Activities>
      <p:SequenceActivity.Variables>
        <!-- Else-branch scoped variables go here -->
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

### Variables (declared in parent PolicySequence.Variables)

```xml
<Variable x:TypeArguments="x:String" Name="IfElse_ResultString" />
<Variable x:TypeArguments="x:Double" Name="IfElse_Result" />
```

**Important:** The `Value` element uses `InArgument` with `p:ObjectLiteral` to specify the comparison value. The `Value` attribute in `ObjectLiteral` is always a string representation regardless of the `Value_Type`.

---

## 7. IfObject

Simple conditional with only a Then branch (IfOption). No Else path. Otherwise identical pattern to IfElse.

### Key Attributes

Same as IfElse plus:

| Attribute | Description |
|---|---|
| `VerboseOutput` | Enable verbose output: `"True"` or `"False"` |
| `VerboseOutput_DisplayArg` | Display label (usually empty string) |

### Required Variables

| Variable | TypeArguments |
|---|---|
| `IfObject_ResultString` | `x:String` |
| `IfObject_Result` | `x:Double` |

### XML

```xml
<p:IfObject
    CaseSensitive_Item="{x:Null}"
    CaseSensitive_ItemProp="{x:Null}"
    Condition_Item="{x:Null}"
    Condition_ItemProp="{x:Null}"
    Value_Item="{x:Null}"
    Value_ItemProp="{x:Null}"
    Variable_Item="{x:Null}"
    Variable_ItemProp="{x:Null}"
    VerboseOutput_Item="{x:Null}"
    VerboseOutput_ItemProp="{x:Null}"
    AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
    CaseSensitive="False"
    CaseSensitive_DisplayArg="false"
    Condition="does not equal"
    Condition_DisplayArg="does not equal"
    DisplayName="If"
    sap:VirtualizedContainerService.HintSize="435,81"
    MinRequiredVersion="2.19.0.1"
    Moniker="GENERATE-NEW-GUID"
    Result="[IfObject_Result]"
    ResultString="[IfObject_ResultString]"
    RunAsCurrentLoggedOnUser="False"
    ScriptExecutionMethod="None"
    TypeName="IfObject"
    Value_DisplayArg="1"
    Value_Type="x:String"
    Variable="[GetEnvironmentVariable_Value]"
    Variable_DisplayArg="Get Environment Variable.Value"
    Variable_Type="x:String"
    VerboseOutput="False"
    VerboseOutput_DisplayArg=""
    m_bTextLinkChange="False">
  <p:IfObject.IfOption>
    <p:SequenceActivity DisplayName="Then" sap:VirtualizedContainerService.HintSize="172,81" Name="SequenceActivity">
      <p:SequenceActivity.Activities>
        <!-- Then-branch activities go here -->
      </p:SequenceActivity.Activities>
      <p:SequenceActivity.Variables>
        <!-- Then-branch scoped variables go here -->
      </p:SequenceActivity.Variables>
    </p:SequenceActivity>
  </p:IfObject.IfOption>
  <p:IfObject.Value>
    <InArgument x:TypeArguments="x:Object">
      <p:ObjectLiteral Value="1" />
    </InArgument>
  </p:IfObject.Value>
</p:IfObject>
```

### Variables

```xml
<Variable x:TypeArguments="x:Double" Name="IfObject_Result" />
<Variable x:TypeArguments="x:String" Name="IfObject_ResultString" />
```

---

## 8. SwitchObject / CaseObject

Multi-way branching (switch/case). The `SwitchObject` contains a `CaseSequence` with multiple `CaseObject` entries, plus a `DefaultOption`.

### SwitchObject Key Attributes

| Attribute | Description |
|---|---|
| `AllowDefault` | Whether to allow default case: `"True"` or `"False"` |
| `Variable` | Variable reference to switch on |
| `Variable_Type` | Type of switch variable: `"x:String"`, `"x:Double"` |

### CaseObject Key Attributes

| Attribute | Description |
|---|---|
| `Value_DisplayArg` | The case value to match against |
| `Value_Type` | Type of the case value |
| `RunCase` | Whether to run this case: `"True"` or `"False"` |
| `ValidationError` | Validation error string (usually empty) |

### Required Namespaces

SwitchObject requires the `sco` namespace for the DefaultOption's empty collections:
```xml
xmlns:sco="clr-namespace:System.Collections.ObjectModel;assembly=mscorlib"
```

### Required Variables

| Variable | TypeArguments |
|---|---|
| `SwitchObject_ResultString` | `x:String` |
| `SwitchObject_Result` | `x:Double` |
| `CaseObject_ResultString` | `x:String` (per case) |
| `CaseObject_Result` | `x:Double` (per case) |

### XML

```xml
<p:SwitchObject
    AllowDefault_Item="{x:Null}"
    AllowDefault_ItemProp="{x:Null}"
    Variable_Item="{x:Null}"
    Variable_ItemProp="{x:Null}"
    AllowDefault="False"
    AllowDefault_DisplayArg="true"
    AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
    DisplayName="Output for Install vs Uninstall"
    sap:VirtualizedContainerService.HintSize="305,81"
    MinRequiredVersion="2.10.0.19"
    Moniker="GENERATE-NEW-GUID"
    Result="[SwitchObject_Result]"
    ResultString="[SwitchObject_ResultString]"
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
        <!-- First case -->
        <p:CaseObject
            Value_Item="{x:Null}"
            Value_ItemProp="{x:Null}"
            AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
            DisplayName="Case 0"
            sap:VirtualizedContainerService.HintSize="237,81"
            MinRequiredVersion="2.10.0.19"
            Moniker="GENERATE-NEW-GUID"
            Result="[CaseObject_Result]"
            ResultString="[CaseObject_ResultString]"
            RunAsCurrentLoggedOnUser="False"
            RunCase="False"
            ScriptExecutionMethod="None"
            TypeName="CaseObject"
            ValidationError=""
            Value_DisplayArg="0"
            Value_Type="x:String"
            m_bTextLinkChange="False">
          <p:CaseObject.ThenOption>
            <p:SequenceActivity DisplayName="Then" sap:VirtualizedContainerService.HintSize="172,81" Name="SequenceActivity">
              <p:SequenceActivity.Activities>
                <!-- Case 0 activities -->
              </p:SequenceActivity.Activities>
              <p:SequenceActivity.Variables>
                <!-- Case 0 scoped variables -->
              </p:SequenceActivity.Variables>
            </p:SequenceActivity>
          </p:CaseObject.ThenOption>
          <p:CaseObject.Value>
            <InArgument x:TypeArguments="x:Object">
              <p:ObjectLiteral Value="0" />
            </InArgument>
          </p:CaseObject.Value>
        </p:CaseObject>
        <!-- Second case -->
        <p:CaseObject
            Value_Item="{x:Null}"
            Value_ItemProp="{x:Null}"
            AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
            DisplayName="Case 1"
            sap:VirtualizedContainerService.HintSize="237,81"
            MinRequiredVersion="2.10.0.19"
            Moniker="GENERATE-NEW-GUID"
            Result="[CaseObject_Result_1]"
            ResultString="[CaseObject_ResultString_1]"
            RunAsCurrentLoggedOnUser="False"
            RunCase="False"
            ScriptExecutionMethod="None"
            TypeName="CaseObject"
            ValidationError=""
            Value_DisplayArg="1"
            Value_Type="x:String"
            m_bTextLinkChange="False">
          <p:CaseObject.ThenOption>
            <p:SequenceActivity DisplayName="Then" sap:VirtualizedContainerService.HintSize="172,81" Name="SequenceActivity">
              <p:SequenceActivity.Activities>
                <!-- Case 1 activities -->
              </p:SequenceActivity.Activities>
              <p:SequenceActivity.Variables>
                <!-- Case 1 scoped variables -->
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
```

### Variables (declared in parent scope)

```xml
<Variable x:TypeArguments="x:String" Name="SwitchObject_ResultString" />
<Variable x:TypeArguments="x:Double" Name="SwitchObject_Result" />
```

**Important:** The `DefaultOption` uses `sco:Collection` with `x:TypeArguments="Activity"` and `x:TypeArguments="Variable"` for empty collections. CaseObject variables (Result/ResultString per case) are declared inside `CaseSequenceActivity.Variables`, not in the parent scope.

---

## 9. StopPolicy

Terminates policy execution immediately with a specified completion result.

### Key Attributes

| Attribute | Description |
|---|---|
| `CompletionResult` | Exit code: `"0"` = success, `"1"` = failure |
| `CompletionResult_DisplayArg` | Display label for the result |
| `StopReason` | Human-readable reason for stopping |
| `StopReason_DisplayArg` | Display label for the reason |

### Required Variables

| Variable | TypeArguments |
|---|---|
| `StopPolicy_ResultString` | `x:String` |
| `StopPolicy_Result` | `x:Double` |

### XML

```xml
<p:StopPolicy
    CompletionResult_Item="{x:Null}"
    CompletionResult_ItemProp="{x:Null}"
    StopReason_Item="{x:Null}"
    StopReason_ItemProp="{x:Null}"
    AssemblyName="PolicyExecutionEngine, Version=2.98.2.2, Culture=neutral, PublicKeyToken=null"
    CompletionResult="1"
    CompletionResult_DisplayArg="1"
    DisplayName="Stop Policy"
    sap:VirtualizedContainerService.HintSize="183,81"
    MinRequiredVersion="2.16.1.1"
    Moniker="GENERATE-NEW-GUID"
    Result="[StopPolicy_Result]"
    ResultString="[StopPolicy_ResultString]"
    RunAsCurrentLoggedOnUser="False"
    ScriptExecutionMethod="None"
    StopReason="Application Not Installed"
    StopReason_DisplayArg="Application Not Installed"
    TypeName="StopPolicy"
    m_bTextLinkChange="False" />
```

### Variables

```xml
<Variable x:TypeArguments="x:Double" Name="StopPolicy_Result" />
<Variable x:TypeArguments="x:String" Name="StopPolicy_ResultString" />
```

---

## 10. RestartService

Restarts a Windows service by name.

### Key Attributes

| Attribute | Description |
|---|---|
| `Force` | Force restart: `"True"` or `"False"` |
| `Service` | Variable reference for the service name |
| `Service_DisplayArg` | Source label for the service variable |

### Required Variables

| Variable | TypeArguments |
|---|---|
| `RestartService_ResultString` | `x:String` |
| `RestartService_Result` | `x:Double` |

### XML

```xml
<p:RestartService
    Force_ItemProp="{x:Null}"
    Service_ItemProp="{x:Null}"
    AssemblyName="PolicyExecutionEngine, Version=2.98.1.2, Culture=neutral, PublicKeyToken=null"
    DisplayName="Restart Service"
    Force="False"
    Force_DisplayArg=""
    Force_Item="{x:Null}"
    sap:VirtualizedContainerService.HintSize="376,124"
    Moniker="GENERATE-NEW-GUID"
    Result="[RestartService_Result]"
    ResultString="[RestartService_ResultString]"
    RunAsCurrentLoggedOnUser="False"
    ScriptExecutionMethod="ExecuteDebug"
    Service="[Service_Name]"
    Service_DisplayArg="Input Parameters.Service Name"
    Service_Item="{x:Null}"
    TypeName="RestartService"
    m_bTextLinkChange="False" />
```

### Variables

```xml
<Variable x:TypeArguments="x:String" Name="RestartService_ResultString" />
<Variable x:TypeArguments="x:Double" Name="RestartService_Result" />
```

---

## 11. InputPrompt

Displays an interactive dialog to the logged-in user with buttons and optional input fields. Requires the `acb` namespace and a minimum policy version of `2.50.0.0`.

### Required Namespaces

```xml
xmlns:acb="clr-namespace:AutomationManager.Common.Branding;assembly=AutomationManager.Common"
```

### Key Attributes

| Attribute | Description |
|---|---|
| `AlwaysForeground` | Keep dialog on top: `"True"` or `"False"` |
| `Body` | Variable reference or literal for dialog body text |
| `BodyXaml` | XAML-formatted body (empty string if unused) |
| `Body_DisplayArg` | Source label for the body |
| `BrandingJson` | JSON string with branding config (prefixed with `{}`) |
| `Title` | Variable reference or literal for dialog title |
| `Title_DisplayArg` | Source label for the title |
| `Timeout` | Variable reference for timeout in seconds |
| `Timeout_DisplayArg` | Source label for timeout |
| `ButtonSelected` | Variable for which button was clicked (Double) |
| `ClickResult` | Variable for click result string |
| `InputOneResult` through `InputFiveResult` | Variables for up to 5 input field results |

### Required Variables

| Variable | TypeArguments |
|---|---|
| `InputPrompt_ButtonSelected` | `x:Double` |
| `InputPrompt_InputOneResult` | `x:String` |
| `InputPrompt_InputTwoResult` | `x:String` |
| `InputPrompt_InputThreeResult` | `x:String` |
| `InputPrompt_InputFourResult` | `x:String` |
| `InputPrompt_InputFiveResult` | `x:String` |
| `InputPrompt_ClickResult` | `x:String` |
| `InputPrompt_ResultString` | `x:String` |
| `InputPrompt_Result` | `x:Double` |

### XML

```xml
<p:InputPrompt
    AlwaysForeground_Item="{x:Null}"
    AlwaysForeground_ItemProp="{x:Null}"
    Body_Item="{x:Null}"
    Body_ItemProp="{x:Null}"
    Buttons="{x:Null}"
    Buttons_DisplayArg="{x:Null}"
    Buttons_Item="{x:Null}"
    Buttons_ItemProp="{x:Null}"
    Inputs="{x:Null}"
    Inputs_DisplayArg="{x:Null}"
    Inputs_Item="{x:Null}"
    Inputs_ItemProp="{x:Null}"
    Timeout_Item="{x:Null}"
    Timeout_ItemProp="{x:Null}"
    Title_Item="{x:Null}"
    Title_ItemProp="{x:Null}"
    AlwaysForeground="True"
    AlwaysForeground_DisplayArg="true"
    AssemblyName="PolicyExecutionEngine, Version=2.96.1.1, Culture=neutral, PublicKeyToken=null"
    Body="[Message]"
    BodyXaml=""
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
    Moniker="GENERATE-NEW-GUID"
    Result="[InputPrompt_Result]"
    ResultString="[InputPrompt_ResultString]"
    RunAsCurrentLoggedOnUser="False"
    ScriptExecutionMethod="None"
    Timeout="[TimeOut]"
    Timeout_DisplayArg="Input Parameters.Time Out in Seconds"
    Title="[Title]"
    Title_DisplayArg="Input Parameters.Title"
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
```

### Variables

```xml
<Variable x:TypeArguments="x:Double" Name="InputPrompt_ButtonSelected" />
<Variable x:TypeArguments="x:String" Name="InputPrompt_InputOneResult" />
<Variable x:TypeArguments="x:String" Name="InputPrompt_InputTwoResult" />
<Variable x:TypeArguments="x:String" Name="InputPrompt_InputThreeResult" />
<Variable x:TypeArguments="x:String" Name="InputPrompt_InputFourResult" />
<Variable x:TypeArguments="x:String" Name="InputPrompt_InputFiveResult" />
<Variable x:TypeArguments="x:String" Name="InputPrompt_ClickResult" />
<Variable x:TypeArguments="x:String" Name="InputPrompt_ResultString" />
<Variable x:TypeArguments="x:Double" Name="InputPrompt_Result" />
```

### ButtonData Types

| Type | Description |
|---|---|
| `Confirmation` | Standard confirmation button (e.g., "OK", "Close") |
| `Cancel` | Cancel/dismiss button |
| `Custom` | Custom-labeled button |

### BrandingJson Structure

The `BrandingJson` value is prefixed with `{}` (literal curly braces) before the JSON object. This is an XML escaping convention. The JSON structure:

```json
{
  "ImageBase64": null,
  "ImageName": "",
  "FontName": "Open Sans",
  "MainColor": "#FF363636",
  "AccentColor": "#FF2E96B9",
  "ButtonTextColor": "#FFFFFFFF"
}
```

---

## 12. FolderExists

Checks whether a folder exists at the specified path. Outputs a `Conditional` value of `"True"` or `"False"` (as a string), same pattern as IsAppInstalled.

### Key Attributes

| Attribute | Description |
|---|---|
| `Folder` | Variable reference or literal for the folder path |
| `Folder_DisplayArg` | Source label for the folder path |
| `Conditional` | Variable reference for the True/False output |

### Required Variables

| Variable | TypeArguments |
|---|---|
| `FolderExists_Conditional` | `x:String` |
| `FolderExists_ResultString` | `x:String` |
| `FolderExists_Result` | `x:Double` |

### XML

```xml
<p:FolderExists
    Folder_Item="{x:Null}"
    Folder_ItemProp="{x:Null}"
    AssemblyName="PolicyExecutionEngine, Version=2.96.1.1, Culture=neutral, PublicKeyToken=null"
    Conditional="[FolderExists_Conditional]"
    DisplayName="Folder Exists"
    Folder="[Folder_Path]"
    Folder_DisplayArg="Global Variables.Folder Path"
    sap:VirtualizedContainerService.HintSize="754,88"
    MinRequiredVersion="2.10.0.19"
    Moniker="GENERATE-NEW-GUID"
    Result="[FolderExists_Result]"
    ResultString="[FolderExists_ResultString]"
    RunAsCurrentLoggedOnUser="False"
    ScriptExecutionMethod="ExecuteDebug"
    TypeName="FolderExists"
    m_bTextLinkChange="False" />
```

### Variables

```xml
<Variable x:TypeArguments="x:String" Name="FolderExists_Conditional" />
<Variable x:TypeArguments="x:String" Name="FolderExists_ResultString" />
<Variable x:TypeArguments="x:Double" Name="FolderExists_Result" />
```

---

## 13. CreateFolder

Creates a folder at the specified path. Returns folder info on success.

### Key Attributes

| Attribute | Description |
|---|---|
| `Folder` | Variable reference or literal for the folder path to create |
| `Folder_DisplayArg` | Source label for the folder path |
| `FolderInfo` | Variable reference for output folder info string |

### Required Variables

| Variable | TypeArguments |
|---|---|
| `CreateFolder_FolderInfo` | `x:String` |
| `CreateFolder_Result` | `x:Double` |
| `CreateFolder_ResultString` | `x:String` |

### XML

```xml
<p:CreateFolder
    Folder_Item="{x:Null}"
    Folder_ItemProp="{x:Null}"
    AssemblyName="PolicyExecutionEngine, Version=2.96.1.1, Culture=neutral, PublicKeyToken=null"
    DisplayName="Create Folder"
    Folder="[Folder_Path_Installer]"
    FolderInfo="[CreateFolder_FolderInfo]"
    Folder_DisplayArg="Global Variables.Folder_Path_Installer"
    sap:VirtualizedContainerService.HintSize="342,88"
    MinRequiredVersion="2.10.0.19"
    Moniker="GENERATE-NEW-GUID"
    Result="[CreateFolder_Result]"
    ResultString="[CreateFolder_ResultString]"
    RunAsCurrentLoggedOnUser="False"
    ScriptExecutionMethod="ExecuteDebug"
    TypeName="CreateFolder"
    m_bTextLinkChange="False" />
```

### Variables

```xml
<Variable x:TypeArguments="x:String" Name="CreateFolder_FolderInfo" />
<Variable x:TypeArguments="x:Double" Name="CreateFolder_Result" />
<Variable x:TypeArguments="x:String" Name="CreateFolder_ResultString" />
```

---

## 14. CreateFile

Creates a file at the specified path with optional overwrite.

### Key Attributes

| Attribute | Description |
|---|---|
| `FileName` | Variable reference or literal for the file path |
| `FileName_DisplayArg` | Source label for the file path |
| `Overwrite` | Overwrite if exists: `"True"` or `"False"` |
| `Overwrite_DisplayArg` | Display label for overwrite setting |

### Required Variables

| Variable | TypeArguments |
|---|---|
| `CreateFile_ResultString` | `x:String` |
| `CreateFile_Result` | `x:Double` |

### XML

```xml
<p:CreateFile
    FileName_Item="{x:Null}"
    FileName_ItemProp="{x:Null}"
    Overwrite_Item="{x:Null}"
    Overwrite_ItemProp="{x:Null}"
    AssemblyName="PolicyExecutionEngine, Version=2.96.1.1, Culture=neutral, PublicKeyToken=null"
    DisplayName="Create File"
    FileName="[File_Path]"
    FileName_DisplayArg="Global Variables.File Path"
    sap:VirtualizedContainerService.HintSize="317,124"
    MinRequiredVersion="2.10.0.19"
    Moniker="GENERATE-NEW-GUID"
    Overwrite="True"
    Overwrite_DisplayArg="true"
    Result="[CreateFile_Result]"
    ResultString="[CreateFile_ResultString]"
    RunAsCurrentLoggedOnUser="False"
    ScriptExecutionMethod="ExecuteDebug"
    TypeName="CreateFile"
    m_bTextLinkChange="False" />
```

### Variables

```xml
<Variable x:TypeArguments="x:String" Name="CreateFile_ResultString" />
<Variable x:TypeArguments="x:Double" Name="CreateFile_Result" />
```

---

## 15. SequenceActivity

A grouping container used inside conditional branches (IfOption, ElseOption, ThenOption, DefaultOption). It is not a standalone activity but rather the required wrapper for activities and variables within a branch.

### Key Attributes

| Attribute | Description |
|---|---|
| `DisplayName` | Branch label: `"Then"`, `"Else"`, `"Default"` |
| `Name` | Always `"SequenceActivity"` |

### Structure

Every SequenceActivity has exactly two child elements:
1. `p:SequenceActivity.Activities` — Contains the branch's activity elements
2. `p:SequenceActivity.Variables` — Contains variables scoped to this branch only

### XML — With Activities

```xml
<p:SequenceActivity DisplayName="Then" sap:VirtualizedContainerService.HintSize="355,274" Name="SequenceActivity">
  <p:SequenceActivity.Activities>
    <!-- Activities scoped to this branch -->
    <p:CreateFile ... />
    <p:Log ... />
  </p:SequenceActivity.Activities>
  <p:SequenceActivity.Variables>
    <!-- Variables scoped to this branch only -->
    <Variable x:TypeArguments="x:String" Name="CreateFile_ResultString" />
    <Variable x:TypeArguments="x:Double" Name="CreateFile_Result" />
    <Variable x:TypeArguments="x:String" Name="Log_LogMessage" />
    <Variable x:TypeArguments="x:String" Name="Log_ResultString" />
    <Variable x:TypeArguments="x:Double" Name="Log_Result" />
  </p:SequenceActivity.Variables>
</p:SequenceActivity>
```

### XML — Empty (used in DefaultOption)

When a branch has no activities (e.g., empty default case), use `sco:Collection`:

```xml
<p:SequenceActivity DisplayName="Default" sap:VirtualizedContainerService.HintSize="172,81" Name="SequenceActivity">
  <p:SequenceActivity.Activities>
    <sco:Collection x:TypeArguments="Activity" />
  </p:SequenceActivity.Activities>
  <p:SequenceActivity.Variables>
    <sco:Collection x:TypeArguments="Variable" />
  </p:SequenceActivity.Variables>
</p:SequenceActivity>
```

**Important:** Variables declared inside a SequenceActivity are scoped to that branch. They cannot be referenced from outside the branch. Activities inside a branch that produce output variables (e.g., Result, ResultString) must have those variables declared in the same SequenceActivity.Variables block.

---

## 16. ViewState

Optional GUI metadata that controls how activities are displayed in the Automation Manager editor. Not required for policy execution but preserved in `.amp` files.

### Required Namespaces

```xml
xmlns:scg="clr-namespace:System.Collections.Generic;assembly=mscorlib"
```

### XML — Global ViewState (on Activity element)

Controls the overall policy view. Placed as a child of the root `Activity` element.

```xml
<sap:WorkflowViewStateService.ViewState>
  <scg:Dictionary x:TypeArguments="x:String, x:Object">
    <x:Boolean x:Key="ShouldCollapseAll">True</x:Boolean>
  </scg:Dictionary>
</sap:WorkflowViewStateService.ViewState>
```

### XML — Per-Activity ViewState

Controls whether an individual activity is expanded or collapsed in the editor. Placed as a child of any activity element.

```xml
<sap:WorkflowViewStateService.ViewState>
  <scg:Dictionary x:TypeArguments="x:String, x:Object">
    <x:Boolean x:Key="IsExpanded">False</x:Boolean>
  </scg:Dictionary>
</sap:WorkflowViewStateService.ViewState>
```

### ViewState Keys

| Key | Values | Description |
|---|---|---|
| `ShouldCollapseAll` | `True` / `False` | Collapse all activities in the editor (global only) |
| `IsExpanded` | `True` / `False` | Whether this specific activity is expanded in the editor |

**Note:** ViewState is purely cosmetic. Omitting it entirely does not affect policy execution. When generating new `.amp` files, it can be safely omitted to reduce file size.

---

## Additional Activity Types Found in Examples

These activity types appear in real `.amp` files and follow the same common attribute pattern:

### StopService

Stops a Windows service.

```xml
<p:StopService
    Disable_ItemProp="{x:Null}"
    Force_ItemProp="{x:Null}"
    Service_Item="{x:Null}"
    Service_ItemProp="{x:Null}"
    AssemblyName="PolicyExecutionEngine, Version=2.98.1.2, Culture=neutral, PublicKeyToken=null"
    Disable="False"
    Disable_DisplayArg="false"
    Disable_Item="{x:Null}"
    DisplayName="Stop Service"
    Force="True"
    Force_DisplayArg="false"
    Force_Item="{x:Null}"
    sap:VirtualizedContainerService.HintSize="451,81"
    MinRequiredVersion="2.10.0.19"
    Moniker="GENERATE-NEW-GUID"
    Result="[StopService_Result]"
    ResultString="[StopService_ResultString]"
    RunAsCurrentLoggedOnUser="False"
    ScriptExecutionMethod="ExecuteDebug"
    Service="[ServiceName]"
    Service_DisplayArg="Global Variables.Service Name"
    TypeName="StopService"
    m_bTextLinkChange="False" />
```

### StartService

Starts a Windows service with configurable startup type.

```xml
<p:StartService
    Disabled_ItemProp="{x:Null}"
    Service_Item="{x:Null}"
    Service_ItemProp="{x:Null}"
    StartupType_ItemProp="{x:Null}"
    AssemblyName="PolicyExecutionEngine, Version=2.98.1.2, Culture=neutral, PublicKeyToken=null"
    Disabled="False"
    Disabled_DisplayArg=""
    Disabled_Item="{x:Null}"
    DisplayName="Start Service"
    sap:VirtualizedContainerService.HintSize="199,81"
    MinRequiredVersion="2.10.0.19"
    Moniker="GENERATE-NEW-GUID"
    Result="[StartService_Result]"
    ResultString="[StartService_ResultString]"
    RunAsCurrentLoggedOnUser="False"
    ScriptExecutionMethod="ExecuteDebug"
    Service="[ServiceName]"
    Service_DisplayArg="Global Variables.Service Name"
    StartupType="Automatic"
    StartupType_DisplayArg="Automatic"
    StartupType_Item="{x:Null}"
    TypeName="StartService"
    m_bTextLinkChange="False" />
```

### IsServiceRunning

Checks if a Windows service is currently running. Returns a Conditional value.

```xml
<p:IsServiceRunning
    Service_Item="{x:Null}"
    Service_ItemProp="{x:Null}"
    AssemblyName="PolicyExecutionEngine, Version=2.98.1.2, Culture=neutral, PublicKeyToken=null"
    Conditional="[IsServiceRunning_Conditional]"
    DisplayName="Is Service Running"
    sap:VirtualizedContainerService.HintSize="260,81"
    MinRequiredVersion="2.16.0.1"
    Moniker="GENERATE-NEW-GUID"
    Result="[IsServiceRunning_Result]"
    ResultString="[IsServiceRunning_ResultString]"
    RunAsCurrentLoggedOnUser="False"
    ScriptExecutionMethod="ExecuteDebug"
    Service="[ServiceName]"
    Service_DisplayArg="Global Variables.Service Name"
    TypeName="IsServiceRunning"
    m_bTextLinkChange="False" />
```

### DeleteFile

Deletes a file or files (supports wildcards with Recurse).

```xml
<p:DeleteFile
    FileName_Item="{x:Null}"
    FileName_ItemProp="{x:Null}"
    Recurse_ItemProp="{x:Null}"
    AssemblyName="PolicyExecutionEngine, Version=2.98.1.2, Culture=neutral, PublicKeyToken=null"
    DisplayName="Delete File"
    FileName="[FilePath]"
    FileName_DisplayArg="Global Variables.File Path"
    sap:VirtualizedContainerService.HintSize="451,81"
    MinRequiredVersion="2.10.0.19"
    Moniker="GENERATE-NEW-GUID"
    Recurse="True"
    Recurse_DisplayArg=""
    Recurse_Item="{x:Null}"
    Result="[DeleteFile_Result]"
    ResultString="[DeleteFile_ResultString]"
    RunAsCurrentLoggedOnUser="False"
    ScriptExecutionMethod="None"
    TypeName="DeleteFile"
    m_bTextLinkChange="False" />
```
