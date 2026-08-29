---
name: observeops-cli
description: >-
  Operate the ObserveOps CLI (`observeops` binary) against a Motadata ObserveOps
  instance: authentication and profiles, monitor inventory and health, active
  alerts, discovery jobs, credential profiles, alerting policies, dashboards and
  widgets, network configuration backups, live event streaming, and any raw
  ObserveOps API call. Use when the user mentions Motadata or ObserveOps tasks,
  "list monitors", "which devices are down", "show active alerts", "run a
  discovery", "check monitor health", "back up a device config", "watch alerts",
  "observeops api", or any ad-hoc ObserveOps request. Prefer the CLI over raw
  curl: it handles login, token refresh, self-signed TLS, the response envelope,
  pagination, and name-to-id resolution automatically.
license: UNLICENSED
---

# ObserveOps CLI

The `observeops` binary is a pre-authenticated gateway to a Motadata ObserveOps instance. It wraps roughly 795 API endpoints behind typed commands, validates write payloads against the product's own entity schemas before sending them, and resolves human names to ids. When a task touches a Motadata resource, reach for `observeops` instead of hand-rolling `curl`.

> This skill targets `observeops` 0.1.x against ObserveOps 8.2.7 and 10.0.0. The binary is the source of truth: run `observeops <command> --help` to verify anything claimed here, and `observeops docs` for the whole surface as JSON. If the CLI and this skill disagree, the CLI wins and this skill needs re-auditing.

## Prerequisites (run at session start)

```sh
observeops --version            # confirm the binary is on PATH
observeops info                 # connectivity, credentials, server version
```

**Always run `observeops info` first.** It is the health check: it proves the host is reachable, the stored token is valid, and reports the server version. Many behaviors differ between 8.2.7 and 10.0.0, so knowing the version up front prevents confusing failures later.

If `info` exits 4, the session is not authenticated. See [references/auth.md](references/auth.md).

## The mental model

| Layer | What it holds | Commands |
| --- | --- | --- |
| **Session** | Credentials, hosts, profiles | `auth login`, `auth status`, `profile use`, `info` |
| **Inventory** | The things being monitored | `monitor`, `group`, `tag`, `agent`, `credential`, `discovery` |
| **Detection** | What fires and why | `alert`, `policy metric\|event\|netroute\|trace\|rum` |
| **Presentation** | Saved views and queries | `dashboard`, `widget`, `report` |
| **Network config** | Device config backups and change approvals | `ncm config`, `ncm approval` |
| **Product data** | What each product observed | `metric`, `log`, `flow`, `trace`, `rum`, `slo`, `compliance` |
| **Live** | The server's event stream | `watch alerts`, `watch events`, `query run` |
| **Everything else** | Every remaining resource type | `resource` |
| **Escape hatch** | Any endpoint at all | `api` |

Roughly fifteen resources have a typed command. The product has ninety-six. Anything without a typed command is still reachable through `resource`, with the same validation and name resolution.

## Dotted keys are literal (the single most important rule)

Motadata property names contain dots but are **not** nested paths. `object.name` is one key whose text is `object.name`. It is never `{"object": {"name": ...}}`.

```sh
# Correct
observeops monitor create --set 'object.name=web-01' --set 'object.host=10.0.0.5'

# Wrong: this sends a nested object the API will reject
--set 'object={"name":"web-01"}'
```

This is why the CLI ships `--fields` instead of leaning on `jq`: `--fields object.name` looks up the literal key. In `jq` the same thing must be written `.["object.name"]`, and `.object.name` silently yields null. Piping to `jq` still works because stdout is clean, but prefer `--fields` for this API.

## Discover resources, do not memorize them

```sh
observeops resource types                     # all 96 types
observeops resource types --untyped           # those without a nicer typed command
observeops resource types --mount /settings   # filter by API mount
observeops resource describe slo-profiles     # verbs, required fields, allowed values, custom routes
```

`resource describe` is the discovery entry point for writes. It prints exactly which fields are required and, for enum fields, which values are accepted, so a payload can be built without reading product docs:

```
slo-profiles  /settings/slo-profiles
  verbs      list, get, create, update, delete, search, delete-all
  name field slo.profile.name

  Required to create
    slo.profile.name (string)
    slo.profile.type (string)  one of: "Availability", "Performance"
```

Use `--json` on `describe` to get every property, not just the first twenty.

## Writing data

Every writable resource takes the same three flags:

```sh
--set 'key=value'      # repeatable; keys are literal dotted names
--from-json <json>     # whole payload as JSON, @file, or @- for stdin
--dry-run              # validate locally and print the payload, send nothing
```

**Always `--dry-run` first.** Payloads are validated against the product's 107 entity schemas before anything leaves the machine, and every problem is reported at once. The server reports only one per round trip and collapses all of them into a single opaque `MD022 Bad Request`, so local validation is the difference between one attempt and five.

```sh
observeops credential create \
  --set 'credential.profile.name=core-snmp' \
  --set 'credential.profile.protocol=SNMP V1/V2c' \
  --dry-run
```

**`update` is a PATCH, not a replace.** Send only the keys being changed; everything else keeps its stored value. Required fields are therefore not enforced on update, but explicitly setting a required key to an empty value is still rejected, because that is clearing it rather than omitting it.

```sh
observeops monitor update web-01 --set 'object.name=web-01-dc2'   # nothing else changes
```

**Deletes prompt unless `-y/--yes` is passed.** Non-interactively, a delete without `--yes` exits 2 rather than guessing. Name resolution prefers an exact match; when only a partial match is found, the resolved name and id are always announced, including under `--yes`.

## Anything by name or id

Every command taking an id also accepts a name, resolved through search:

```sh
observeops monitor get web-01
observeops monitor get 900000000009
```

Exact matches win. Ambiguous names error and list the candidates rather than guessing.

## Inspecting large outputs (do not flood your context)

A monitor list can be thousands of rows and a widget definition hundreds of fields deep. Never read a whole response into the conversation. Project first, and save large payloads to a file:

```sh
# Project to the fields that matter
observeops monitor list --limit 250 --json --fields id,object.name,object.status

# Bare values for shell loops
observeops monitor list --fields id,object.name --raw | while read -r id name; do echo "$id"; done

# Persist, then query
observeops monitor list --limit 250 --json > /tmp/monitors.json
node -e 'const d=require("/tmp/monitors.json"); console.log(d.length)'
```

`--fields` fills missing keys with `null` rather than dropping them, so record shape stays stable across rows.

## Core commands at a glance

| Command | Purpose | Notes |
| --- | --- | --- |
| `auth login` | Password or PAT login, stored per profile | `--pat` mints a long-lived token for CI |
| `auth status` / `auth token` | Active identity and permissions / print the raw token | `token` writes to stdout for piping |
| `info` | Connectivity, credentials, server version | The health check; run it first |
| `profile list\|show\|use\|remove` | Work with several instances | `--profile <name>` overrides per command |
| `monitor list\|get\|status\|instances\|counts\|unhealthy` | Inventory and health | `unhealthy` is the fastest triage entry point |
| `monitor enable\|disable\|maintenance` | Change monitor state | Affects live monitoring; confirm first |
| `alert list` | Currently firing alerts | `--severity` filters |
| `alert ack\|clear\|suppress` | Act on alerts | EventBus-backed; see the caveat below |
| `discovery list\|get\|run\|abort\|result` | Device discovery | `run --follow` streams progress |
| `credential` / `group` / `user` / `tag` / `agent` | Supporting inventory | Full CRUD except `tag` |
| `credential template <protocol>` | Starter payload for SNMP, SSH, HTTP, JDBC, and more | Use it; a profile with no context cannot authenticate |
| `agent start\|stop\|restart\|reset` | Agent lifecycle | Needs a target or `--all` |
| `policy metric\|event\|netroute\|trace\|rum` | Alerting policies by data type | All list through search |
| `dashboard` / `widget` / `report` | Visualizations | `widget run` executes one; `widget references` says what depends on it |
| `explorer` / `noc-view` / `template` | Saved views and monitor templates | `template clone\|assign\|references\|resolve` |
| `report schedule\|recategorize` | Report actions that are not CRUD | `schedule` re-syncs a flag, it does not create a schedule |
| `metric timeseries\|aggregate` | Metric data over REST | No event stream needed, so a PAT works |
| `log query` / `flow query` / `trace query` / `rum query` | Query each product with filters | `--where` before aggregation, `--having` after |
| `log` / `flow` / `trace` / `rum` | What each product is observing | Sources, interfaces, services |
| `slo` / `compliance` | Objective and posture reporting | `slo status <cycle-id>`, `compliance status` |
| `ncm config list\|get\|backup\|diff` | Device configs, one of the products | `backup` prints raw config text; `ncm firmware`, `ncm checksum` |
| `ncm approval list\|get` | Change approvals | Approve and reject are deliberately not wrapped |
| `resource types\|describe\|list\|get\|create\|update\|delete` | Every type without a typed command | See the discovery section above |
| `log tail` | Live log tail, filtered server-side | Needs a password login; `--grep`, `--source`, `--pipeline` |
| `watch <resource>` | Live streams per resource | `alerts`, `discovery`, `agent`, `config`, `compliance`, `report`, `runbook`, `system`, `user`, `flow`, `events`, `types` |
| `query context` | Run a widget context over REST | Prefer this over `query run` |
| `query run <ui.action.*>` | Raw real-time query over the event stream | Advanced; needs a session id |
| `api <method> <path>` | Any endpoint, authenticated | `--search` for the paginated search form |
| `catalog` | Lookup vocabularies for filter values | `catalog` alone lists what is available |
| `config` | Local settings: host, TLS, default profile | Never stores credentials |
| `forget-column` | Remove a column name mapping | The only mutation under /misc; confirms first |
| `skills` | Install the bundled agent skills | `skills install --project\|--user`; confirms first |
| `docs` | Whole command catalog as JSON | Written for agents to read |
| `completion bash\|zsh\|fish` | Shell completion | Dynamic, suggests live resource types |

**`observeops <command> --help` is the source of truth for flags.** This table is a hint, not a spec.

## Filtering data queries

Two filters, applied at different times. Getting them the wrong way round returns the wrong rows
rather than an error, so it is worth stating precisely:

- `--where` runs **before** aggregation and compiles to a WHERE clause. It sees raw field values.
- `--having` runs **after** aggregation and compiles to a HAVING clause. It sees aggregated
  numbers, and its operands are result aliases with dots replaced by underscores.

```sh
observeops log query --data-point count --aggregator count \
  --where 'severity in ERROR,FATAL' --having 'count > 100' --group-by host
```

Operators: `=`, `!=`, `>`, `<`, `>=`, `<=`, `in`, `not in`, `contains`,
`starts with`, `ends with`. Text operators work only in `--where`. Repeated flags join with
AND; add `--where-any` or `--having-any` for OR. Mixed AND/OR needs an exported widget context.

Two things to know when reading `--dry-run` output. The server has **no not-equal for
pre-filters**, so `--where 'x != y'` is sent as an `equal` condition inside an `exclude`
group. And an unknown data source type returns empty rather than erroring, which is why the CLI
validates `--type` against the 75 known values locally.

### Helper lookups for visualizations

Most of the product's `/misc` endpoints exist to populate the pickers its own interface shows
while somebody builds a widget, dashboard or report. Every one of them is wrapped, and
`observeops catalog` lists them in two groups: values you can filter on, and helpers for
building.

```sh
observeops catalog                      # both groups
observeops catalog archived-status      # how far back a report can reach
observeops catalog column-mapping       # column display names for grids
observeops catalog topology-backgrounds # topology widget backgrounds
observeops ncm firmware                 # firmware images, with ncm checksum <file>
```

Reach for these when assembling a visualization, not only when filtering. A report that asks for
a window beyond the archive retention returns nothing, and `archived-status` is how you know.

### Finding operands and values

Never guess either side of an operator. Three sources, in order of reliability:

```sh
observeops widget get "<similar widget>" --json   # operands known to return data here
observeops catalog                                # fixed vocabularies: vendors, services, columns
observeops log values severity                    # values a field actually takes, by sampling
```

Only **indexed** columns are filterable (`observeops catalog columns`). Filtering an unindexed one
returns nothing and is indistinguishable from no data, which is the most common false negative.

`<product> values <field>` works by grouping over a window, because the product has no
distinct-values endpoint. A value missing from its output did not occur **in that window**; say
that rather than "the value does not exist".

Always `--dry-run` a filter the first time. Filtering is not yet verified against a live
instance.

## Actions that are not CRUD

A resource factory generates list, get, create, update and delete. Everything below is an action
it cannot generate, which is exactly why each one is easy to miss and worth naming.

| Action | What it really does |
| --- | --- |
| `template assign` | Binds a template to monitors. **Changes what is displayed for every monitor named** |
| `template clone` | Copies a template. Prefer this over editing one already in use |
| `template references` / `widget references` | What depends on this. Run before deleting |
| `template resolve` | Which template a monitor type inherits |
| `report schedule` | **Does not create a schedule.** Re-syncs the scheduled flag from scheduler entries |
| `report recategorize` | Moves every report in a category. Confirm the count first |
| `agent reset` | The most disruptive agent verb. Try `restart` first |
| `monitor enable\|disable\|maintenance` | Changes what is watched, and therefore who gets paged |
| `alert ack\|clear\|suppress` | EventBus-backed; the server does not confirm the effect |
| `forget-column` | Changes how a column is labelled everywhere it appears |

Two of these are commonly misread. `report schedule` sounds like it schedules a report and does
not: real schedules are a separate resource (`observeops resource list schedulers`). And
`references` is the only way to know whether deleting a widget will empty somebody's dashboard.

## Output contract and exit codes

- **stdout is data. stderr is everything else.** `observeops monitor list --json | jq` is always safe. Under `--json`, stdout is exactly one JSON document; under `watch --json` it is one JSON object per line.
- **Exit codes:** `0` success including empty results, `1` error, `2` usage error or cancellation, `4` authentication or permission failure, `8` pending or timed out while still running.
- `8` on a write means the outcome is genuinely unknown. Re-read the resource before retrying.
- Errors under `--json` are a single object with `reason`, `message`, `error.code`, and a `next` array of suggested commands.

Full detail in [references/output-contract.md](references/output-contract.md).

## Known server behavior worth knowing

These are the server's quirks, not the CLI's. The CLI works around each one, but they explain otherwise baffling results:

- **`MD022` means either a validation failure or a permission denial**, distinguished only by the message text. An unknown API path therefore reports as a permission error and exits 4, not 404.
- **`GET /settings/event-policies` returns 500.** All policy listing goes through search instead.
- **Updating a credential profile that has no `credential.profile.context` hangs** until the CLI times out with exit 8. Sending a context in the update heals it. Always create credential profiles from `credential template`.
- **`/settings/schedulers` times out** through both routes.
- **Session tokens live ten minutes.** The CLI refreshes silently; a PAT avoids the issue entirely for automation.
- **Data querying has two paths.** `widget run`, `query context` and `metric` use plain REST and
  work with a PAT. Only `watch` and `query run` need the event stream and therefore a password
  login. Reach for REST first.
- **The product commands are unverified.** `metric`, `log`, `flow`, `trace`, `rum`, `slo`,
  `compliance`, `widget run` and `query context` were built from the server's route definitions
  but not yet exercised live. Use `--dry-run` on `metric`, and treat an unexpected failure as
  possibly a CLI bug rather than assuming the user is wrong.

## Safety rules for autonomous use

1. **Run `observeops info` before anything else.** Do not diagnose failures without knowing the server version and auth state.
2. **`--dry-run` every create and update** before running it for real.
3. **`resource describe <type>` before writing an unfamiliar type.** Do not guess field names; they are discoverable.
4. **Confirm with the user before changing monitored state.** `monitor disable`, `monitor maintenance`, `agent stop`, and any policy edit affect live monitoring and paging.
5. **Never print or commit tokens.** `auth token` writes a live credential to stdout; use it only to pipe into another command.
6. **Treat `--insecure` as deliberate.** On-premise instances usually serve self-signed certificates, so it is often required, but never add it silently to a command targeting a host the user did not name.
7. **Alert actions are EventBus-backed and their effect is unconfirmed.** `alert ack`, `clear`, and `suppress` report only what the server acknowledged receiving. Verify with `alert list` afterwards.

## MCP server (persistent sessions)

The same binary serves the platform as Model Context Protocol tools: `observeops mcp` on
stdio. When the host supports MCP, prefer registering the server over shelling out per
command: one authenticated session spans the whole conversation (no process start or token
resolution per call), results are structured JSON instead of parsed stdout, and inputs are
schema-validated before anything is sent.

```sh
claude mcp add observeops -- observeops mcp        # register with Claude Code
observeops mcp --read-only                          # no write tools at all
observeops mcp --allow-raw                          # add the raw REST/EventBus escape hatches
```

The tool catalog mirrors this skill's command map (`whoami`, `search_monitors`,
`search_alerts`, `resource`, `query_metrics`, `watch_events`, ...). The same safety rules
apply, enforced by the tools themselves: writes validate locally and support `dry_run`,
deletes refuse fuzzy name matches, and alert actions report only what was observed. A
personal access token cannot open the event stream; the streaming tools then return
`capability_missing` naming the fix rather than hanging.

## Templates

Copy-ready artefacts, in `templates/`. Lift them rather than re-deriving:

- `templates/ci-health-check.sh`: a build gate. PAT auth, isolated config, fails on unhealthy
  monitors, branches on exit codes rather than parsing output.
- `templates/bulk-update.sh`: the safe bulk pattern. Build the list, dry-run the batch, confirm,
  apply, and collect exit-8 items separately because their outcome is unknown.
- `templates/query-context.json`: a filtered query context with both filter blocks filled in,
  showing the exact shape `data.filter` and `result.filter` take.

## References

- [references/auth.md](references/auth.md): login modes, PATs, profiles, token lifetime, TLS, precedence order.
- [references/recipes.md](references/recipes.md): copy-pasteable patterns for triage, inventory, and bulk edits.
- [references/output-contract.md](references/output-contract.md): exit codes, JSON shapes, `--fields`, NDJSON, error envelopes.
