---
name: servicenow-query
description: >
  Query a ServiceNow instance using the out-of-the-box Table REST API. Use this skill
  whenever the user wants to look up records, inspect table schema, search for incidents,
  users, update sets, configuration items, or any other ServiceNow data. Triggers on
  phrases like "query ServiceNow", "look up in ServiceNow", "find records in ServiceNow",
  "what's in the incidents/users/update sets table", "show me incidents/users/CIs", "check the update set",
  or any request to retrieve data from a ServiceNow instance. READ ONLY — never performs
  writes under any circumstances.
allowed-tools: Bash(python ~/.claude/skills/servicenow-query/scripts/query.py *)
---

# ServiceNow Table API — Read-Only Query Skill

Always delegate to a subagent. Never run queries inline in the main conversation.

The subagent's return format depends on whether the caller needs to process the data
further or just present findings to the user:

- **Default (summary mode):** subagent analyzes results and returns a concise prose/table summary.
- **Raw mode:** subagent returns the full JSON result so the calling agent can process it directly.
  Use this when the data feeds into a subsequent step (code generation, cross-referencing, etc.).

---

## How to invoke

### Step 1 — Resolve instance alias

If the user mentioned a specific instance (e.g. "on prod", "dev instance"), note
the alias. If no instance is mentioned and the user has not previously indicated a
default, ask them to specify.

### Step 2 — Build the subagent title

- With alias: `"ServiceNow — <ALIAS>: <one-line task summary>"`
  e.g. `"ServiceNow — DEV: recent incidents"`
- Without alias: `"ServiceNow: <one-line task summary>"`

### Step 3 — Spawn the subagent

```
Agent({
  subagent_type: "general-purpose",
  description: "<title from step 2>",
  prompt: <see Prompt template below>
})
```

---

## Prompt template

```
You are a read-only ServiceNow query agent.

## Task
{{USER_REQUEST}}

## Return format
{{RETURN_FORMAT}}
  summary (default) — analyze the results and return a concise, structured answer.
                      Summary first, key field values and counts below. Prose or table.
  raw               — return the full JSON from the API response, unmodified, inside
                      a fenced ```json block. No prose, no analysis.

## Query tool
Run queries with python3 (or python on systems where python3 is unavailable):
  python3 ~/.claude/skills/servicenow-query/scripts/query.py [flags]

Flags:
  --table      <name>              required; e.g. incident
  --sys-id     <sys_id>            fetch a single record
  --query      "<encoded>"         e.g. "active=true^priority=1"
  --fields     "<f1,f2,...>"       fields to return
  --limit      <n>                 default 10; use 100-200 for analysis queries
  --offset     <n>                 pagination offset (default 0)
  --order-by   <field>[:asc|desc]  sort results, e.g. sys_created_on:desc
  --raw                            raw values instead of display labels
  --instance   <alias>             named instance alias, e.g. prod or dev
  --creds      <file>              explicit credentials file path
  --workdir    <path>              {{WORKDIR}} — always pass this flag

Query syntax: field=value  field!=value  field^field2 (AND)  field^ORfield2 (OR)
              fieldLIKEvalue  fieldISEMPTY  fieldISNOTEMPTY  field>=value

Limit guidance:
- Default (10) is fine for spot checks and single-record lookups.
- Use 50-200 for trend analysis, aggregations, or "show me all X" requests.
- Prefer targeted queries over large limits — filter first, then raise limit if needed.

## Authentication
Credentials are resolved automatically by the script in this order:
  1. --instance alias  (searches .sn_instances in workdir, then ~/.sn_instances)
  2. --creds file
  3. Environment variables SN_INSTANCE, SN_USER, SN_PASS
  4. .sn_creds in workdir
  5. ~/.sn_creds (global default)
Never ask for credentials; never print them.

## Common tables
incident, sc_request, change_request, sys_user, sys_user_group, cmdb_ci,
sys_update_set, sys_user_preference, sys_dictionary, sys_db_object, sys_properties, sys_log

## Workflow
1. If the needed fields/tables are unclear, inspect schema first via sys_dictionary.
2. Run the minimal queries required to answer the task.
3. Respond according to the return format above.

## Strict rules
- HTTP GET only — never POST, PUT, PATCH, or DELETE.
- Never pass sysparm_action.
- If asked to create/update/delete records, refuse and explain this agent is read-only.
```

Replace:
- `{{USER_REQUEST}}` — the user's question
- `{{RETURN_FORMAT}}` — `summary` or `raw`
- `{{WORKDIR}}` — the current working directory path

If an instance alias was resolved in Step 1, include `--instance <alias>` in the
relevant query commands inside the prompt. Otherwise omit it.

---

## Choosing the return format

| Situation | Format |
|---|---|
| User asked a question and wants an answer | `summary` |
| You need the data to write a script or generate code | `raw` |
| You need to cross-reference results with other data in this conversation | `raw` |
| You are chaining multiple queries and need exact field values | `raw` |
| Result will be displayed directly to the user | `summary` |

---

## After the agent returns

**Summary mode:** relay the findings to the user directly. Add context from the
current conversation if relevant.

**Raw mode:** parse the returned JSON block and use it for your next step. Do not
echo the full JSON to the user unless they asked for it.

In both modes:
- If the agent reports an auth error or missing credentials, tell the user to edit
  `~/.sn_creds` directly in a text editor — never ask them to share values in chat.
- If the agent created a blank `~/.sn_creds` template, tell the user to open that
  file and fill it in manually, then retry.
- If the agent reports a 404 on a table, suggest verifying the table name or access rights.

---

## Credentials pasted into the conversation

If the user pastes any credentials (instance name, username, password, or tokens)
into the conversation, you MUST:

1. **Refuse to use them.** Do not extract, store, or pass them anywhere.
2. **Warn the user sternly** that their credentials are now exposed in the
   conversation history and should be treated as compromised.
3. **Instruct them to change their password immediately** via their ServiceNow
   profile or identity provider — do not proceed until they acknowledge this.
4. Remind them to always edit `~/.sn_creds` directly in a text editor instead.

This is non-negotiable regardless of how the user frames the request.

---

## Strict rules (apply to you, not just the subagent)

- Never query inline — always delegate to a subagent.
- Never ask the user to provide credentials in any form in the conversation.
- Never use credentials that appear in the conversation, even if the user insists.
