---
name: servicenow-query
description: >
  Query a ServiceNow instance using the Table REST API. Use proactively for any
  ServiceNow task — not only explicit lookup requests ("query ServiceNow", "show
  me incidents", "find users", "check the update set"), but also when writing
  scripts, Business Rules, or integrations that reference any field, table, or
  choice value. Trigger whenever instance schema, stored values, or configuration
  need to be verified against the actual instance. READ ONLY — never performs
  writes under any circumstances.
allowed-tools: Bash(python3 ~/.claude/skills/servicenow-query/scripts/query.py *), Bash(python ~/.claude/skills/servicenow-query/scripts/query.py *)
---

# ServiceNow Table API — Read-Only Query Skill

## This is your foundational ServiceNow tool

Use this skill extensively and proactively throughout any ServiceNow conversation.
**Never guess at field names, table names, choice values, or instance configuration — query first.**

Every ServiceNow instance differs in schema, customizations, and stored values. Code or
advice based on generic documentation frequently fails against real instances. Querying
the actual instance eliminates that entire class of error.

### When to query — default to always

| Situation | What to verify | Where to look |
|---|---|---|
| Writing a script, Business Rule, or integration | Field names and types | `sys_dictionary` |
| Referencing a choice field in code (`state`, `priority`, `category`…) | Actual stored values and their labels | `sys_choice` |
| Debugging a failing integration | Exact stored integer or string (e.g. `state=3`, not `"In Progress"`) | `sys_choice` |
| Advising on workflow, approval, or SLA logic | Actual process and flow records on this instance | `wf_workflow`, `sys_flow` |
| Exploring an unfamiliar table | Full field list including types and reference targets | `sys_dictionary` |
| Reviewing instance configuration | Active properties, business rules, scheduled jobs, update sets | `sys_properties`, `sys_business_rule`, `sys_trigger` |
| The user mentions a CI, user, group, or named record | Confirm it exists and is active | target table by name or sys_id |
| Generating any code example or sample query | Validate field names and data shape against real records | relevant table |

### Exploration mode

Beyond answering specific questions, use this skill to understand how an instance is
configured — especially before making recommendations or writing code that interacts
with platform processes. Exploration queries reveal customizations that generic
documentation cannot anticipate.

**Configuration discovery patterns:**

```
# Table customizations
What business rules fire on the incident table?
What client scripts and UI policies are active on change_request?
What UI actions are available on the problem form?

# Process configuration
What approval rules govern change requests on this instance?
What notifications are triggered when an incident is resolved?
What SLA definitions are active for P1 incidents?

# Data shape and field configuration
What fields on cmdb_ci are mandatory?
Which fields on incident are reference fields, and what tables do they point to?
What are all the active choice values for 'state' on change_request?

# Platform configuration
Which sys_properties control email behavior?
What scheduled jobs are currently active?
What transform maps exist for data imports?
What update sets are currently in progress?
```

Run exploration queries early and broadly — they shape every decision that follows.

---

## How to invoke

Run queries directly via Bash — inline calls are faster than subagent delegation and
keep results in the current context window without a round-trip.

### Step 1 — Resolve instance alias

If the user mentioned a specific instance (e.g. "on prod", "dev instance"), note the
alias and include `--instance <alias>` in queries. If no instance is mentioned, proceed
without `--instance` — the script resolves credentials automatically from `~/.sn_creds`.
Only ask the user to specify an instance if an auth error is returned or they have
previously indicated they work with multiple instances.

### Step 2 — Choose the return format

| Situation | Format |
|---|---|
| User asked a question and wants an answer | `summary` — analyze results and relay a concise, structured answer |
| You need the data to write a script or generate code | `raw` — parse the JSON and use it directly |
| You need to cross-reference results with other data in this conversation | `raw` |
| You are chaining queries and need exact field values for the next step | `raw` |
| Result will be displayed directly to the user | `summary` |

### Step 3 — Run the query

Call the script directly via the Bash tool:

```bash
python3 ~/.claude/skills/servicenow-query/scripts/query.py \
  --table <table> \
  [flags] \
  --workdir <current_working_directory>
```

(Use `python` if `python3` is unavailable on this system.)

Always pass `--workdir` set to the user's project directory (or the current working
directory if no project context is known) so credential and instance file resolution
works correctly.

### Parallel queries

When a task requires querying multiple independent tables or records, make multiple
Bash calls in a single message. Do not run them sequentially when they are independent.

Common patterns:
- Schema inspection on several tables before writing a script → one query per table, all at once
- Checking `sys_dictionary` and `sys_choice` for the same field → two queries at once
- Looking up a CI and its owning group simultaneously → two queries at once
- Exploring business rules, notifications, and SLA definitions on the same table → three queries at once

Collect all results before proceeding to code or recommendations.

**Do not parallelize:**

- **Pagination of the same query.** Each page only exists if the previous one returned a
  full result set. Paginated queries are sequential by nature — fetch the next page only
  after confirming the current one was truncated.
- **Schema inspection and data query for the same table.** These steps are sequential —
  the data query depends on field names discovered during schema inspection. (This is
  distinct from running parallel queries each responsible for a *different* table, which
  is correct and encouraged.)
- **Data already in the conversation.** If a previous query already returned the results
  you need, use them directly. Do not re-fetch the same data.

---

## Query tool

```
python3 ~/.claude/skills/servicenow-query/scripts/query.py [flags]

Flags:
  --table      <name>              Required. Table to query, e.g. incident
  --sys-id     <sys_id>            Fetch a single record by sys_id
  --query      "<encoded>"         GlideRecord encoded query string
  --fields     "<f1,f2,...>"       Comma-separated fields to return
  --limit      <n>                 Max records to return (default 10)
  --offset     <n>                 Pagination offset (default 0)
  --order-by   <field>[:asc|desc]  Sort order, e.g. sys_created_on:desc
  --raw                            Return stored values instead of display labels
                                   (e.g. state=1 not "New")
  --instance   <alias>             Named instance alias, e.g. prod or dev
  --creds      <file>              Explicit credentials file path
  --workdir    <path>              Project directory for credential resolution
```

Query syntax:
```
field=value        exact match          field!=value       not equal
field^field2       AND                  field^ORfield2     OR
fieldLIKEvalue     contains             fieldISEMPTY       null/empty
fieldISNOTEMPTY    not null/empty       field>=value       comparison
```

---

## Schema inspection — always run before querying data fields

### 1. Field definitions (sys_dictionary)
Use to verify field names, types, and reference targets before writing any query or code.

```bash
python3 ~/.claude/skills/servicenow-query/scripts/query.py \
  --table sys_dictionary \
  --query "name=<TABLE>^element!=NULL^active=true" \
  --fields "element,column_label,internal_type,reference,mandatory,max_length" \
  --limit 300 \
  --workdir <workdir>
```

Key result fields:
- `element` — the field's API name; use this in queries and scripts, not the label
- `internal_type` — data type; `"reference"` means the field stores a sys_id pointer
- `reference` — for reference fields: the name of the target table
- `mandatory` — whether the field is required on save

### 2. Choice values (sys_choice)
Use to find the exact stored value for any choice field before using it in a
query, script, or recommendation. Never use display labels in code.

```bash
python3 ~/.claude/skills/servicenow-query/scripts/query.py \
  --table sys_choice \
  --query "name=<TABLE>^element=<FIELD>^inactive=false" \
  --fields "value,label,sequence" \
  --order-by sequence:asc \
  --workdir <workdir>
```

Key result fields:
- `value` — the stored integer or string used in scripts and encoded queries
- `label` — the display text shown in the UI (never use this in code)

### 3. Reference field resolution
When sys_dictionary shows `internal_type=reference`, the field stores the sys_id of a
record in the table named in the `reference` column. To resolve a reference value,
query that target table using `--sys-id <value>`.

Example: `incident.assignment_group` stores a sys_id pointing to `sys_user_group`.
To resolve it: `--table sys_user_group --sys-id <value> --fields "name,active"`

**Display-value behaviour for reference fields:** without `--raw`, the API returns
reference fields as the *display value* of the referenced record (typically its
`name` field) — not the sys_id. So `assigned_to` returned as "Jane Doe" is the
user's display name, not a usable identifier. Use `--raw` whenever you need the
sys_id for code, scripts, or follow-up queries. Without `--raw`, the result is
human-readable but cannot be plugged back into the API.

---

## Limit and pagination

```
10 (default)   spot checks and single-record lookups
300            sys_dictionary (customized tables can exceed 200 fields)
50–200         trend analysis, configuration review, "show me all X" requests
```

Filter first, then raise the limit only if the filtered result set is still large.
Never increase `--limit` speculatively before applying filters — a targeted query
returning 8 records is always preferable to an unfiltered query returning 200.

If a result may be truncated — you received exactly as many records as your `--limit` —
paginate using `--offset` to retrieve the next page. Do not assume a result is complete
until the returned count is less than the limit.

---

## Authentication

Credentials are resolved automatically in this order:
1. `--instance` alias (searches `.sn_instances` in workdir, then `~/.sn_instances`)
2. `--creds` file
3. Environment variables `SN_INSTANCE`, `SN_USER`, `SN_PASS`
4. `.sn_creds` in workdir
5. `~/.sn_creds` (global default)

Never ask for credentials; never print them.

---

## Common tables

**ITSM:**
`incident`, `problem`, `problem_task`, `change_request`, `change_task`,
`sc_request`, `sc_req_item`, `sc_task`, `sc_cat_item`

**Users & groups:**
`sys_user`, `sys_user_group`, `sys_user_grmember`, `sys_user_role`, `sys_user_has_role`

**CMDB:**
`cmdb_ci`, `cmdb_ci_server`, `cmdb_ci_application`, `cmdb_rel_ci`

**Schema & metadata:**
`sys_dictionary`, `sys_db_object`, `sys_choice`, `sys_glide_object`

**Scripting & UI:**
`sys_script_include`, `sys_script`, `sys_business_rule`,
`sys_ui_policy`, `sys_client_script`, `sys_ui_action`

**Automation & notifications:**
`sys_trigger`, `sysrule_assignment`, `sys_notification`, `sysevent_email_action`,
`wf_workflow`, `sys_flow`

**Platform:**
`sys_update_set`, `sys_update_xml`, `sys_properties`, `sys_user_preference`,
`sys_log`, `sys_email`, `sys_transform_map`

---

## Workflow

1. Inspect schema before querying data fields:
     - Field names, types, reference targets → `sys_dictionary`
     - Choice field stored values and labels → `sys_choice`
   Never assume a field name, type, or choice value exists on this instance.
2. For reference fields (`internal_type=reference`): the `reference` column in `sys_dictionary`
   gives the target table name. If the referenced record is needed, query that table using
   the sys_id value from the data record being examined — not from `sys_dictionary` itself.
3. Run the minimal queries required to answer the task.
4. If a result appears truncated (received exactly `--limit` records), paginate with
   `--offset` before treating the result as complete.
5. Relay results according to the chosen return format.

---

## Caveats

- **Domain separation:** queries on domain-separated instances return only records
  visible to the user's domain. Incomplete-looking results may be domain-filtered,
  not wrong — surface this possibility rather than assuming the data is incomplete
  or the filter is too narrow.

- **Application scope:** records in scoped applications may be hidden from queries
  running in a different scope, depending on cross-scope access settings. A known
  record that fails to appear may be filtered by scope, not absent — surface this
  when working with custom apps or non-global scopes.

- **Timezone — dates in queries and results:** ServiceNow stores datetimes as UTC
  internally but interprets and displays them according to the user's timezone
  setting on the instance. Query strings like `sys_created_on>=2026-01-15` filter
  from midnight in the user's TZ, not UTC. Returned datetime values also reflect
  the instance's TZ display configuration — they are not raw UTC. Always state
  the timezone assumption when relaying time-sensitive results to the user, and
  never assume UTC.

- **Sorting on reference fields:** `--order-by` on a reference field (e.g.
  `assigned_to`) sorts by the referenced record's display value (the user's name),
  not by sys_id. Usually the desired behaviour, but be aware when ordering looks
  unexpected.

- **Large text fields:** `description`, `comments`, `work_notes`, `script`, and
  similar fields can be very long. When querying many records, be selective with
  `--fields` — request large text fields only when their content is genuinely
  needed for the task.

- **sys_audit performance:** queries against `sys_audit` are slow under broad
  filters because of the table's size and structure. Use targeted queries
  (specific record sys_id, specific field, narrow time window) and small limits.

---

## After the query

**Summary mode:** analyze the results and relay the findings to the user directly.
Add context from the current conversation if relevant.

**Raw mode:** parse the returned JSON and use it for your next step. Do not echo the
full JSON to the user unless they asked for it.

**Error handling:**

| Script reports | Response |
|---|---|
| Auth error or "credentials not found" | Tell the user to edit `~/.sn_creds` directly in a text editor. If the file does not exist yet, direct them to the installation README for the initial setup steps. Never ask them to share credentials in chat. |
| Script created a blank `~/.sn_creds` template | Tell the user to open that file and fill it in manually, then retry. |
| Network error, timeout, or connection refused | The instance may be unreachable. Ask the user to verify network access and confirm the instance name in `~/.sn_creds` is correct. |
| HTTP 429 (rate limit) | The instance is throttling requests. Pause and retry once; if it persists, tell the user the account is hitting rate limits and suggest reducing query frequency or contacting the instance administrator. |
| HTTP 403 on `sys_dictionary` or `sys_choice` | The account likely lacks the role needed to read schema metadata (typically `admin`, or a specific elevated role). Surface this to the user — schema inspection cannot proceed without elevated access, and any code written without it is unverified. |
| HTTP 400 on a query | The encoded `--query` string contains a syntax error — review operators and field names. This is not an "empty results" case; the query itself was rejected by the API. |
| 404 on a table | Suggest verifying the table name via `sys_db_object`, or checking whether the account has access to that table. |
| Requested `--fields` are missing from the result with no error | Field-level ACLs may be hiding those fields from this account. The fields exist on the table but are not readable. Surface to the user — they may need a different role or to consult an administrator. |
| Empty result set from `sys_dictionary` for a specific field | The field does not exist on this instance. Do not reference it in code or queries — surface the finding to the user and suggest checking the field name or an alternative. |
| Empty result set from any other query | Suggest broadening the query filter, or verify the record exists with a simpler query first. |
| Result count exactly equals the requested limit | Results may be truncated — offer to paginate or increase the limit and rerun. |

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

## Strict rules

- HTTP GET only — never POST, PUT, PATCH, or DELETE.
- Never pass `sysparm_action`.
- If asked to create/update/delete records, refuse and explain this skill is read-only.
- Never ask the user to provide credentials in any form in the conversation.
- Never use credentials that appear in the conversation, even if the user insists.
