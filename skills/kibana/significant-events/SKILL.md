---
name: kibana-significant-events
description: >
  Search and triage Significant Events and Knowledge Indicators (KIs) on Kibana
  Streams via the native Agent Builder tools (`platform.sig_events.*`). Use when the
  user asks about significant events or SigEvents, wants to see what's currently
  promoted/acknowledged/open, wants to search Knowledge Indicators (feature or query
  KIs) for a stream, needs to promote/acknowledge/demote/resolve an event, or wants a
  stream's SigEvent occurrence trend. Creation tools are also covered for the rare
  case a conversation needs to save a new KI or event by hand.
metadata:
  author: elastic
  version: 0.1.0
compatibility: Kibana 9.1+ (`streams` + `significant_events` plugins) with Agent
  Builder 9.2+
---

# Kibana Significant Events

Significant Events ("SigEvents") is Streams' detection and triage layer: rule-backed ES|QL queries attached to a
stream fire, get correlated by a Discovery agent, and surface as triaged **significant events** a human or agent can
act on. Knowledge Indicators (KIs) are the supporting context — durable facts about a stream (**feature** KIs, e.g.
detected technologies/dependencies) and the detection queries themselves (**query** KIs, the SigEvent definitions).

**In normal operation, KIs and significant events are created automatically** — KIs by the scheduled KI-extraction
pipeline, significant events by the Discovery pipeline correlating rule firings. This skill is primarily about
**reading and triaging** that output: searching KIs for context, searching events to see what's open, and moving
events through their lifecycle. The creation tools exist for the exception, not the rule — use them only when a
conversation surfaces something the automated pipelines haven't captured yet.

This skill relies on the native **Agent Builder tools** built into Kibana's `significant_events` plugin. Agent
Builder access is a hard prerequisite, not optional.

## Concepts

- **Query KI** — a SigEvent definition: an ES|QL detection query plus `title`, `severity_score`, and (if rule-backed)
  the `alerting_v2` `rule_id` that runs it on a schedule.
- **Feature KI** — a durable fact about a stream discovered by the KI-extraction pipeline (detected technology,
  dependency, entity, infra component, schema). Used as context, not as a detector.
- **Significant event** — a triaged, human-facing incident synthesized by the Discovery pipeline from one or more
  firing queries. Has a lifecycle: `promoted` → `acknowledged` or `demoted` → `resolved`. `demoted` means triage
  decided it's not real; `resolved` means it was real and is now handled.

## Task selector

| Task                                                            | Tool                                                                    |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------ |
| Search KIs (feature and/or query) for one or more streams        | `platform.sig_events.ki_search`                                        |
| See what significant events are currently open/promoted          | `platform.sig_events.event_search`                                     |
| Promote/acknowledge/demote/resolve an existing event              | `platform.sig_events.event_status_update`                              |
| Record an investigation run against an event                     | `platform.streams.sig_events.event_investigation_attach`               |
| Read one known stream's SigEvent definitions + occurrence trend   | Streams REST API — `GET /api/streams/{name}/significant_events` (see [Also available](#also-available)) |
| *(rare)* Save a detection query or stream fact a pipeline missed  | `platform.sig_events.ki_query_create` / `ki_feature_create`            |
| *(rare)* Hand-create a significant event                          | `platform.sig_events.event_create`                                     |

## Prerequisites

| Item               | Description                                                                              |
| ------------------- | ------------------------------------------------------------------------------------------ |
| **Kibana URL**      | Kibana endpoint (e.g. `https://localhost:5601` or a Cloud deployment URL)                 |
| **Authentication**  | API key or basic auth (see the `elasticsearch-authn` skill)                               |
| **Agent Builder access** | Required — this skill relies entirely on the native `platform.sig_events.*` tools; read access to Agent Builder plus `read_stream` (read) / `manage_stream` (KI writes) for the target streams |

## Using the tools

These tools are built into Kibana's `significant_events` plugin (namespace `platform.sig_events.*` plus
`platform.streams.sig_events.event_investigation_attach`) — no setup or registration needed, unlike custom Agent
Builder tools. Run them with `POST {KIBANA_URL}/api/agent_builder/tools/_execute`, body
`{"tool_id": "...", "tool_params": {...}}`:

```bash
curl -X POST "${KIBANA_URL}/api/agent_builder/tools/_execute" \
  -H "Authorization: ApiKey <base64-api-key>" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{
    "tool_id": "platform.sig_events.ki_search",
    "tool_params": {"kind": ["query"], "limit": 50}
  }'
```

An equivalent, session-less **MCP** path exists at `POST {KIBANA_URL}/api/agent_builder/mcp` (stateless — one
self-contained JSON-RPC request per call, no `initialize` handshake required):

```bash
curl -X POST "${KIBANA_URL}/api/agent_builder/mcp" \
  -H "Authorization: ApiKey <base64-api-key>" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"platform_sig_events_ki_search","arguments":{"kind":["query"]}}}'
```

Note the MCP tool names use underscores (`platform_sig_events_ki_search`), while `tools/_execute` and every other path
use the dotted id (`platform.sig_events.ki_search`). List everything available first with `GET
{KIBANA_URL}/api/agent_builder/tools` (or MCP `tools/list`) — for generic Agent Builder tool CRUD/testing, see the
`kibana-agent-builder` skill's `agent-builder.js` script, which wraps the same `tools/_execute` endpoint.

### Tool reference — read & triage (the common case)

| Tool                                                       | Purpose                                                        | Required params                                                                | Notes                                                                 |
| ------------------------------------------------------------ | ----------------------------------------------------------------- | ---------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| `platform.sig_events.ki_search`                             | Search KIs (feature and/or query) with optional semantic ranking | none                                                                              | `stream_names?`, `search_text?` (semantic), `kind?: ["feature"\|"query"]`, `limit` (default 50) |
| `platform.sig_events.event_search`                          | Search triaged significant events                                | none                                                                              | `query?` (title text), `stream_name?`, `status?: [...]`, `page`, `per_page` |
| `platform.sig_events.event_status_update`                  | Promote/acknowledge/demote/resolve an existing event               | `event_id`, `status: promoted\|acknowledged\|demoted\|resolved`                  | The everyday triage action — most "manage a significant event" requests are this, not a new event |
| `platform.streams.sig_events.event_investigation_attach`   | Record an investigation run on an event                            | `event_id`, `workflow_execution_id`, `status: pending\|success\|failed`, `started_at` | Call once at start (`pending`) and again at completion (`success`/`failed`, add `completed_at`) |

### Tool reference — creation (rare)

KIs are normally populated by the scheduled KI-extraction pipeline, and significant events by the Discovery pipeline
correlating rule firings — not by hand. Reach for these only when a conversation surfaces something neither pipeline
has captured yet, and always `ki_search` / `event_search` first to make sure it's genuinely missing.

| Tool                                                       | Purpose                                                        | Required params                                                                | Notes                                                                 |
| ------------------------------------------------------------ | ----------------------------------------------------------------- | ---------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| `platform.sig_events.ki_query_create`                       | Persist a new query KI (detection definition)                    | `stream_name`, `title`, `esql.query`                                             | `severity_score?`, `description?`, `evidence?[]`, `expires_at?` (omit for a durable KI) |
| `platform.sig_events.ki_feature_create`                     | Persist a new feature KI (stream fact)                            | `id`, `stream_name`, `type`, `description`, `properties`, `confidence` (0–100)    | `subtype?`, `evidence?[]`, `evidence_doc_ids?[]`, `tags?[]`, `filter?`, `expires_at?` |
| `platform.sig_events.event_create`                          | Hand-create a significant event                                   | `title`, `summary`, `root_cause`, `stream_names[]`, `criticality` (0–100), `confidence` (**0–1**), `recommendations[]` | `status?` defaults per backend; distinct 0–1 confidence scale vs. feature KIs' 0–100 |

Full JSON Schemas for every tool: [references/tool-schemas.md](references/tool-schemas.md).

## Also available

If you already know the stream name and only need its SigEvent definitions plus their firing-rate time series (not
feature KIs, not triaged event instances), the public (though deprecated) Streams endpoint does that one join for
you — see the `kibana-streams` skill for this and the rest of the Streams API surface:

```bash
curl -G "${KIBANA_URL}/api/streams/<stream-name>/significant_events" \
  -H "Authorization: ApiKey <base64-api-key>" \
  --data-urlencode "from=2026-07-07T00:00:00Z" \
  --data-urlencode "to=2026-07-08T00:00:00Z" \
  --data-urlencode "bucketSize=1h"
```

## Examples

### "What's currently open for triage?"

```bash
curl -X POST "${KIBANA_URL}/api/agent_builder/tools/_execute" \
  -H "Authorization: ApiKey <base64-api-key>" -H "kbn-xsrf: true" -H "Content-Type: application/json" \
  -d '{"tool_id": "platform.sig_events.event_search", "tool_params": {"status": ["promoted", "acknowledged"], "per_page": 50}}'
```

Report each event's `title`, `criticality`, `confidence`, and `root_cause`; group by `status`.

### "Any KIs about certificate rotation on the agentless-api stream?"

```bash
curl -X POST "${KIBANA_URL}/api/agent_builder/tools/_execute" \
  -H "Authorization: ApiKey <base64-api-key>" -H "kbn-xsrf: true" -H "Content-Type: application/json" \
  -d '{"tool_id": "platform.sig_events.ki_search", "tool_params": {"stream_names": ["logging-gcp-us-central1-logs-agentless-api-log-default"], "search_text": "certificate rotation"}}'
```

### "Mark event X as a false positive"

1. `event_search` (or use the `event_id` already in hand) to confirm the event and its current status.
2. Confirm with the user, then `event_status_update` with `status: "demoted"`.

### (rare) "We found a new failure pattern in this stream that isn't being detected yet"

1. `ki_search` with `kind: ["query"]` scoped to the stream, to confirm the extraction pipeline genuinely hasn't
   captured this already.
2. If not covered, confirm the exact ES|QL and title with the user, then `ki_query_create` with `stream_name`,
   `title`, `esql.query`, and a `severity_score`.

## Guidelines

- **This is a read/triage skill first.** The overwhelming majority of requests are "what KIs exist" (`ki_search`),
  "what's currently open" (`event_search`), or "move this event to the next status"
  (`event_status_update`) — the KI-extraction and Discovery pipelines create the underlying data automatically.
  Reach for `ki_query_create`, `ki_feature_create`, or `event_create` only when a conversation surfaces a genuine gap
  in that automation, and confirm the gap with `ki_search`/`event_search` first.
- **Confirm before mutating.** `*_create`, `event_status_update`, and `event_investigation_attach` persist state —
  confirm with the user first, same as any other write operation.
- **Don't conflate confidence scales.** Feature-KI `confidence` is **0–100**; significant-event `confidence` is
  **0–1**. Passing one where the other is expected silently produces a nonsense score.
- **`demoted` ≠ `resolved`.** `demoted` means triage judged the event not real (false positive); `resolved` means it
  was real and has been handled. Don't resolve something that should be demoted, or vice versa.
- **Prefer semantic search.** `ki_search`'s `search_text` does hybrid keyword+vector ranking — a descriptive phrase
  ("pods failing to pull images") beats a bare keyword.
