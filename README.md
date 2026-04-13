# N-Sight RMM Automation Plugin

A Claude Code plugin for creating, editing, reading, and managing N-Able N-Sight RMM Automation Manager (`.amp`) policy files and PowerShell scripts.

## What It Does

This plugin gives Claude deep knowledge of:

- **The `.amp` XML schema** — the policy format used by N-Sight's Automation Manager
- **PowerShell embedding** — Base64 encoding (UTF-16LE), parameter injection via InArgs
- **N-Sight RMM conventions** — exit codes, output mechanisms, SYSTEM context constraints
- **Policy patterns** — deployment, checker, utility, and client-tool policy categories
- **Standalone scripting** — PowerShell scripts for N-Sight Script Manager upload

## Commands

| Command | Description |
|---------|-------------|
| `/nsight-create` | Create a new `.amp` policy or standalone PowerShell script |
| `/nsight-edit` | Edit an existing `.amp` file or PowerShell script |
| `/nsight-read` | Decode and explain an `.amp` file or PowerShell script |
| `/nsight-script` | Generate a standalone PowerShell script for Script Manager |

## Usage Examples

```
# Create a utility policy to disable Windows hibernation
/nsight-create utility disable Windows hibernation and sleep mode

# Create a checker policy for Huntress agent
/nsight-create checker verify Huntress agent is installed

# Read and explain an existing policy
/nsight-read path/to/SomeChecker.amp

# Edit a policy to change parameters
/nsight-edit path/to/SomePolicy.amp

# Generate a standalone PowerShell script
/nsight-script check if Windows Defender is running and report status

# Create a deployment script
/nsight-create deployment install DnsFilter agent with API key parameter
```

## Installation

### As a local plugin (for development)

```bash
claude --plugin-dir /path/to/nable-nsight-scripter
```

### As a project plugin

Copy or symlink this directory into your project, then add to your Claude Code settings.

## Reference Documentation

The plugin includes comprehensive reference docs in `skills/nsight-scripter/references/`:

| Document | Description |
|----------|-------------|
| `amp-format-spec.md` | Complete .amp XML schema specification |
| `activity-types.md` | All activity types with XML snippets and composition patterns |
| `policy-templates.md` | Schema rules and 6 complete ready-to-use templates |
| `powershell-conventions.md` | PowerShell encoding, templates, and best practices |

An `Examples/` directory with production `.amp` files is included for local development reference.

## Plugin Structure

```
nable-nsight-scripter/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest
├── skills/
│   └── nsight-scripter/
│       ├── SKILL.md             # Core knowledge skill
│       └── references/
│           ├── amp-format-spec.md       # .amp XML schema
│           ├── activity-types.md        # Activity type reference
│           ├── powershell-conventions.md # PS encoding & best practices
│           └── policy-templates.md      # Category patterns & templates
├── commands/
│   ├── nsight-create.md         # /nsight-create command
│   ├── nsight-edit.md           # /nsight-edit command
│   ├── nsight-read.md           # /nsight-read command
│   └── nsight-script.md        # /nsight-script command
├── Examples/                    # Production .amp files (local dev reference, gitignored)
└── README.md
```

## Author

DatoTech
