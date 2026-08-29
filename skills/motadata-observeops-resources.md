---
name: observeops-resources
description: >-
  Create, update, and delete any of the 96 Motadata ObserveOps resource types
  safely, using the product's own entity schemas for local validation. Use when
  the user wants to change configuration in Motadata rather than read it:
  "create a monitor", "add a credential profile", "set up an SLO", "edit a
  policy", "bulk update monitors", "what fields does X need", "why is Motadata
  rejecting this payload", or any MD022 error. Covers dotted literal keys,
  required fields and enums, PATCH semantics, prerequisites, and the resource
  catalog for types with no typed command.
license: UNLICENSED
---

# Motadata resources and schemas

Writing to ObserveOps is safe and predictable once three things are understood: keys are literal, payloads are validated locally before they are sent, and update is a patch. Everything else follows from those.

This skill assumes the `observeops` binary is available and authenticated. See the `observeops-cli` skill for setup.

## Dotted keys are literal

`object.name` is one key whose text is `object.name`. It is not a path into a nested object, and nesting it produces a payload the API rejects.

```sh
observeops monitor create --set 'object.name=web-01'          # correct
observeops monitor create --set 'object={"name":"web-01"}'    # wrong
```

The only genuinely nested values are `*.context` map fields, which take real JSON:

```sh
--set 'credential.profile.context={"username":"svc","password":"..."}'
```

## Two different vocabularies

Configuration fields and query operands are **not the same thing**, and confusing them wastes
turns. `resource describe <type>` gives the fields of a stored entity. Filter operands are
columns in the data store, discovered separately:

```sh
observeops resource describe objects   # configuration fields, for create and update
observeops catalog columns             # query operands, for --where
```

A schema property is not automatically filterable.

## Discover the shape before writing

```sh
observeops resource describe <type>            # required fields, enums, custom routes
observeops resource describe <type> --json     # every property, not just the first twenty
observeops resource types                      # all 96 types
observeops resource types --untyped            # those with no typed command
```

`describe` prints the required fields and, for enum properties, the exact accepted values. Read it before composing any unfamiliar payload. Guessing field names wastes round trips against a server that reports one error at a time.

## Local validation, and why it exists

The CLI bundles the product's 107 entity schemas and mirrors the server's own rule engine, including its message wording. It reports **every** problem at once:

```
✗ 2 problem(s) with the monitor payload:
  Missing information: Host is a required field  (object.host, rule: required)
  Missing information: Group(s) is a required field  (object.groups, rule: required)
```

The server, by contrast, collapses every validation failure into a single opaque `MD022 Bad Request` and stops at the first bad property. A five-field mistake is five round trips server-side and one locally. Local validation also adds the thing the server withholds: the list of allowed values for an enum.

```sh
observeops credential create --set 'credential.profile.protocol=BAD' --dry-run
# Value of Protocol is invalid — allowed: "SNMP V1/V2c", "SNMP V3", "SSH", ...
```

**Validation is advisory, not authoritative.** The schemas describe the build they were generated from. If a payload is rejected locally that is genuinely valid, `observeops api` sends it unvalidated. That escape hatch is intentional.

## Always dry-run first

```sh
observeops <type> create --set '...' --dry-run     # validates, prints the payload, sends nothing
```

`--dry-run` prints the exact JSON that would be sent, which is also the fastest way to check that `--set` parsed a value the way you intended.

## Rules the schemas encode

| Rule | Meaning | Checked locally |
| --- | --- | --- |
| `required` | Must be present and non-blank on create | Yes |
| `unique` | No other row may share the value | No, needs the server |
| `pattern` | Must match a regex | Yes |
| `minimum` / `maximum` / `range` | Numeric bounds | Yes |
| `range-divisible` | Bounds plus a step | Yes |
| enum (`values`) | One of a fixed list | Yes, with values listed on failure |
| `prerequisites` | Only applies when another field has a given value | Yes, see below |
| `private` | Server-managed, cannot be supplied | Only when asked |

### Prerequisites make a field forbidden, not optional

If `scheduler.days` declares a prerequisite of `scheduler.timeline = Weekly`, then sending `scheduler.days` while the timeline is anything else is rejected outright. The CLI strips such fields rather than letting the request fail, and says so:

```
Not applicable and will be removed: scheduler.days (prerequisites not met)
```

On update the rule relaxes: a gate field absent from a patch is assumed satisfied, because the stored entity may already satisfy it.

### `*.context` is undeclared

Schemas do not describe the interior of `credential.profile.context`, `policy.context`, `object.context`, or any other map field. They are reported as unknown fields, which is never an error. For credentials, start from a template:

```sh
observeops credential template ssh
observeops credential template snmp-v2c > profile.json
```

## Update is a PATCH

Send only what changes. Everything else keeps its stored value.

```sh
observeops monitor update web-01 --set 'object.name=web-01-dc2'
```

Consequences:

- Required fields are **not** enforced on update. A one-field change is legal.
- Setting a required key to an empty value **is** rejected: that is clearing it, not omitting it.
- Password fields are required on create and ignored on update; the server keeps the stored secret.
- Do not re-send a whole entity fetched with `get`. It is unnecessary and may include server-managed fields.

## Deletes

```sh
observeops <type> delete <id-or-name>          # prompts interactively
observeops <type> delete <id-or-name> --yes    # non-interactive
```

Without `--yes` and without a TTY, the command exits 2 rather than guessing. Name resolution prefers exact matches; if only a partial match is found, the resolved name and id are announced before the delete proceeds, including under `--yes`. Read that line: `delete foo` can resolve to `foo-production`.

## The long tail

Roughly fifteen resources have a typed command; the rest are reached generically with identical semantics:

```sh
observeops resource list slo-profiles
observeops resource get slo-profiles gold-tier
observeops resource create slo-profiles --set 'slo.profile.name=gold' --dry-run
observeops resource update slo-profiles gold --set 'slo.profile.type=Availability'
observeops resource delete slo-profiles gold --yes
```

Listing prefers `POST /search` where a type has it, because several types answer a plain GET with an empty array and one answers 500. Use `--via get` when that guess is wrong.

Three of the product's router classes are absent from the catalog by design (`tags`, `misc`, `query`): they expose only custom routes and are already covered by `observeops tag`, `monitor`/`alert`, and `query`.

## Visualization resources behave the same way

`dashboard`, `widget`, `report`, `explorer`, `noc-view` and `template` are ordinary
schema-backed resources: same `--set`, same `--dry-run`, same PATCH semantics. Two differences
worth knowing:

- Their layout and content fields are only **partly described** by the schema, so read an existing
  one with `--json` and adapt it rather than composing from scratch.
- They are shared artefacts. A dashboard edit is visible immediately; a NOC view is on a wall
  somewhere; `template assign` changes what every named monitor displays. Confirm before writing,
  and run `references` before deleting.

## Bulk changes

There is no bulk verb. Drive a loop with projected output, and dry-run the whole batch first:

```sh
observeops monitor list --filter dc1 --fields id,object.name --raw > /tmp/targets.tsv

while IFS=$'\t' read -r id name; do
  observeops monitor update "$id" --set 'object.tags=[10000000000005]' --dry-run
done < /tmp/targets.tsv
```

Only after the dry-run output looks right should the `--dry-run` be dropped. Bulk edits against monitoring configuration are high-consequence: confirm the target list with the user before applying.

## Regenerating the schemas

The bundled schemas and resource catalog are generated from a specific product build:

```sh
npm run generate      # regenerates both from the product source
npm test              # the suite asserts against the real bundle
```

Do this after every product upgrade. A stale bundle produces confidently wrong local errors, which is worse than no validation at all.

## Diagnosing a rejection

1. `--dry-run` the payload. Most problems surface without touching the network.
2. `observeops resource describe <type>` and compare field names exactly, including dots.
3. If the server returns `MD022` with "Unauthorized access", it is a permission problem or a wrong path, not a payload problem. Exit code 4 rather than 1 confirms this.
4. If local validation and the server disagree, regenerate the schemas, then fall back to `observeops api` to send the payload unvalidated.
