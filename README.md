<p align="center">
  <picture>
    <source srcset="assets/hero.svg" type="image/svg+xml"/>
    <img src="assets/hero.png" alt="ServiceNow Query Skill" width="100%"/>
  </picture>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/python-3.9%2B-3776ab?logo=python&logoColor=white" alt="Python"/>
  <img src="https://img.shields.io/badge/platform-macOS%20%7C%20Linux-lightgrey" alt="Platform"/>
  <img src="https://img.shields.io/badge/license-AGPL--3.0-blue" alt="License"/>
  <img src="https://img.shields.io/badge/requires-Claude%20Code-orange" alt="Requires Claude Code"/>
</p>

<p align="center">
  <strong>Query any ServiceNow table in plain English, directly inside Claude Code.</strong>
</p>

<p align="center">
  Ask things like <em>"show me open P1 incidents assigned to the network team"</em> and
  Claude queries the ServiceNow Table REST API, summarises the results, and returns
  them directly in your conversation — no copy-pasting URLs or writing API calls
  by hand.
</p>

<p align="center">
  <a href="#features">Features</a> ·
  <a href="#installation">Installation</a> ·
  <a href="#quick-start">Quick start</a> ·
  <a href="#examples">Examples</a> ·
  <a href="#authentication">Authentication</a> ·
  <a href="#credentials-setup">Credentials</a> ·
  <a href="#cli-reference">CLI reference</a> ·
  <a href="#security">Security</a> ·
  <a href="#contributors">Contributors</a> ·
  <a href="#sponsors">Sponsors</a>
</p>

## Features

- **Plain-English queries** — describe what you want; Claude builds and runs the query
- **Instance-validated development** — when writing ServiceNow scripts or integrations, Claude queries your actual instance to inspect real table schemas, verify field names, look up reference field targets, and confirm record structures, so the code it produces is correct for your specific instance from the start
- **Local execution** — queries run as a Python script on your machine; the AI never handles your credentials or touches your instance directly
- **Read-only by construction** — HTTP GET only; no code path can write, update, or delete records
- **Multi-instance support** — switch between dev, test, and prod (or multiple clients) with named aliases
- **Zero dependencies** — standard library only; nothing to install beyond Python 3.9

## Requirements

- [Claude Code](https://claude.ai/claude-code) (any recent version)
- Python 3.9 or later
- Network access to your ServiceNow instance

## Installation

Clone into your Claude skills directory:

```bash
# SSH
git clone git@github.com:Eiluviann/servicenow-query-skill.git \
  ~/.claude/skills/servicenow-query

# HTTPS
git clone https://github.com/Eiluviann/servicenow-query-skill \
  ~/.claude/skills/servicenow-query
```

No build step. No `pip install`. That's it.

## Quick start

1. Set up credentials (see [Credentials setup](#credentials-setup) below)
2. Open any Claude Code conversation
3. Ask naturally:

```
Show me open P1 incidents assigned to the network team
How many change requests were raised this week?
Find the user record for john.smith
What fields are on the cmdb_ci table?
Get the active update set for user jdoe
```

Claude summarises results inline. When building scripts or integrations it
fetches structured JSON and uses it directly, without dumping raw API output
into the conversation.

## Examples

**Operational queries**

```
Show me all P1 and P2 incidents opened in the last 24 hours with no assigned team.
How many emergency change requests were raised this month, and which teams raised the most?
Find all active admin accounts that have not logged in for more than 90 days.
What update sets are currently in progress, and who owns them?
```

**Instance-validated development** — Claude queries your actual instance to confirm
field names, choice values, and schema before writing any code, so the output is
correct for your specific instance rather than based on generic documentation.

```
Write a Business Rule that assigns incidents to the network team when category is
'Network'. Check the actual field names and choice values on my instance first.

I have an integration failing when setting 'state' to 'Implement' on change_request.
What is the actual stored value for that state on my instance?

Write a Python script fetching all Linux Server CIs — use only fields that actually
exist on my cmdb_ci table.
```

## Authentication

### Current support

Only **HTTP Basic Auth** is currently supported. Credentials are passed as a
`username:password` pair with every request.

OAuth 2.0 support is planned for a future release. Until then, follow the
account recommendations below to minimise risk.

### Recommended account setup

> [!IMPORTANT]
> Do not use an account with write access to your instance with this tool until
> OAuth is implemented. Basic Auth credentials stored in a file grant the same
> level of access as the account itself — a read-only account strictly limits
> the blast radius if credentials are ever compromised.

Create a dedicated service account for this tool, for example:

```
username:  aureliosolis.readonly
```

Assign it the following roles:

| Role | Purpose |
|---|---|
| `snc_read_only` | Enforces read-only access at the platform level — prevents any writes regardless of other permissions |
| `admin` | Required to query all tables, including restricted ones like `sys_dictionary` and `sys_properties` |

This combination gives the tool full read visibility across the instance while
making writes impossible even if the credentials are misused.

## Credentials setup

> [!WARNING]
> Always edit credentials files directly in a text editor — never paste values
> into a chat window. If credentials appear in a conversation they should be
> treated as compromised and rotated immediately.

### Global credentials (one-time setup)

Create `~/.sn_creds` and restrict it to your user:

```bash
touch ~/.sn_creds && chmod 600 ~/.sn_creds
```

```
SN_INSTANCE=mycompany
SN_USER=your.username
SN_PASS=your_password
```

`SN_INSTANCE` is the subdomain only — for `mycompany.service-now.com` use `mycompany`.

### Multiple instances (named profiles)

Create `~/.sn_instances` to switch between environments by alias:

```
# alias = display name | path to credentials file
prod = Acme Corp | ~/.sn_creds_acme_prod
dev  = Acme Corp | ~/.sn_creds_acme_dev
```

This file contains **no credentials** — only labels and paths. Each referenced
file follows the same `SN_INSTANCE / SN_USER / SN_PASS` format as `~/.sn_creds`.

```bash
chmod 600 ~/.sn_creds_acme_prod ~/.sn_creds_acme_dev
```

### Project-specific credentials

Drop a `.sn_creds` file in a project directory. It takes priority over `~/.sn_creds`
automatically whenever you work in that directory — no changes to `~/.sn_instances`
needed.

### Credential resolution order

The script stops at the first complete match:

| Priority | Source |
|---|---|
| 1 | `--instance <alias>` — resolves via `.sn_instances` (project dir, then `~`) |
| 2 | `--creds <file>` — explicit file path |
| 3 | `SN_INSTANCE` / `SN_USER` / `SN_PASS` environment variables |
| 4 | `.sn_creds` in the working directory |
| 5 | `~/.sn_creds` (global default) |

Individual environment variables can fill gaps left by a partial credentials file —
for example, `SN_INSTANCE` from the environment combined with `SN_USER`/`SN_PASS`
from `.sn_creds`.

## CLI reference

`scripts/query.py` can be run directly from a terminal without Claude Code:

```
python3 scripts/query.py --table <table> [options]

Required:
  --table      <name>              Table to query, e.g. incident

Options:
  --sys-id     <sys_id>            Fetch a single record by sys_id
  --query      "<encoded>"         Filter, e.g. "active=true^priority=1"
  --fields     "<f1,f2,...>"       Fields to return
  --limit      <n>                 Max records (default 10)
  --offset     <n>                 Pagination offset
  --order-by   <field>[:asc|desc]  Sort order, e.g. sys_created_on:desc
  --raw                            Raw values instead of display labels
  --instance   <alias>             Named instance from .sn_instances
  --creds      <file>              Explicit credentials file path
  --workdir    <path>              Directory to search for .sn_instances and
                                   .sn_creds (default: current directory)
```

### Query syntax

The `--query` flag accepts standard GlideRecord encoded query syntax. See the
ServiceNow documentation for the full reference:
[Encoded query strings →](https://docs.servicenow.com/bundle/washingtondc-platform-user-interface/page/use/using-lists/concept/c_EncodedQueryStrings.html)

### Examples

```bash
# 5 most recent open P1 incidents
python3 scripts/query.py \
  --table incident \
  --query "active=true^priority=1" \
  --fields "number,short_description,assigned_to,state" \
  --order-by sys_created_on:desc \
  --limit 5

# Look up a user
python3 scripts/query.py \
  --table sys_user \
  --query "user_name=jsmith" \
  --fields "sys_id,name,email,active"

# Inspect all fields on a table
python3 scripts/query.py \
  --table sys_dictionary \
  --query "name=incident^element!=NULL" \
  --fields "element,column_label,internal_type,mandatory" \
  --limit 200

# Query a named instance
python3 scripts/query.py \
  --instance prod \
  --table change_request \
  --query "state=1" \
  --limit 20
```

## Repository layout

```
servicenow-query-skill/
├── .gitignore       # excludes credential files and Python cache
├── README.md        # this file
├── SKILL.md         # Claude Code skill definition (triggers, prompt template)
└── scripts/
    └── query.py     # standalone query script; also invoked by the skill
```

`SKILL.md` is the Claude Code skill manifest. It is not needed to run
`query.py` directly from a terminal.

## Security

This tool is designed to be safe to use against production ServiceNow instances.

### Local execution — the AI never touches your credentials

This is a **local-only tool**. When Claude queries ServiceNow it does not make
the HTTP request itself. It invokes `scripts/query.py` on your local machine,
under your own user account, which makes the request and returns only the
results. Concretely:

- **Credentials stay on your machine.** They are read from local files or
  environment variables and passed directly to the HTTP request. They are never
  sent to Claude, never included in any prompt, and never leave your machine.
- **Claude receives only the query results** — not your credentials, not session
  tokens, not any data beyond what you asked for.
- **Access is controlled by you.** The script runs only via an explicit entry in
  your local `~/.claude/settings.json`. Remove that entry at any time to revoke
  access entirely.

### Additional safeguards

| Safeguard | Detail |
|---|---|
| Read-only by construction | HTTP GET only — no code path can POST, PUT, PATCH, or DELETE |
| Input validation | Table, instance, and sys_id values are checked against `[a-zA-Z0-9_-]` before URL construction |
| Secure file creation | Credential templates are written `0o600` (owner-read only) from the start |
| No third-party dependencies | Standard library only — zero supply-chain risk |
| Credential leak detection | Credentials pasted into a conversation are refused; the skill instructs immediate rotation |

## Contributing

Bug reports and pull requests are welcome. For significant changes please open
an issue first to discuss what you'd like to change.

## Contributors

<!-- Contributors will be listed here -->

## Sponsors

Thanks to the following for supporting this project:

<p>
  <a href="https://nelem.eu"><img src="https://nelem.eu/wp-content/uploads/2024/12/Nelem-white-1.webp" alt="Nelem" height="32"/></a>
</div>

*Built for [Claude Code](https://claude.ai/claude-code)*
