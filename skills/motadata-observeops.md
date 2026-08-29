---
name: observeops
description: >-
  Motadata ObserveOps router. Use when the user mentions Motadata or ObserveOps:
  operating the CLI, logging in, listing or creating monitors, triaging alerts
  and outages, running discoveries, managing credential profiles and policies,
  editing any configuration resource, backing up network device configs,
  streaming live events, or running real-time metric and log queries.
  Automatically routes to the specific skill for the task.
license: UNLICENSED
metadata:
  version: 1.0.0
---

# Motadata skills router

Motadata ObserveOps is an on-premise observability platform. The `observeops` CLI is the supported way for an agent to work with it: it handles login and token refresh, self-signed TLS, the response envelope, pagination, name-to-id resolution, and local payload validation.

## First, always

```sh
observeops --version
observeops info          # connectivity, credentials, server version
```

If `info` fails, fix that before anything else. Every later result is untrustworthy until it passes.

Supported server versions are 8.2.7 and later, verified against 8.2.7 and 10.0.0. Behavior differs between them, so read the version `info` reports rather than assuming.

---

## By task

**Operating the CLI at all** → use `observeops-cli`
- Installing, authenticating, profiles for several instances
- The command tree and what each group covers
- Output contract: stdout versus stderr, exit codes, `--json`, `--fields`
- The raw `api` escape hatch

**Something is broken right now** → use `observeops-triage`
- What is firing, and whether it is real
- Telling a collection failure apart from a device outage
- Blast radius, agents, credentials, recent policy changes
- Writing a handoff someone else can act on

**Changing configuration** → use `observeops-resources`
- Creating and updating monitors, credentials, policies, SLOs, and the other 90-odd types
- Required fields, enums, prerequisites, and why `MD022` is unhelpful
- Dotted literal keys, PATCH semantics, dry runs, bulk edits
- The resource catalog for types with no typed command

**Live data and queries** → use `observeops-eventbus`
- `watch alerts` and `watch events`
- Why the stream needs a password login rather than a PAT
- NDJSON output and long-running command handling
- Real-time metric and log queries, and getting a working query context from a saved widget

---

## Three facts that prevent most mistakes

**1. Dotted keys are literal.** `object.name` is one key, not a path. This is why `--fields object.name` works and `jq '.object.name'` returns null. Full explanation in `observeops-resources`.

**2. stdout is data, stderr is everything else.** An empty stdout with exit 0 means "no results", not "failure". Exit codes are `0` success, `1` error, `2` usage or cancellation, `4` auth or permission, `8` pending or timed out with an unknown outcome.

**3. `MD022` means either a validation failure or a permission denial**, told apart only by the message text. An unknown API path reports as a permission error and exits 4, so check the path before chasing a missing grant.

---

## Choosing between typed and generic commands

Roughly fifteen resources have a hand-written command with richer output and extra verbs. The other eighty are reached through `observeops resource`, with identical validation and name resolution.

```sh
observeops resource types --untyped     # what has no typed command
observeops resource describe <type>     # required fields and allowed values for any type
```

Prefer the typed command when one exists. `observeops resource types` marks which types have one.

---

## Safety

Monitoring configuration is load-bearing: disabling a monitor stops someone being paged. Before any state change, confirm with the user, and prefer the reversible option. `--dry-run` exists on every create and update and costs nothing.
