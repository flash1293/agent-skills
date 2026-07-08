# Significant Events — Agent Builder tool schemas

Full JSON Schema for every native `platform.sig_events.*` / `platform.streams.sig_events.*` tool, captured from a live
`GET /api/agent_builder/tools` / MCP `tools/list` response. Pass these as `tool_params` to `POST
{KIBANA_URL}/api/agent_builder/tools/_execute` with `tool_id` set to the tool name.

## `platform.sig_events.ki_search`

Search Knowledge Indicators (KIs) derived from streams data to enrich context for a target stream, service, or group
of streams. Includes feature-based indicators (stream features) and query-based indicators (stored stream queries).

```json
{
  "type": "object",
  "properties": {
    "stream_names": {
      "description": "Optional. If omitted, search across all accessible streams.",
      "type": "array",
      "items": { "type": "string" }
    },
    "search_text": {
      "description": "Optional. Natural-language search with semantic ranking (hybrid keyword + vector). Descriptive phrases work better than single keywords.",
      "type": "string"
    },
    "kind": {
      "default": [],
      "description": "What to return. ['query']: queries-only KIs. ['feature']: feature-based KIs only. default (empty/omitted): both.",
      "type": "array",
      "items": { "type": "string", "enum": ["feature", "query"] }
    },
    "limit": {
      "default": 50,
      "description": "Optional safety cap for returned items.",
      "type": "number",
      "minimum": 1
    }
  }
}
```

## `platform.sig_events.event_search`

Search significant events across all streams or a specific stream. Use before creating or updating events to
understand current event state.

```json
{
  "type": "object",
  "properties": {
    "query": { "description": "Optional text search in event title.", "type": "string" },
    "stream_name": { "description": "Optional stream name to scope the search.", "type": "string" },
    "status": {
      "description": "Optional event status filters.",
      "type": "array",
      "items": { "type": "string", "enum": ["promoted", "acknowledged", "demoted", "resolved"] }
    },
    "page": { "default": 1, "type": "integer", "minimum": 1 },
    "per_page": { "default": 20, "type": "integer", "minimum": 1, "maximum": 100 }
  }
}
```

## `platform.sig_events.ki_query_create`

Create a query Knowledge Indicator (a SigEvent detection definition) for a stream and persist it to significant
events query storage. Use when the conversation discovers a new detection query that should be saved for future
investigations.

```json
{
  "type": "object",
  "properties": {
    "stream_name": { "type": "string", "description": "Target stream name where this query KI should be saved." },
    "title": { "type": "string", "minLength": 1 },
    "esql": {
      "type": "object",
      "properties": { "query": { "type": "string" } },
      "required": ["query"]
    },
    "severity_score": { "type": "number" },
    "evidence": { "type": "array", "items": { "type": "string" } },
    "description": { "default": "", "type": "string" },
    "expires_at": {
      "description": "Optional expiry deadline (ISO 8601). Provide to create a managed KI that expires at this date. Omit to create a durable KI with no expiry.",
      "type": "string",
      "format": "date-time"
    },
    "id": { "type": "string" }
  },
  "required": ["stream_name", "title", "esql"]
}
```

## `platform.sig_events.ki_feature_create`

Create a feature Knowledge Indicator for a stream and persist it to significant events feature storage. Use when the
conversation discovers a new stream behavior pattern that should be saved as a feature KI for future investigations.

```json
{
  "type": "object",
  "properties": {
    "id": { "type": "string" },
    "stream_name": { "type": "string" },
    "type": { "type": "string" },
    "subtype": { "type": "string" },
    "title": { "type": "string" },
    "description": { "type": "string" },
    "properties": { "type": "object", "additionalProperties": {} },
    "confidence": { "type": "number", "minimum": 0, "maximum": 100 },
    "evidence": { "type": "array", "items": { "type": "string" } },
    "evidence_doc_ids": { "type": "array", "items": { "type": "string" } },
    "tags": { "type": "array", "items": { "type": "string" } },
    "filter": { "description": "Streamlang Condition object (field/eq/and/or/not/exists/range/...), same shape used by stream routing conditions." },
    "meta": { "type": "object", "additionalProperties": {} },
    "expires_at": {
      "description": "Optional expiry deadline (ISO 8601). Omit to create a durable KI with no expiry.",
      "type": "string",
      "format": "date-time"
    }
  },
  "required": ["id", "stream_name", "type", "description", "properties", "confidence"]
}
```

## `platform.sig_events.event_create`

Create a significant event for one or more streams.

```json
{
  "type": "object",
  "properties": {
    "status": {
      "description": "Status for the new event.",
      "type": "string",
      "enum": ["promoted", "acknowledged", "demoted", "resolved"]
    },
    "title": { "type": "string", "maxLength": 500 },
    "summary": { "type": "string", "maxLength": 4000 },
    "root_cause": { "type": "string", "maxLength": 4000 },
    "stream_names": { "minItems": 1, "maxItems": 100, "type": "array", "items": { "type": "string", "maxLength": 255 } },
    "criticality": { "type": "number", "minimum": 0, "maximum": 100 },
    "confidence": { "type": "number", "minimum": 0, "maximum": 1 },
    "recommendations": { "minItems": 1, "maxItems": 50, "type": "array", "items": { "type": "string", "maxLength": 1000 } }
  },
  "required": ["title", "summary", "root_cause", "stream_names", "criticality", "confidence", "recommendations"]
}
```

Note `confidence` here is **0–1** — a different scale from `ki_feature_create`'s 0–100 `confidence`.

## `platform.sig_events.event_status_update`

Update the status of an existing significant event.

```json
{
  "type": "object",
  "properties": {
    "event_id": { "type": "string", "description": "Identifier of the significant event to update." },
    "status": {
      "type": "string",
      "enum": ["promoted", "acknowledged", "demoted", "resolved"],
      "description": "Target status value to set."
    }
  },
  "required": ["event_id", "status"]
}
```

## `platform.streams.sig_events.event_investigation_attach`

Record an investigation run on a significant event. Call this when an investigation starts (`status: pending`) and
again when it finishes (`status: success` or `failed`) to keep the event up to date.

```json
{
  "type": "object",
  "properties": {
    "event_id": { "type": "string", "description": "Identifier of the significant event to attach the investigation to." },
    "workflow_execution_id": { "type": "string", "description": "The investigation workflow execution id returned by execute_workflow. Used to fetch detailed RCA data." },
    "status": {
      "type": "string",
      "enum": ["pending", "success", "failed"],
      "description": "Status of the investigation: \"pending\" while running, \"success\" or \"failed\" when done."
    },
    "started_at": { "type": "string", "format": "date-time", "description": "ISO-8601 datetime when the investigation started." },
    "completed_at": { "type": "string", "format": "date-time", "description": "ISO-8601 datetime when the investigation completed. Only set for terminal statuses." }
  },
  "required": ["event_id", "workflow_execution_id", "status", "started_at"]
}
```

Note this tool's id sits under the `platform.streams.sig_events.*` namespace (three segments before the leaf name),
not `platform.sig_events.*` like the other six — if filtering the MCP endpoint's `tools/list` by `?namespace=`, filter
on `platform.streams.sig_events` separately from `platform.sig_events` to include it.
