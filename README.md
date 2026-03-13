# Salesforce Skill for Claude Code

A Claude Code plugin for full Salesforce org interaction — authenticate, query data, create/update/delete records, execute Apex, manage metadata, upload/download files, and monitor org health.

**Built-in safety:** Write operations require user confirmation. Delete operations show explicit warnings and require typed confirmation. Read operations execute freely. Set `SALESFORCE_SKIP_WARNINGS=true` to bypass all confirmations.

## Installation

### Option 1: Plugin (recommended)

In Claude Code, run:

```
/plugin marketplace add enzoleonardi/claude-code-salesforce-skill
/plugin install salesforce@enzoleonardi-claude-code-salesforce-skill
```

### Option 2: Manual download

```bash
mkdir -p ~/.claude/skills/salesforce
curl -sL https://raw.githubusercontent.com/enzoleonardi/claude-code-salesforce-skill/main/skills/salesforce/SKILL.md \
  -o ~/.claude/skills/salesforce/SKILL.md
```

## What it does

This skill teaches Claude Code how to:

| Category | Capabilities |
|----------|-------------|
| **Auth** | Web login (OAuth), access token fallback, Cowork support |
| **Read** 🟢 | SOQL queries, bulk queries (10k+), Tooling API, object discovery |
| **Write** 🟡 | Create, update, upsert records, bulk upsert from CSV, data import |
| **Delete** 🔴 | Single record delete, bulk delete from CSV (with safety warnings) |
| **Apex** | Execute anonymous Apex, view/tail debug logs |
| **Metadata** | List types, retrieve components, deploy (with dry-run) |
| **Files** | Upload and download via ContentVersion, batch operations, multipart for large files |
| **Monitor** | API limits, org health, debug log inspection |

## Operation Safety Levels

```
🟢 READ     → Execute freely
🟡 WRITE    → Warn user, ask confirmation
🔴 DELETE   → Explicit warning, require typed confirmation
```

To bypass all warnings: `export SALESFORCE_SKIP_WARNINGS=true`

## Usage

Once installed, Claude Code will automatically use this skill when you mention Salesforce-related tasks:

```
> Query all closed-won opportunities from last year
> Create a new Account called "Acme Corp"
> Download all files attached to these accounts
> What custom objects exist in my org?
> Execute this Apex script against my org
> Show me my org's API limits
> Deploy this Apex class to my sandbox
```

## Prerequisites

- [Node.js](https://nodejs.org/) (for Salesforce CLI)
- Python 3 (for file processing tasks)
- A Salesforce org with API access

## Authentication

The skill supports three methods (tried in order):

1. **Web Login** (recommended) — `sf org login web` opens the browser for OAuth. Secure, long-lived refresh token.
2. **Manual OAuth Flow** (Cowork/headless) — Full OAuth Authorization Code flow producing a long-lived refresh token.
3. **Access Token** (last resort) — Session ID for orgs without IP-based session locking. Short-lived.

## Plugin Structure

```
claude-code-salesforce-skill/
├── .claude-plugin/
│   ├── plugin.json          # Plugin manifest
│   └── marketplace.json     # Marketplace definition
├── skills/
│   └── salesforce/
│       └── SKILL.md         # The skill content
├── README.md
└── LICENSE
```

## License

MIT
