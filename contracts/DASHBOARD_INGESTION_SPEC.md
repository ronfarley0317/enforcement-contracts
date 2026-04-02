# Dashboard Ingestion Specification
This specification defines the production ingestion contract for the Enforcement Agent Library dashboard. It applies to all events emitted by enforcement agents and `n8n` workflows. All inbound payloads must conform to the `universal enforcement event schema v1`.

## Scope

This spec governs:

- ingestion of events from multiple agents
- validation against the canonical event schema
- secure delivery from `Hostinger VPS` runtimes and `n8n`
- durable storage for observability and revenue proof
- rendering support for the live dashboard
- forward compatibility for future schema versions

Architecture assumptions:

- agents run on `Hostinger VPS`
- `n8n` is the orchestration layer
- the live dashboard is the observability and proof layer
- supported agents include `invoice enforcement`, `proposal follow-up`, and `renewal enforcement`

---

## 1. Ingestion Endpoint Design

**Primary endpoint**

- `POST /api/events/enforcement`

**Purpose**

- Accept one canonical enforcement event per request.
- Serve as the only write path for dashboard-visible agent events in `v1`.

**Optional bulk endpoint**

- `POST /api/events/enforcement/bulk`

**Purpose**

- Accept batched delivery from `n8n` or replay jobs.
- Intended for backfill, retry drains, or buffered flushes.
- Each element in the batch must independently satisfy schema validation.

**Request format**

- Content-Type: `application/json`
- Character encoding: `UTF-8`
- Payload:
  - single endpoint: one event object
  - bulk endpoint: object with `events` array

**Response design**

- `202 Accepted`
  - event accepted for validation and persistence
- `200 OK`
  - acceptable only if validation and persistence are synchronous and complete
- `400 Bad Request`
  - malformed JSON or structurally invalid payload
- `401 Unauthorized`
  - missing or invalid authentication
- `403 Forbidden`
  - authenticated sender not allowed to submit
- `409 Conflict`
  - duplicate event already processed with identical canonical identity
- `422 Unprocessable Entity`
  - valid JSON but schema or semantic validation failed
- `429 Too Many Requests`
  - sender exceeded rate limits
- `500 Internal Server Error`
  - dashboard ingestion fault
- `503 Service Unavailable`
  - temporary ingestion impairment; sender should retry

**Endpoint behavior**

- Single-event ingestion is the default contract.
- Bulk ingestion must return per-event acceptance status.
- Dashboard ingestion must never require agent-specific endpoint variations.
- The ingestion layer must remain schema-centric, not workflow-centric.

---

## 2. Authentication and Webhook Security

**Allowed producers**

- agent runtimes on `Hostinger VPS`
- `n8n` workflows acting as orchestrators or relays
- controlled replay/backfill jobs operated by platform owners

**Authentication model**

Use both:

- `Authorization: Bearer <ingestion_token>`
- `X-Enforcement-Signature` HMAC signature over raw request body

**Required headers**

- `Authorization`
- `X-Enforcement-Signature`
- `X-Enforcement-Timestamp`
- `X-Enforcement-Source`
- `Content-Type: application/json`

**Webhook signing**

- Signature algorithm: `HMAC-SHA256`
- Signed content: raw body + timestamp
- Signature secret: unique per producer or producer class
- Timestamp tolerance: default `5 minutes`

**Security requirements**

- Reject unsigned requests.
- Reject requests with expired timestamps.
- Reject requests whose signature does not match the raw body.
- Require TLS for all ingestion traffic.
- Rotate secrets periodically and on compromise.
- Maintain producer allowlist by source type or issuer.
- Log authentication failures without logging secrets or full sensitive payloads.

**Recommended producer identity model**

Each producer should have:

- `producer_id`
- `producer_type`
  - `agent`
  - `n8n`
  - `replay_job`
- secret / token pair
- allowed source scope
  - one agent
  - one environment
  - all internal producers if explicitly approved

---

## 3. Schema Validation Flow

Validation must occur in layers.

**Stage 1: transport validation**

- verify HTTPS
- verify headers
- verify content type
- verify body size limit
- verify authentication and signature

**Stage 2: JSON parsing**

- parse raw request body
- reject malformed JSON with `400`

**Stage 3: schema validation**

- validate payload against `universal enforcement event schema v1`
- reject schema-invalid events with `422`

**Stage 4: semantic validation**

Validate business consistency beyond JSON Schema:

- `event_id` must be non-empty and unique
- `event_time` must be parseable timestamp
- `schema_version` must be supported
- `decision.reason_codes` must be non-empty
- `action.action_status`, `status.outcome`, and `event_type` must not conflict
- revenue fields must be non-negative
- `currency` must be valid ISO-4217 style uppercase code
- if `escalation.escalation_flag = true`, required escalation context must be present
- if `human_review_required = true`, review status cannot be `not_required`

**Stage 5: normalization**

Normalize safe rendering fields for downstream use:

- canonical ingestion time
- canonical producer identity
- derived event date
- normalized agent grouping
- normalized entity grouping

**Stage 6: persistence**

- store raw payload
- store validated canonical record
- store ingestion metadata
- enqueue derivation/update jobs if needed

**Validation outcome**

- accept only events that pass all required stages
- rejected events must produce machine-readable error responses

---

## 4. Idempotency and Duplicate-Event Handling

**Primary idempotency key**

- `event_id`

**Secondary diagnostic uniqueness dimensions**

- `agent.agent_id`
- `execution.execution_id`
- `entity.entity_type`
- `entity.entity_id`
- `event_type`
- `event_time`

**Duplicate handling rules**

- If an event with the same `event_id` and same canonical payload already exists:
  - return `200` or `409` according to implementation policy
  - do not create a second stored event
- If the same `event_id` arrives with a materially different payload:
  - reject as conflict
  - flag for operator review
- Bulk ingestion must evaluate duplicates per event, not per batch.

**Idempotency expectations**

- Producers must treat `event_id` as immutable.
- Retries must reuse the same `event_id`.
- Replay jobs must not mutate historical payloads under the same `event_id`.

**Storage requirement**

- `event_id` must be uniquely indexed.

---

## 5. Error Handling and Retry Expectations

**Producer expectations**

Retry only for transient failures:

- `429`
- `500`
- `503`
- network timeout
- connection reset

Do not retry unchanged payloads on:

- `400`
- `401`
- `403`
- `422`

Retry behavior:

- exponential backoff
- jitter
- bounded retry attempts
- dead-letter or replay queue after exhaustion

**Recommended retry policy**

- attempt 1: immediate
- attempt 2: +30 seconds
- attempt 3: +2 minutes
- attempt 4: +10 minutes
- attempt 5: +1 hour
- then dead-letter / operator review

**Error response format**

Every non-success response should include:

- `error_code`
- `error_message`
- `retryable`
- `received_event_id` if parseable
- `schema_version` if parseable
- `details` for field-level errors when safe

**Examples of error classes**

- `AUTH_INVALID`
- `SIGNATURE_INVALID`
- `TIMESTAMP_EXPIRED`
- `JSON_INVALID`
- `SCHEMA_VALIDATION_FAILED`
- `SEMANTIC_VALIDATION_FAILED`
- `DUPLICATE_EVENT`
- `CONFLICTING_EVENT_ID`
- `RATE_LIMITED`
- `INGESTION_UNAVAILABLE`

---

## 6. Event Storage Requirements

The dashboard must persist both canonical analytics fields and the original event.

**Required storage layers**

- raw event store
- canonical event table / collection
- derived metrics store
- failed/rejected event log

**Raw event store must keep**

- raw JSON payload
- ingestion timestamp
- producer identity
- authentication result
- validation result
- request headers subset safe for audit
- processing outcome

**Canonical event store must keep**

- all schema fields
- normalized time dimensions
- indexing keys
- rendering-ready projections

**Minimum indexing requirements**

- `event_id` unique
- `event_time`
- `agent.agent_id`
- `entity.entity_type`
- `entity.entity_id`
- `status.outcome`
- `action.action_status`
- `event_type`
- `human_review.human_review_status`
- `escalation.escalation_flag`

**Retention requirements**

- raw events: minimum `12 months`
- canonical events: minimum `24 months`
- derived metrics: retain for dashboard history and trend analysis
- rejected events: minimum `90 days`

**Auditability requirements**

- preserve immutable accepted event records
- never silently overwrite accepted events
- corrections must happen through new events or explicit admin repair workflow

---

## 7. Required Fields for Dashboard Rendering

The dashboard must be able to render core observability and proof views using canonical fields only.

**Minimum fields required for rendering**

- `schema_version`
- `event_id`
- `event_time`
- `event_type`
- `agent.agent_id`
- `agent.agent_name`
- `agent.agent_version`
- `execution.execution_id`
- `execution.trigger_type`
- `entity.entity_type`
- `entity.entity_id`
- `decision.decision_code`
- `decision.decision_label`
- `decision.reason_codes`
- `action.action_status`
- `action.action_type` when applicable
- `action.action_channel` when applicable
- `revenue.currency`
- `revenue.revenue_at_risk`
- `revenue.revenue_protected`
- `revenue.revenue_recovered`
- `revenue.expected_value`
- `escalation.escalation_flag`
- `human_review.human_review_required`
- `human_review.human_review_status`
- `status.outcome`
- `status.terminal`

**Recommended but not strictly required for richer rendering**

- `workflow_id`
- `customer_id`
- `account_id`
- `decision.decision_confidence`
- `status.error_code`
- `status.error_message`
- `metadata`

---

## 8. Required Metrics Derivation Rules

All system-wide dashboard metrics must derive from canonical fields, not agent-specific metadata.

**Core counting rules**

- `total_events`
  - count of accepted events
- `total_executions`
  - count of distinct `execution.execution_id`
- `events_by_agent`
  - grouped by `agent.agent_id`
- `entities_touched`
  - distinct `entity.entity_type + entity.entity_id`

**Action metrics**

- `actions_proposed`
  - count where `action.action_status = proposed`
- `actions_queued`
  - count where `action.action_status = queued`
- `actions_sent`
  - count where `action.action_status = sent`
- `actions_executed`
  - count where `action.action_status = executed`
- `actions_suppressed`
  - count where `action.action_status = suppressed`
- `action_failures`
  - count where `action.action_status = failed`

**Outcome metrics**

- `successful_events`
  - count where `status.outcome = success`
- `failed_events`
  - count where `status.outcome = failed`
- `resolved_events`
  - count where `status.outcome = resolved`
- `pending_human_events`
  - count where `status.outcome = pending_human`

**Escalation metrics**

- `escalations_created`
  - count where `escalation.escalation_flag = true`
- `human_reviews_required`
  - count where `human_review.human_review_required = true`
- `human_reviews_pending`
  - count where `human_review.human_review_status = pending`

**Revenue metrics**

- `revenue_at_risk_total`
  - sum of `revenue.revenue_at_risk`
- `revenue_protected_total`
  - sum of `revenue.revenue_protected`
- `revenue_recovered_total`
  - sum of `revenue.revenue_recovered`
- `expected_value_total`
  - sum of `revenue.expected_value`

**Latency metrics**

- `avg_latency_ms`
  - average of `execution.latency_ms` where present
- `p95_latency_ms`
  - percentile over `execution.latency_ms`

**Failure-rate rule**

- `failure_rate = failed_events / total_events`

**Suppression-rate rule**

- `suppression_rate = actions_suppressed / total_executions`
  - or by event count if implementation standardizes this consistently

**Time bucketing**

All trend metrics must support grouping by:

- hour
- day
- week
- month

Primary bucketing should use `event_time`, not ingestion time.

---

## 9. Handling of Late, Failed, or Out-of-Order Events

**Late events**

Definition:

- event arrives materially after `event_time`

Rules:

- accept late events if schema-valid and within retention window
- store both `event_time` and ingestion time
- trend charts should use `event_time`
- operational freshness views may use ingestion time
- very late events should be flagged for monitoring

Recommended lateness classes:

- `on_time`: within 5 minutes
- `late`: 5 minutes to 24 hours
- `very_late`: over 24 hours

**Failed events**

Events with `status.outcome = failed` or `action.action_status = failed` must still be ingested and rendered.

Rules:

- failures are first-class proof events
- dashboard must expose failure counts and reasons
- failures must not be dropped from metrics pipelines
- error codes should power operational breakdowns

**Out-of-order events**

Examples:

- `agent.action_sent` arrives before `agent.action_queued`
- `resolved` arrives before a prior escalation event due to retry timing

Rules:

- do not reject solely for ordering issues if the event is otherwise valid
- preserve immutable arrival record
- derive current entity state using event-time-aware reconciliation rules
- dashboard timeline views should sort by `event_time`
- audit views may show both event order and ingestion order

**Entity state reconstruction**

For per-entity “current status” views:

- reconcile using latest valid event by `event_time`
- break ties by ingestion timestamp
- if still tied, break by deterministic precedence policy
  - terminal states outrank non-terminal
  - failed may remain visible as latest operational state but not override a later resolved business state

---

## 10. Version Compatibility Rules for Future Schema Updates

**Current supported version family**

- `1.x.x`

**Compatibility policy**

- ingestion service must accept all supported `1.x.x` events
- minor version additions must be backward-compatible
- unknown optional fields must not break ingestion if allowed by future schema versions and parser strategy
- major version changes require explicit ingestion support rollout

**Version handling rules**

- `schema_version` is mandatory
- unsupported major version must be rejected with `422` or `400`, depending on validation layer
- supported older minor versions within major `1` must be accepted
- metrics derivation must rely only on stable canonical fields guaranteed across the major version

**Forward-compatibility rules**

- new fields in future minor versions must not change the meaning of existing fields
- agent-specific additions belong in `metadata` unless promoted formally
- dashboard rendering logic must degrade gracefully when new optional fields are absent

**Migration rules**

When introducing `2.0.0` in the future:

- run dual-version support period
- keep `v1` ingestion active until all producers migrate
- version-specific validators must be explicit
- storage should preserve original version of every event
- cross-version metric derivation must either:
  - map both versions into a common internal model, or
  - maintain version-aware derivation logic

---

## Operational Notes

**Producer responsibility**

- agents on `Hostinger VPS` and `n8n` workflows must emit only canonical events
- retries must preserve `event_id`
- failed attempts to ingest must be logged and retried according to policy

**Dashboard responsibility**

- validate strictly
- store durably
- derive metrics only from accepted canonical records
- treat failures and suppressions as proof, not noise

**System stability rule**

No agent-specific dashboard contract is allowed to bypass the universal schema. All enforcement observability must pass through the canonical ingestion contract so the dashboard remains stable across invoice enforcement, proposal follow-up, renewal enforcement, and future agents.

If you want, I can next turn this into:
1. a shorter one-page operator spec, or
2. a matching `dashboard_ingestion_contract_v1` JSON response/error spec.
