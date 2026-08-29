---
name: observeops-eventbus
description: >-
  Stream live events from a Motadata ObserveOps instance and run real-time
  queries over its Vert.x EventBus bridge. Use when the user says "watch
  alerts", "stream events", "show me live activity", "tail observeops", "run a
  metric query", "query logs", "why is watch not showing anything", or needs
  data that only the websocket path exposes. Covers the session-id requirement,
  NDJSON output, the widget-context route into the query DSL, and the failure
  modes that make the stream look silent.
license: UNLICENSED
---

# Motadata EventBus and live queries

ObserveOps exposes a Vert.x SockJS EventBus bridge over a raw websocket. It carries live alert and status activity, and it is the only transport for streaming events as they happen. The CLI wraps it as `watch` and `query run`.

It is **not** the only way to query metric, log, flow, or trace data. Those are also reachable over plain REST, which is usually the better choice: see the next section.

Requires the `observeops` binary, authenticated. See the `observeops-cli` skill.

## Prefer REST when you only need data

The event stream is not the only way to query. Three commands return the same data over plain
HTTP, need no session id, and therefore work with a personal access token:

```sh
observeops widget run "CPU Trend"                      # a saved widget, any data type
observeops query context --context @context.json       # a widget context
observeops metric timeseries --data-point <name>       # metric data directly
observeops log query   --data-point count --where '...'  # logs, filtered
observeops trace query --data-point duration --where '...'  # APM traces, filtered
observeops rum query   --data-point count --where '...'  # RUM sessions, filtered
observeops flow query  --data-point bytes --where '...'  # flow, filtered
```

Use the event stream only for live streaming (`watch`) or for actions other than a
visualization render. If the task is "get me this data", reach for REST first.

Filters go on those commands: `--where` before aggregation, `--having` after. Discover what to
put in them with `observeops catalog` and `<product> values <field>` rather than guessing.

When building or editing a widget, dashboard or report, `observeops catalog` also carries the
helper lookups the product's own interface uses: `archived-status` for how far back data is
queryable, `column-mapping` for grid column names, and `topology-backgrounds`.

These REST paths were built from the server's route definitions and are not yet verified against
a live instance; report failures rather than assuming the user's input is wrong.

## The one prerequisite that catches everyone

**The EventBus needs a password login, not a PAT.** Opening the bridge requires a session id, and a session id is minted only by `auth login` with a username and password. A Personal Access Token authenticates every REST call perfectly and cannot open the stream at all.

```sh
observeops auth status          # token.type must be "session", not "pat"
```

If `watch` fails to connect while every other command works, check this first.

## Streaming

The server broadcasts **twenty-four** event types, not two. There is a subcommand per resource
group, so you never have to know the event names:

```sh
observeops watch types            # the vocabulary, grouped by resource
observeops watch discovery        # progress, probes, statistics, state changes
observeops watch report           # report generation progress
observeops watch config           # device config changes and approvals
observeops watch system           # disk, datastore, processor health
observeops watch agent            # registration and upgrades
observeops watch compliance | runbook | user | flow
```

Prefer these over `events --type`: the group filters are exact matches on a generated
vocabulary, while `--type` is a substring guess.

`watch discovery` is the one worth reaching for unprompted. A running discovery emits probe
results and statistics that polling `discovery get` never shows.

```sh
observeops watch alerts                        # alert activity
observeops watch events                        # everything the server emits
observeops watch alerts --timeout 120          # stop after N seconds
observeops watch alerts --json                 # NDJSON, one object per line
observeops watch events --raw                  # include the full event context
```

Behavior worth knowing:

- Event lines go to **stdout**; the banner and the final count go to stderr.
- `--json` emits **NDJSON**: one compact JSON object per line, never an array, so events can be consumed as they arrive.
- Reaching a `--timeout` is normal completion and exits **0**. The deadline is the stop condition, not a failure.
- Ctrl-C closes the socket cleanly, prints the count, and exits 0.
- Both commands are long-running. In an agent, run them in the background or always pass `--timeout`.

```sh
observeops watch alerts --json --timeout 60 | while read -r line; do
  echo "$line" | jq -r '.["event.type"]'
done
```

## Live tail is not the same as watch

`watch` subscribes to notifications the server broadcasts regardless. `log tail` **negotiates**:
it asks the server to start streaming log lines, and stops it on exit.

```sh
observeops log tail --timeout 60
observeops log tail --grep timeout --grep refused --match any
observeops log tail --source web-01 --json
observeops log tail --pipeline          # pre-ingestion pipeline stage
```

Filtering is **server-side**: `--grep` and `--source` reduce what is sent, not what is shown.
Recommend them over piping to grep, especially on a busy instance.

If the user wants log lines, this is the command. `watch` will never show them, because log
lines are not among the twenty-four broadcast notification types.

The same mechanism carries `Trap Tail`, `Log Pipeline Tail`, `Event Engine Stats` and
`Event Tracker`.

## Why the stream looks silent

A quiet instance is genuinely quiet: these commands show **new** activity only, never current state. Use `observeops alert list` for what is firing right now.

To prove the transport works rather than guessing, generate activity in another shell while watching:

```sh
observeops watch events --timeout 30 &
observeops monitor disable <name> && observeops monitor enable <name>
```

State changes reliably produce `ui.notification.user.notification` frames, and policy evaluation produces `compliance.policy.state.change`. Seeing either confirms the bridge end to end.

## Protocol notes

The CLI handles all of this; it is documented because it explains the failure modes.

- **Endpoint**: `wss://<host>/eventbus/websocket`, raw websocket, not SockJS framing.
- **Inbound** messages are addressed to `server.event`. Nothing else is permitted inbound.
- **Outbound** arrives on `ui.event` and `user.event.<session-id>`.
- **The session id must appear at both body levels** of an emitted frame. Omitting either produces silence rather than an error.
- **A ping is required roughly every five seconds.** Miss it and the server closes the connection with `SOCKET_IDLE`, which looks like an unexplained disconnect.
- **Real event payloads are Snappy-compressed** and base64-encoded, flagged by `compression.type: 1`. Uncompressed frames also occur, so both must be handled.

## Real-time queries

`query run` sends a `ui.action.*` request and streams the response frames.

```sh
observeops query run ui.action.visualization.render --context @context.json
observeops query run <event-type> --context '<json>' --timeout 60 --all
```

`--all` collects every frame until the timeout instead of stopping at the first. Without a valid context the server usually returns nothing rather than an error, so an empty result means "the context was wrong" more often than "there is no data".

## Getting a context that actually works

The query DSL is undocumented, but it does not need to be reverse-engineered: **a saved widget is a complete query context**, and a typical instance carries thousands of them covering every data type.

```sh
observeops widget list --filter cpu
observeops widget get <id> --json > context.json
```

A widget definition contains exactly what the render action consumes:

```json
{
  "visualization.type": "Availability Time Series",
  "visualization.timeline": { "relative.timeline": "today" },
  "visualization.data.sources": [
    {
      "type": "availability",
      "filters": { "data.filter": {}, "result.filter": {} },
      "data.points": [
        { "data.point": "monitor.uptime.percent", "aggregator": "avg", "entity.type": "monitor", "entities": [] }
      ]
    }
  ]
}
```

The workflow is therefore: find a widget that already asks a similar question, edit its context, and replay it.

```sh
observeops widget get <id> --json > context.json
# narrow the timeline, swap the data point, or fill in entities
observeops query run ui.action.visualization.render --context @context.json
```

Useful fields to edit: `visualization.timeline` for the window, `data.points[].data.point` for the metric, `data.points[].entities` to scope to specific monitor ids, and `filters.data.filter` to constrain the source set.

## Alert actions

`alert ack`, `alert clear`, and `alert suppress` travel over the same bridge. The server replies with a notification counter rather than a named confirmation event, so the CLI reports only that the server registered activity. It does not claim the action took effect.

**Always verify with `observeops alert list` afterwards.** The transport is proven; the effect is not, and the CLI is deliberately honest about the difference.

## Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| Connects, then drops after a few seconds | Missed ping; the server closed it as `SOCKET_IDLE` |
| Never connects, REST works fine | Logged in with a PAT; no session id exists |
| Connects, stays silent | Nothing is happening. Generate activity to confirm |
| `query run` returns nothing, no error | The context is wrong. Start from a real widget |
| Garbled or skipped frames | Snappy-compressed payload not being decoded |

Add `--debug` to any of these commands to trace connection, frames, and event types on stderr without touching stdout.
