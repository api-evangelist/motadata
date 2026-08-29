---
name: observeops-triage
description: >-
  Diagnose a live incident on a Motadata ObserveOps instance: find what is
  firing, decide whether it is real, isolate the blast radius, and hand back a
  written summary. Use when the user says "what is broken", "why is this device
  down", "triage these alerts", "is this a real outage", "what changed", "why is
  my monitor unhealthy", "nothing is polling", or pastes an alert and asks what
  it means. Encodes the order to run commands in and how to read the answers.
license: UNLICENSED
---

# Motadata incident triage

A triage pass answers four questions in order: is the instance itself healthy, what is firing, is it real, and how far does it reach. Running commands in that order avoids the most common mistake, which is investigating a device problem that is actually a collection problem.

Requires the `observeops` binary, authenticated. See the `observeops-cli` skill.

## The loop

```sh
# 1. Is the tool telling the truth?
observeops info

# 2. Fleet state in one number per bucket
observeops monitor counts

# 3. What is firing
observeops alert list --severity CRITICAL
observeops alert list                      # everything, if CRITICAL is empty

# 4. What is failing to poll, which is a different question
observeops monitor unhealthy

# 5. Drill into one device
observeops monitor status <name>           # per-plugin last-poll ages
observeops monitor get <name>              # configuration and current state
observeops monitor instances <name>        # interfaces, disks, sub-entities
```

Steps 3 and 4 answer different questions and often disagree. An alert means a policy threshold was crossed. An unhealthy monitor means the data stopped arriving. A device with no data cannot breach a threshold, so a silent device may show zero alerts while being completely down.

## Reading the answers

### `observeops info`

If this fails, stop. Every later result is untrustworthy. Exit 4 means re-authenticate; a network error means the instance or the path to it is the problem, which may itself be the incident.

Note the reported server version. Behavior differs between 8.2.7 and 10.0.0, most visibly in `monitor counts`, whose buckets are version-dependent. Never assume a fixed set of status keys.

### `observeops monitor counts`

Counts grouped by status. Render whatever comes back rather than expecting specific buckets. A sudden mass shift into `UNKNOWN` or `UNREACHABLE` points at collection infrastructure, not at the devices.

### `observeops alert list`

Severity comes from the `severity` field. Filter with `--severity CRITICAL`, `MAJOR`, `WARNING`, `CLEAR`, `DOWN`, or `UNREACHABLE`.

```sh
observeops alert list --json --fields severity,object.name,message,timestamp
```

A wall of alerts sharing one timestamp is usually one cause. Group by time before treating them as separate incidents.

### `observeops monitor unhealthy`

The fastest path to "why is there no data". Then, per device:

```sh
observeops monitor status core-switch-1
```

This shows the last successful poll per plugin. Interpretation:

- **One plugin stale, others fresh**: a credential or protocol problem for that plugin, not a dead device.
- **Every plugin stale**: the device is unreachable, or the collecting agent is down.
- **All devices behind one agent stale**: check the agent, not the devices.

```sh
observeops agent list                     # state and last contact per agent
```

## Distinguishing collection failure from a real outage

Work down this list. Stop at the first one that explains the symptom.

1. **Is the agent alive?** `observeops agent list`. An agent that is down takes every device behind it with it.
2. **Did credentials change?** `observeops credential list` shows an "in use" count. A recently edited profile that many monitors depend on is a strong suspect.
3. **Is the device in maintenance?** `observeops monitor get <name>` and read `object.state`. A monitor in `MAINTENANCE` or `DISABLE` is silent on purpose.
4. **Did a policy change?** `observeops policy metric list` and the other four policy types. A newly edited threshold explains a sudden flood of alerts with no underlying change.
5. **Only then treat it as a device problem.**

## Watching it happen

When the question is "is this still happening" rather than "what happened":

```sh
observeops watch alerts --timeout 120
observeops watch events --timeout 60        # everything, including discovery and diagnostics
```

Both need a password login, not a PAT. Reaching the timeout is normal and exits 0. Run these in the background rather than blocking, and remember that the stream shows **new** activity only: `alert list` remains the way to see current state.

## Acting on alerts

```sh
observeops alert ack <id>
observeops alert clear <id>
observeops alert suppress <id>
```

These go over the EventBus. The server acknowledges receipt but does not confirm the effect, so the CLI reports only what it observed. **Always verify with `observeops alert list` afterwards** rather than assuming the action landed. Confirm with the user before acknowledging alerts in bulk: an acknowledged alert stops paging someone.

## Changing state during an incident

`monitor disable`, `monitor maintenance`, and `agent restart` all affect live monitoring and paging. Confirm with the user first, state what will stop being watched, and prefer `maintenance` over `disable` for a planned window since it preserves intent.

```sh
observeops monitor maintenance core-switch-1
```

## Digging into product data

When the monitor-level view is not enough, query the product directly. These are REST, so a
personal access token works:

```sh
observeops log query --data-point count --aggregator count \
  --where 'severity in ERROR,FATAL' --group-by host --from -6h
observeops trace query --data-point duration --aggregator avg --where 'service = checkout'
```

Discover operands with `observeops catalog columns` and values with
`observeops log values <field>` rather than guessing them mid-incident.

## Writing the handoff

Finish a triage pass with a summary the next person can act on. Include:

- What is firing, grouped by probable cause rather than listed one alert per line.
- Whether data is flowing, from `monitor unhealthy` and `monitor status`, and since when.
- The blast radius: how many monitors, behind which agent, in which groups.
- What was ruled out, and by which command.
- What was changed, if anything, and how to reverse it.
- Exactly what remains unknown. Do not present a plausible cause as a confirmed one.

Save the raw evidence rather than pasting it whole:

```sh
observeops alert list --json > /tmp/alerts.json
observeops monitor unhealthy --json > /tmp/unhealthy.json
```

Then quote the few fields that matter. Reading entire payloads into the conversation costs context for no benefit.
