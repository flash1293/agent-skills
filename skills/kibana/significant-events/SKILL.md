---
name: kibana-significant-events
description: >
  Search, triage, and manage Significant Events and Knowledge Indicators (KIs) on
  Kibana Streams — via the native Agent Builder tools (`platform.sig_events.*`), the
  per-stream Streams REST API, or raw ES|QL against the underlying
  `.significant_events-*` and `.rule-events` indices when Agent Builder access isn't
  available. Use when the user asks about significant events or SigEvents, wants to
  see what's currently promoted/acknowledged/open, wants to search or create
  Knowledge Indicators (feature or query KIs) for a stream, needs to
  promote/acknowledge/demote/resolve an event, or wants a stream's SigEvent
  occurrence trend.
metadata:
  author: elastic
  version: 0.1.0
compatibility: Kibana 9.1+ (`streams` + `significant_events` plugins); the native tool
  path additionally requires Agent Builder 9.2+
---

# Kibana Significant Events

Significant Events ("SigEvents") is Streams' detection and triage layer: rule-backed ES|QL queries attached to a
stream fire, get correlated by a Discovery agent, and surface as triaged **significant events** a human or agent can
act on. Knowledge Indicators (KIs) are the supporting context — durable facts about a stream (**feature** KIs, e.g.
detected technologies/dependencies) and the detection queries themselves (**query** KIs, the SigEvent definitions).

This skill covers three ways to read and manage this data, in preference order: native **Agent Builder tools**
(richest, includes triage state), the per-stream **Streams REST API** (public but deprecated, definitions +
occurrence counts only), and raw **ES|QL** against the backing indices (works with no Agent Builder access, but you
own the "latest revision" reduction yourself).

## Concepts

- **Query KI** — a SigEvent definition: an ES|QL detection query plus `title`, `severity_score`, and (if rule-backed)
  the `alerting_v2` `rule_id` that runs it on a schedule.
- **Feature KI** — a durable fact about a stream discovered by the KI-extraction pipeline (detected technology,
  dependency, entity, infra component, schema). Used as context, not as a detector.
- **Significant event** — a triaged, human-facing incident synthesized by the Discovery pipeline from one or more
  firing queries. Has a lifecycle: `promoted` → `acknowledged` or `demoted` → `resolved`. `demoted` means triage
  decided it's not real; `resolved` means it was real and is now handled.
- Both KIs and significant events are **append-only revision logs**, not overwritten-in-place documents — every
  update or delete writes a new revision. "Current state" always means the latest non-deleted revision per logical
  entity.

## Method selector

| Task                                                            | Method                                                                  |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------ |
| Search KIs (feature and/or query) for one or more streams        | `platform.sig_events.ki_search`                                        |
| See what significant events are currently open/promoted          | `platform.sig_events.event_search`                                     |
| Save a newly-discovered detection query as a KI                  | `platform.sig_events.ki_query_create`                                  |
| Save a newly-discovered stream fact as a KI                       | `platform.sig_events.ki_feature_create`                                |
| Promote/acknowledge/demote/resolve, or create a new event         | `platform.sig_events.event_create` / `event_status_update`             |
| Record an investigation run against an event                     | `platform.streams.sig_events.event_investigation_attach`               |
| Read one known stream's SigEvent definitions + occurrence trend   | Streams REST API — `GET /api/streams/{name}/significant_events`        |
| No Agent Builder tool/MCP access at all                          | Raw ES|QL fallback (see below)                                          |

## Prerequisites

| Item               | Description                                                                              |
| ------------------- | ------------------------------------------------------------------------------------------ |
| **Kibana URL**      | Kibana endpoint (e.g. `https://localhost:5601` or a Cloud deployment URL)                 |
| **Authentication**  | API key or basic auth (see the `elasticsearch-authn` skill)                               |
| **Privileges**      | `read_stream` (read) / `manage_stream` (KI writes) for the target streams; read access to Agent Builder for the tool path |

## Method 1: Agent Builder tools (preferred)

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

### Tool reference

| Tool                                                       | Purpose                                                        | Required params                                                                | Notes                                                                 |
| ------------------------------------------------------------ | ----------------------------------------------------------------- | ---------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| `platform.sig_events.ki_search`                             | Search KIs (feature and/or query) with optional semantic ranking | none                                                                              | `stream_names?`, `search_text?` (semantic), `kind?: ["feature"\|"query"]`, `limit` (default 50) |
| `platform.sig_events.event_search`                          | Search triaged significant events                                | none                                                                              | `query?` (title text), `stream_name?`, `status?: [...]`, `page`, `per_page`. **Call this before creating or updating** to avoid duplicates. |
| `platform.sig_events.ki_query_create`                       | Persist a new query KI (detection definition)                    | `stream_name`, `title`, `esql.query`                                             | `severity_score?`, `description?`, `evidence?[]`, `expires_at?` (omit for a durable KI) |
| `platform.sig_events.ki_feature_create`                     | Persist a new feature KI (stream fact)                            | `id`, `stream_name`, `type`, `description`, `properties`, `confidence` (0–100)    | `subtype?`, `evidence?[]`, `evidence_doc_ids?[]`, `tags?[]`, `filter?`, `expires_at?` |
| `platform.sig_events.event_create`                          | Create a new significant event                                    | `title`, `summary`, `root_cause`, `stream_names[]`, `criticality` (0–100), `confidence` (**0–1**), `recommendations[]` | `status?` defaults per backend; distinct 0–1 confidence scale vs. feature KIs' 0–100 |
| `platform.sig_events.event_status_update`                  | Change an event's lifecycle status                                 | `event_id`, `status: promoted\|acknowledged\|demoted\|resolved`                  | —                                                                          |
| `platform.streams.sig_events.event_investigation_attach`   | Record an investigation run on an event                            | `event_id`, `workflow_execution_id`, `status: pending\|success\|failed`, `started_at` | Call once at start (`pending`) and again at completion (`success`/`failed`, add `completed_at`) |

Full JSON Schemas for every tool: [references/tool-schemas.md](references/tool-schemas.md).

## Method 2: Streams REST API (per-stream, deprecated)

If you already know the stream name and only need its SigEvent definitions plus their firing-rate time series (not
feature KIs, not triaged event instances), the public (though deprecated) Streams endpoint does the join for you:

```bash
curl -G "${KIBANA_URL}/api/streams/<stream-name>/significant_events" \
  -H "Authorization: ApiKey <base64-api-key>" \
  --data-urlencode "from=2026-07-07T00:00:00Z" \
  --data-urlencode "to=2026-07-08T00:00:00Z" \
  --data-urlencode "bucketSize=1h"
```

Returns `{ queries: [...with occurrences[]...], aggregated_occurrences: [...] }`. See the `kibana-streams` skill for
the rest of the Streams API surface (stream lifecycle, ingest/query settings, attachments).

## Method 3: Raw ES|QL fallback (no Agent Builder access)

KIs and significant events both live in **hidden** data streams — reference them by exact name, not a wildcard
pattern, and add `METADATA _id, _source` plus `KEEP _source` to get the full document (most of the interesting
fields, like `root_cause`, `criticality`, `dependency_edges`, and `feature.properties`, aren't in the explicit
mapping and won't come back as named columns).

Because both are append-only revision logs, reduce to the latest non-deleted revision per logical entity yourself
with a two-stage `INLINE STATS` (this is the exact pattern Kibana's own reader uses internally):

```esql
FROM .significant_events-knowledge_indicators METADATA _id, _source
| WHERE deleted IS NULL OR deleted == false
| WHERE excluded IS NULL OR excluded == false
| INLINE STATS latest_ts = MAX(@timestamp) BY `stream.name`, type, id
| WHERE @timestamp == latest_ts
| INLINE STATS tiebreaker_id = MAX(_id) BY `stream.name`, type, id
| WHERE _id == tiebreaker_id
| KEEP _source
| LIMIT 500
```

```esql
FROM .significant_events-events METADATA _id, _source
| INLINE STATS latest_ts = MAX(@timestamp) BY discovery_slug
| WHERE @timestamp == latest_ts
| INLINE STATS tiebreaker_id = MAX(_id) BY discovery_slug
| WHERE _id == tiebreaker_id
| KEEP _source
| LIMIT 500
```

Both queries were validated against a live cluster and returned counts matching `ki_search`/`event_search` exactly.
SigEvent **occurrence counts** (firing time series) are a separate concern — they come from alerts, not these
indices; for `alerting_v2`-based deployments they land in `.rule-events` keyed by `rule.id`. Full field reference,
the occurrence-count query, and the `deleted`/`excluded`/`expires_at` semantics:
[references/data-model-and-esql.md](references/data-model-and-esql.md).

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

### "We found a new failure pattern in this stream — save it as a detection"

1. `ki_search` with `kind: ["query"]` scoped to the stream to confirm it isn't already covered.
2. If not covered, confirm the exact ES|QL and title with the user, then `ki_query_create` with `stream_name`,
   `title`, `esql.query`, and a `severity_score`.

### "Mark event X as a false positive"

1. `event_search` (or use the `event_id` already in hand) to confirm the event and its current status.
2. Confirm with the user, then `event_status_update` with `status: "demoted"`.

### No Agent Builder access — just show me the KIs

Run the Method 3 reduction query for `.significant_events-knowledge_indicators`, add `| WHERE type == "feature"`
before the `INLINE STATS` stages, decode each row's `_source`, and present the same shape `ki_search` would return.

## Guidelines

- **Search before you create.** Always run `ki_search` / `event_search` first — creating a duplicate KI or event for
  an already-known condition is the most common mistake. `event_search`'s own description says to use it "before
  creating or updating events."
- **Confirm before mutating.** `*_create`, `event_status_update`, and `event_investigation_attach` persist state —
  confirm with the user first, same as any other write operation.
- **Don't conflate confidence scales.** Feature-KI `confidence` is **0–100**; significant-event `confidence` is
  **0–1**. Passing one where the other is expected silently produces a nonsense score.
- **`demoted` ≠ `resolved`.** `demoted` means triage judged the event not real (false positive); `resolved` means it
  was real and has been handled. Don't resolve something that should be demoted, or vice versa.
- **Prefer semantic search.** `ki_search`'s `search_text` does hybrid keyword+vector ranking — a descriptive phrase
  ("pods failing to pull images") beats a bare keyword.
- **Only fall back to raw ES|QL when you must.** It requires the latest-revision reduction above, doesn't give you
  the tools' semantic search, and won't stop you from writing a duplicate — prefer Method 1 whenever Agent Builder
  access is available.
- **Reference indices by exact name, not wildcard.** `.significant_events-knowledge_indicators`,
  `.significant_events-events`, and `.rule-events` are all `hidden: true` — a `*` pattern in `list_indices` or
  `_resolve` won't surface them.
