---
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, Agent
description: Analyze a project and generate an N-Sight RMM .amp policy or PowerShell script through a 4-stage agent pipeline
argument-hint: "create [amp|script] description of what to automate"
disable-model-invocation: false
---

Orchestrate a 4-stage agent pipeline to analyze the current project and generate an N-Sight RMM automation file (.amp policy or PowerShell script).

## Parse the Request

From the user's input, determine:
- **Output type**: `.amp` policy or `.ps1` standalone script
  - "amp", "policy", "automation" → .amp file
  - "script", "ps1", "powershell" → .ps1 file
  - If ambiguous, default to .amp
- **Purpose**: What the automation should do (deploy, check, monitor, configure, etc.)

## Pipeline Overview

This command runs 4 specialized agents in sequence. Each agent's output feeds the next:

```
Step 1: codebase-mapper (Haiku)  → Codebase Map
Step 2: project-analyzer (Sonnet) → Analysis Report
Step 3: automation-planner (Opus)  → Construction Plan
Step 4: automation-executor (Sonnet) → Final .amp/.ps1 File
```

## Execution

### Step 1: Map the Codebase

Tell the user:
> **Step 1/4:** Mapping the codebase...

Launch the **codebase-mapper** agent (Haiku — fast, cheap) with this prompt:

```
Map this project for N-Sight RMM automation creation.

User's request: {user's full request}
Output type: {amp or ps1}
Working directory: {current working directory}

Explore the codebase and produce a structured map identifying:
- Project type and tech stack
- Install/deploy scripts and methods
- Service definitions
- Configuration files
- Prerequisites
- Existing automation

Follow your Process instructions exactly.
```

Wait for the result. Save the codebase map.

### Step 2: Analyze Key Files

Tell the user:
> **Step 2/4:** Analyzing project files...

Launch the **project-analyzer** agent (Sonnet — deeper analysis) with this prompt:

```
Analyze this project for N-Sight RMM automation creation.

User's request: {user's full request}
Output type: {amp or ps1}

## Codebase Map (from Step 1)
{paste the full codebase map output here}

Read the key files identified in the map and produce a detailed analysis report.
Follow your Process instructions exactly.
```

Wait for the result. Save the analysis report.

### Step 3: Plan the Automation

Tell the user:
> **Step 3/4:** Planning the automation architecture...

Launch the **automation-planner** agent (Opus — highest quality planning) with this prompt:

```
Design an N-Sight RMM automation based on this analysis.

User's request: {user's full request}
Output type: {amp or ps1}

## Analysis Report (from Step 2)
{paste the full analysis report here}

Read the nsight-scripter skill and reference documents for schema details.
Read matching example files from ${CLAUDE_PLUGIN_ROOT}/Examples/.
Produce a complete construction plan that the executor can follow mechanically.
Follow your Process instructions exactly.
```

Wait for the result. Save the construction plan.

### Step 4: Execute the Plan

Tell the user:
> **Step 4/4:** Building the final file...

Launch the **automation-executor** agent (Sonnet — efficient execution) with this prompt:

```
Build the final N-Sight RMM automation file from this plan.

User's request: {user's full request}

## Construction Plan (from Step 3)
{paste the full construction plan here}

Read the matching example file from ${CLAUDE_PLUGIN_ROOT}/Examples/ as a structural template.
Assemble, encode, write, and validate the final file.
Follow your Process instructions exactly.
```

Wait for the result.

## Final Output

After all 4 stages complete, present to the user:

```
## Digest Complete

**Created:** {filename}
**Type:** {.amp policy | PowerShell script}
**Purpose:** {brief description}

### Pipeline Summary
| Step | Agent | Model | Result |
|------|-------|-------|--------|
| 1. Map | codebase-mapper | Haiku | {file count, key findings} |
| 2. Analyze | project-analyzer | Sonnet | {key details extracted} |
| 3. Plan | automation-planner | Opus | {activity count / script structure} |
| 4. Execute | automation-executor | Sonnet | {file written, validation result} |

### What Was Created
{2-3 sentence summary of the automation: what it does, parameters, key features}

### Next Steps
- Review the generated file
- Test with `/nsight-read {filename}` to verify structure
- Edit with `/nsight-edit {filename}` if adjustments are needed
```

## Error Handling

If any stage fails:
1. Report which stage failed and why
2. Show the output from the failed stage
3. Suggest what the user can do:
   - Retry the stage
   - Provide more information
   - Use `/nsight-create` for manual creation instead
