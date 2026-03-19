# N-Able N-Sight RMM .amp File Format Specification

## 1. Overview

`.amp` files are **UTF-8 XML 1.0** documents with **CRLF (`\r\n`) line endings**. They define automation policies for the N-Able N-Sight RMM agent. The format is based on **Microsoft Windows Workflow Foundation (WF) XAML** with a custom `PolicyExecutor` namespace for RMM-specific activities.

A complete `.amp` file has this top-level structure:

```xml
<?xml version="1.0" encoding="utf-8"?>
<Policy>
  <Object />
  <LinkManager />
  <Diagnostics />
  <Activity />
</Policy>
```

Each of these four child elements is required and must appear in this order.

---

## 2. Root `<Policy>` Element

The `<Policy>` element is the document root. All attributes are required.

### Attributes

| Attribute | Type | Description |
|---|---|---|
| `ID` | GUID | Unique identifier, lowercase with hyphens, no braces. Example: `a1b2c3d4-e5f6-7890-abcd-ef1234567890` |
| `Name` | String | Display name of the policy. Plain text. |
| `Description` | String | **Base64-encoded UTF-8** string of the policy description. An empty description is encoded as an empty string `""`. |
| `Version` | Version string | Semantic version of the policy. Example: `1.0.0.0` |
| `MinRequiredVersion` | Version string | Minimum agent version required. Use `2.98.2.2` as a safe default. |
| `RemoteCategory` | Integer | **Always `0`**. |
| `ExecutionType` | Enum | `Local` (runs as SYSTEM) or `CurrentLoggedOnUser` (runs as logged-in user). |
| `MinimumPSVersionRequired` | Version string | Minimum PowerShell version. Use `0.0` if no PowerShell is used, or `5.1` for modern scripts. |

### Example

```xml
<?xml version="1.0" encoding="utf-8"?>
<Policy ID="f47ac10b-58cc-4372-a567-0e02b2c3d479"
        Name="Example Policy"
        Description="VGhpcyBpcyBhIHRlc3QgcG9saWN5"
        Version="1.0.0.0"
        MinRequiredVersion="2.98.2.2"
        RemoteCategory="0"
        ExecutionType="Local"
        MinimumPSVersionRequired="5.1">
  <!-- child elements -->
</Policy>
```

Note: `Description="VGhpcyBpcyBhIHRlc3QgcG9saWN5"` is the Base64 encoding of the UTF-8 string `This is a test policy`.

---

## 3. `<Object>` Element (Parameters & Global Variables)

The `<Object>` element defines input parameters and global variables for the policy.

### Attributes

| Attribute | Type | Description |
|---|---|---|
| `ID` | GUID | GUID **with braces**. Example: `{B4FDB623-2227-4500-AA7F-B53A2C5F1B90}` |
| `Type` | GUID | **Always `{B6FA6D8B-EEAA-47A6-8463-7F9A4F5BBB6E}`** (fixed type identifier). |
| `Data` | String | **HTML-encoded XML** containing the parameter and variable definitions. |

### `Data` Attribute Format

The `Data` attribute contains an XML document that has been HTML-entity-encoded. When decoded, it has this structure:

#### When no parameters or global variables exist

```xml
<Object ID="{B4FDB623-2227-4500-AA7F-B53A2C5F1B90}"
        Type="{B6FA6D8B-EEAA-47A6-8463-7F9A4F5BBB6E}"
        Data="&lt;xml /&gt;" />
```

The literal `Data` value is `&lt;xml /&gt;` which decodes to `<xml />`.

#### When parameters and/or global variables exist

The decoded XML structure:

```xml
<xml>
  <Parameters>
    <Parameter Name="ServerName" Type="string" Variable="ServerName" />
    <Parameter Name="MaxRetries" Type="number" Variable="MaxRetries" />
  </Parameters>
  <GlobalVariables>
    <GlobalVariable Name="ResultMessage" Type="string" Variable="ResultMessage" />
    <GlobalVariable Name="ErrorCount" Type="number" Variable="ErrorCount" />
  </GlobalVariables>
</xml>
```

#### Parameter/GlobalVariable Attributes

| Attribute | Values | Description |
|---|---|---|
| `Name` | String | Display name shown in the UI. |
| `Type` | `string` or `number` | Data type. |
| `Variable` | String | Variable name used in the workflow. Must match a `<Variable>` in the `PolicySequence`. |

#### Encoded Example

With parameters, the `Data` attribute is HTML-encoded:

```xml
<Object ID="{B4FDB623-2227-4500-AA7F-B53A2C5F1B90}"
        Type="{B6FA6D8B-EEAA-47A6-8463-7F9A4F5BBB6E}"
        Data="&lt;xml&gt;&lt;Parameters&gt;&lt;Parameter Name=&quot;ServerName&quot; Type=&quot;string&quot; Variable=&quot;ServerName&quot; /&gt;&lt;Parameter Name=&quot;MaxRetries&quot; Type=&quot;number&quot; Variable=&quot;MaxRetries&quot; /&gt;&lt;/Parameters&gt;&lt;GlobalVariables /&gt;&lt;/xml&gt;" />
```

---

## 4. `<LinkManager>` Element

The `<LinkManager>` element is **always static boilerplate**. It does not vary between policies.

```xml
<LinkManager xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xmlns:xsd="http://www.w3.org/2001/XMLSchema" />
```

Always include it exactly as shown.

---

## 5. `<Diagnostics>` Element

The `<Diagnostics>` element records versioning metadata.

### Attributes

| Attribute | Description |
|---|---|
| `OriginalVersion` | The version of the N-Sight editor that created the policy. Use `2.98.2.2` as default. |

### Example

```xml
<Diagnostics OriginalVersion="2.98.2.2" />
```

---

## 6. `<Activity>` Element

The `<Activity>` element contains the actual workflow definition as a WF XAML document. It wraps a `PolicySequence` that holds all activities.

### Required Namespace Declarations

The `<Activity>` element must declare these namespaces:

```xml
<Activity
  xmlns="http://schemas.microsoft.com/netfx/2009/xaml/activities"
  xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
  xmlns:mva="clr-namespace:Microsoft.VisualBasic.Activities;assembly=System.Activities"
  xmlns:p="http://schemas.microsoft.com/netfx/2009/xaml/activities"
  xmlns:sads="http://schemas.microsoft.com/netfx/2009/xaml/activities/debugger"
  xmlns:sap="http://schemas.microsoft.com/netfx/2009/xaml/activities/presentation"
  xmlns:scg="clr-namespace:System.Collections.Generic;assembly=mscorlib"
  xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
  mc:Ignorable="sap sads"
  x:Class="PolicyExecutor.PolicyWorkflow">
```

### Conditional Additional Namespaces

Add these only when the corresponding activity types are used:

| Prefix | Declaration | When needed |
|---|---|---|
| `sco` | `xmlns:sco="clr-namespace:System.Collections.ObjectModel;assembly=mscorlib"` | When using `SwitchObject` activities |
| `acb` | `xmlns:acb="clr-namespace:AutomationCommon.Blocks;assembly=AutomationCommon"` | When using `InputPrompt` activities |

### Required Boilerplate Inside `<Activity>`

Immediately inside `<Activity>`, before the `<PolicySequence>`, you must include `x:Members` and `VisualBasic.Settings`:

```xml
<Activity ...namespaces...>
  <x:Members>
    <x:Property Name="PolicyGUID" Type="InArgument(x:String)" />
    <x:Property Name="HintSize" Type="InArgument(x:String)" />
  </x:Members>
  <mva:VisualBasic.Settings>
    <x:Null />
  </mva:VisualBasic.Settings>
  <PolicySequence ...>
    <!-- workflow content -->
  </PolicySequence>
</Activity>
```

---

## 7. `<PolicySequence>` Container

`PolicySequence` is the main workflow container. Activities inside it execute **top to bottom** in sequential order.

### Structure

```xml
<PolicySequence DisplayName="Main"
                sap:VirtualizedContainerService.HintSize="250,500"
                mva:VisualBasic.Settings="{x:Null}">
  <PolicySequence.Variables>
    <!-- Variable declarations -->
  </PolicySequence.Variables>
  <!-- Activity elements in execution order -->
</PolicySequence>
```

### Key Points

- `DisplayName` is always `"Main"` for the root sequence.
- `sap:VirtualizedContainerService.HintSize` controls UI layout; use a reasonable default like `"250,500"`.
- Activities are executed in the order they appear in the XML.
- The `<PolicySequence.Variables>` section declares all variables used by the workflow.

---

## 8. Variable Declarations

Variables are declared inside `<PolicySequence.Variables>`. Every variable referenced by any activity must be declared here, including parameter variables.

### Variable Types

| XAML Type | Description |
|---|---|
| `x:String` | Text/string values. |
| `x:Double` | Numeric values (integers and decimals). |
| `scg:IEnumerable(x:Object)` | Collection/list type (used for activity outputs that return lists). |

### Naming Convention

Variable names follow a specific pattern based on the activity that produces them:

1. **First output of an activity type**: `{ActivityType}_{OutputName}`
   - Example: `RunScript_PowerShellOutput`
2. **Subsequent outputs of the same activity type**: `{ActivityType}_{OutputName}_{N}` where N increments starting from 1
   - Example: `RunScript_PowerShellOutput_1`, `RunScript_PowerShellOutput_2`
3. **Parameter variables**: Named to match the `Variable` attribute in the `<Object>` `Data` XML. They use a `Default` attribute to set the initial value.

### Example Variable Declarations

```xml
<PolicySequence.Variables>
  <!-- Parameter variables with defaults -->
  <Variable x:TypeArguments="x:String" Default="localhost" Name="ServerName" />
  <Variable x:TypeArguments="x:Double" Default="3" Name="MaxRetries" />

  <!-- Activity output variables -->
  <Variable x:TypeArguments="x:String" Name="RunScript_PowerShellOutput" />
  <Variable x:TypeArguments="x:String" Name="RunScript_ErrorOutput" />
  <Variable x:TypeArguments="x:Double" Name="RunScript_ExitCode" />

  <!-- Second RunScript activity outputs -->
  <Variable x:TypeArguments="x:String" Name="RunScript_PowerShellOutput_1" />
  <Variable x:TypeArguments="x:String" Name="RunScript_ErrorOutput_1" />
  <Variable x:TypeArguments="x:Double" Name="RunScript_ExitCode_1" />

  <!-- Collection variable -->
  <Variable x:TypeArguments="scg:IEnumerable(x:Object)" Name="GetRegistryValue_RegistryValues" />
</PolicySequence.Variables>
```

### Default Values

- For `x:String` variables: `Default="some text"` (string literal).
- For `x:Double` variables: `Default="0"` or any numeric literal.
- When no default is needed, omit the `Default` attribute entirely.
- Parameter variables should always have a `Default` that matches the expected input type.

---

## 9. HTML Encoding Rules for the `Data` Attribute

The `<Object>` element's `Data` attribute contains XML that must be HTML-entity-encoded. Apply these substitutions **in order**:

| Character | Encoded As | Note |
|---|---|---|
| `&` | `&amp;` | Encode ampersands **first** to avoid double-encoding. |
| `<` | `&lt;` | |
| `>` | `&gt;` | |
| `"` | `&quot;` | Only inside the already-outer attribute quotes. |

### Encoding Example

**Original XML (decoded):**

```xml
<xml>
  <Parameters>
    <Parameter Name="Path" Type="string" Variable="Path" />
  </Parameters>
  <GlobalVariables />
</xml>
```

**Encoded for the `Data` attribute:**

```
&lt;xml&gt;&lt;Parameters&gt;&lt;Parameter Name=&quot;Path&quot; Type=&quot;string&quot; Variable=&quot;Path&quot; /&gt;&lt;/Parameters&gt;&lt;GlobalVariables /&gt;&lt;/xml&gt;
```

Note: The encoded value is placed on a single line with no extra whitespace. Newlines and indentation from the decoded XML are removed before encoding.

---

## Complete Minimal .amp Example

Below is a complete, valid `.amp` file with one parameter and a placeholder for activities:

```xml
<?xml version="1.0" encoding="utf-8"?>
<Policy ID="f47ac10b-58cc-4372-a567-0e02b2c3d479"
        Name="Minimal Example Policy"
        Description="QSBtaW5pbWFsIGV4YW1wbGU="
        Version="1.0.0.0"
        MinRequiredVersion="2.98.2.2"
        RemoteCategory="0"
        ExecutionType="Local"
        MinimumPSVersionRequired="5.1">
  <Object ID="{B4FDB623-2227-4500-AA7F-B53A2C5F1B90}"
          Type="{B6FA6D8B-EEAA-47A6-8463-7F9A4F5BBB6E}"
          Data="&lt;xml&gt;&lt;Parameters&gt;&lt;Parameter Name=&quot;TargetHost&quot; Type=&quot;string&quot; Variable=&quot;TargetHost&quot; /&gt;&lt;/Parameters&gt;&lt;GlobalVariables /&gt;&lt;/xml&gt;" />
  <LinkManager xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
               xmlns:xsd="http://www.w3.org/2001/XMLSchema" />
  <Diagnostics OriginalVersion="2.98.2.2" />
  <Activity xmlns="http://schemas.microsoft.com/netfx/2009/xaml/activities"
            xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
            xmlns:mva="clr-namespace:Microsoft.VisualBasic.Activities;assembly=System.Activities"
            xmlns:p="http://schemas.microsoft.com/netfx/2009/xaml/activities"
            xmlns:sads="http://schemas.microsoft.com/netfx/2009/xaml/activities/debugger"
            xmlns:sap="http://schemas.microsoft.com/netfx/2009/xaml/activities/presentation"
            xmlns:scg="clr-namespace:System.Collections.Generic;assembly=mscorlib"
            xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
            mc:Ignorable="sap sads"
            x:Class="PolicyExecutor.PolicyWorkflow">
    <x:Members>
      <x:Property Name="PolicyGUID" Type="InArgument(x:String)" />
      <x:Property Name="HintSize" Type="InArgument(x:String)" />
    </x:Members>
    <mva:VisualBasic.Settings>
      <x:Null />
    </mva:VisualBasic.Settings>
    <PolicySequence DisplayName="Main"
                    sap:VirtualizedContainerService.HintSize="250,500"
                    mva:VisualBasic.Settings="{x:Null}">
      <PolicySequence.Variables>
        <Variable x:TypeArguments="x:String" Default="localhost" Name="TargetHost" />
        <Variable x:TypeArguments="x:String" Name="RunScript_PowerShellOutput" />
        <Variable x:TypeArguments="x:String" Name="RunScript_ErrorOutput" />
        <Variable x:TypeArguments="x:Double" Name="RunScript_ExitCode" />
      </PolicySequence.Variables>
      <!-- Activities go here in execution order -->
    </PolicySequence>
  </Activity>
</Policy>
```

## Complete No-Parameters .amp Example

A policy with no parameters or global variables uses the minimal `Data` encoding:

```xml
<?xml version="1.0" encoding="utf-8"?>
<Policy ID="c9a1d3e5-7b2f-4a8c-9e0d-1f2a3b4c5d6e"
        Name="No Parameters Policy"
        Description=""
        Version="1.0.0.0"
        MinRequiredVersion="2.98.2.2"
        RemoteCategory="0"
        ExecutionType="Local"
        MinimumPSVersionRequired="0.0">
  <Object ID="{A1B2C3D4-E5F6-7890-ABCD-EF1234567890}"
          Type="{B6FA6D8B-EEAA-47A6-8463-7F9A4F5BBB6E}"
          Data="&lt;xml /&gt;" />
  <LinkManager xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
               xmlns:xsd="http://www.w3.org/2001/XMLSchema" />
  <Diagnostics OriginalVersion="2.98.2.2" />
  <Activity xmlns="http://schemas.microsoft.com/netfx/2009/xaml/activities"
            xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
            xmlns:mva="clr-namespace:Microsoft.VisualBasic.Activities;assembly=System.Activities"
            xmlns:p="http://schemas.microsoft.com/netfx/2009/xaml/activities"
            xmlns:sads="http://schemas.microsoft.com/netfx/2009/xaml/activities/debugger"
            xmlns:sap="http://schemas.microsoft.com/netfx/2009/xaml/activities/presentation"
            xmlns:scg="clr-namespace:System.Collections.Generic;assembly=mscorlib"
            xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
            mc:Ignorable="sap sads"
            x:Class="PolicyExecutor.PolicyWorkflow">
    <x:Members>
      <x:Property Name="PolicyGUID" Type="InArgument(x:String)" />
      <x:Property Name="HintSize" Type="InArgument(x:String)" />
    </x:Members>
    <mva:VisualBasic.Settings>
      <x:Null />
    </mva:VisualBasic.Settings>
    <PolicySequence DisplayName="Main"
                    sap:VirtualizedContainerService.HintSize="250,500"
                    mva:VisualBasic.Settings="{x:Null}">
      <PolicySequence.Variables>
        <!-- Activity output variables declared here -->
      </PolicySequence.Variables>
      <!-- Activities go here in execution order -->
    </PolicySequence>
  </Activity>
</Policy>
```
