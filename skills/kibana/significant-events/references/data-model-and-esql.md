# Significant Events — data model and ES|QL fallback

Backing indices are plain (if `hidden: true`) Elasticsearch data streams — the same ones the Agent Builder tools
read/write under the hood. Query them directly with ES|QL (e.g. via a Kibana console-proxy call, or any ES|QL-capable
client) when Agent Builder tool/MCP access isn't available. Reference each by its exact name — hidden data streams
don't match `*` wildcards in `list_indices` / `_resolve`.

## `.significant_events-knowledge_indicators`

One unified, `hidden: true` data stream holding **both** feature and query KIs, discriminated by `type`. It is an
**append-only revision log**: every create/update/delete writes a new document rather than mutating one in place — a
`deleted: true` document is a tombstone.

| Field                     | Present on           | Meaning                                                                |
| --------------------------- | ----------------------- | -------------------------------------------------------------------------- |
| `@timestamp`               | all                     | Revision timestamp — the reduction key for "latest"                      |
| `id`                        | all                     | Logical KI id (stable across revisions)                                  |
| `type`                      | all                     | `"feature"` or `"query"` — discriminator                                 |
| `stream.name`               | all                     | Owning stream (dot in the field name — quote it in ES|QL: `` `stream.name` ``) |
| `title` / `description`     | all                     | Human-facing summary                                                     |
| `tags`, `evidence`          | all                     | Optional metadata                                                        |
| `deleted`                   | tombstones              | `true` marks this id+type+stream as deleted as of this revision          |
| `excluded`                  | all                     | User-suppressed but not deleted (e.g. false-positive KI kept for audit)  |
| `expires_at`                | all                     | Managed KIs expire at this timestamp; absent = durable, no expiry        |
| `run_id`                    | all                     | Extraction run that produced this revision                               |
| `feature.type` / `.subtype` / `.slug` / `.properties` / `.confidence` (0–100) / `.evidence_doc_ids` / `.filter` / `.meta` | `type == "feature"` | Feature payload |
| `query.esql` / `.query_type` / `.severity_score` / `.rule_backed` / `.rule_id` / `.features[]` | `type == "query"` | The SigEvent detection query definition; `rule_id` links to the `alerting_v2` rule that runs it |

**Reduction query** — latest non-deleted, non-excluded revision per `(stream.name, type, id)`:

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

Decode each row's `_source` column as JSON to get the full document (nested `feature.*`/`query.*` objects aren't
individually mapped as ES|QL columns). For a lighter summary without decoding JSON, `KEEP` explicit top-level fields
instead of `_source` — e.g. `` KEEP `stream.name`, type, id, title, description, `query.severity_score` ``.

To scope to KIs only or SigEvent definitions only, add `| WHERE type == "feature"` or `| WHERE type == "query"`
before the `INLINE STATS` stages.

Also filter out expired managed KIs if needed: `WHERE expires_at IS NULL OR expires_at >= NOW()`.

## `.significant_events-events`

`hidden: true` data stream holding triaged significant events. Also an append-only revision log — each status change
writes a new document, chained via `previous_event_id`; `discovery_slug` is the stable id for "the same underlying
event" across its whole history.

Mapped columns (usable directly in `WHERE`/`KEEP` without decoding `_source`):

| Field               | Meaning                                                                 |
| --------------------- | ---------------------------------------------------------------------------- |
| `@timestamp`         | Revision timestamp                                                          |
| `event_id`           | This revision's id (changes every update)                                  |
| `discovery_id`       | Discovery run + candidate id that produced this event                      |
| `discovery_slug`     | **Stable id for the logical event** — group by this to find "current state" |
| `previous_event_id`  | Links to the prior revision in the chain                                   |
| `stream_names`       | Streams this event is attached to                                          |
| `status`             | `promoted` \| `acknowledged` \| `demoted` \| `resolved`                     |
| `title`, `summary`   | Human-facing text                                                           |

Everything else the tools return (`criticality`, `confidence`, `root_cause`, `dependency_edges`, `infra_components`,
`evidences`) lives in `_source` but is **not** individually mapped — only reachable via `KEEP _source` + JSON decode,
not as a named ES|QL column.

**Reduction query** — latest revision per logical event (`discovery_slug`):

```esql
FROM .significant_events-events METADATA _id, _source
| INLINE STATS latest_ts = MAX(@timestamp) BY discovery_slug
| WHERE @timestamp == latest_ts
| INLINE STATS tiebreaker_id = MAX(_id) BY discovery_slug
| WHERE _id == tiebreaker_id
| KEEP _source
| LIMIT 500
```

For a lightweight status-only view, `KEEP discovery_slug, event_id, status, title, summary, stream_names` instead of
`_source` — these five plus `stream_names` are the only fields mapped as real columns.

Filter to open work only: add `| WHERE status IN ("promoted", "acknowledged")` before the final `KEEP`.

## `.rule-events` — SigEvent occurrence counts (alerting_v2)

Occurrence/firing counts for rule-backed SigEvents are a separate concern from both indices above — they come from
alerts, not KI or event documents. On `alerting_v2`-based deployments (the modern path — check whether rules run as
`alerting_v2:rule_executor` tasks), alerts land in `.rule-events`, keyed by `rule.id` (nested field) with a
`group_hash` used to dedupe repeated firings within a bucket:

```esql
FROM .rule-events
| WHERE rule.id IN ("<rule-id-1>", "<rule-id-2>")
| STATS count = COUNT_DISTINCT(group_hash) BY rule_id = rule.id, bucket = BUCKET(@timestamp, 1 hour)
| SORT bucket ASC
| LIMIT 1000
```

Get `rule.id` from a query KI's `query.rule_id` field (see above) to attribute counts back to a named SigEvent. On
`alerting_v1`-based deployments the equivalent lives in a different reader (older event-log-backed path) — check
which alerting engine is active before assuming `.rule-events` is populated.

## Space scoping

Every document above carries `kibana.space_ids` (added on write). Kibana's own readers filter with
`` `kibana.space_ids` == "<space>" OR `kibana.space_ids` IS NULL `` before the latest-revision reduction. Omit this
filter if you only operate in the default space or want cross-space visibility; add it back for correctness in a
multi-space deployment:

```esql
| WHERE `kibana.space_ids` == "default" OR `kibana.space_ids` IS NULL
```

Insert this `WHERE` immediately after the `FROM ... METADATA` line, before the `INLINE STATS` stages, in either
reduction query above.
