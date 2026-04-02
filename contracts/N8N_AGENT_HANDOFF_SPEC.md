# n8n-to-Agent Runtime Handoff Specification
This specification defines the production handoff contract between `n8n` and all Enforcement Agent Library runtimes.

It is reusable across all enforcement agents, including invoice enforcement, proposal follow-up, renewal enforcement, and future agents.

Architecture assumptions:

- `Codex` and `Claude Code` build agent runtimes
- `Hostinger VPS` runs the agents
- `n8n` is the orchestration layer
- the live dashboard receives canonical events via the universal enforcement event schema and dashboard ingestion contract

## Purpose

This contract exists to standardize how `n8n` invokes an agent runtime, how the runtime returns decisions, and how `n8n` routes outcomes into actions, escalations, human review, and dashboard event delivery.

The runtime API is decision-oriented, not workflow-oriented. `n8n` owns orchestration. The agent runtime owns evaluation and decision output.

---

## 1. Request Contract From n8n to an Agent Runtime

**Primary endpoint pattern**

- `POST /api/v1/decide`

**Optional agent-specific endpoint alias**

- `POST /api/v1/agents/{agent_id}/decide`

The runtime must support one canonical decision endpoint. Agent-specific routing may exist internally, but the request contract must remain uniform.

**Content type**

- `application/json`

**Character encoding**

- `UTF-8`

**Request purpose**

Each request asks the runtime to:

- validate normalized inputs
- evaluate the enforcement condition
- produce a decision
- return a synchronous decision response
- include one or more dashboard-ready canonical events

**Canonical request envelope**

```json
{
  "api_version": "1.0",
  "request_id": "req_123",
  "idempotency_key": "proposal_123_stage_2_2026-04-01T14",
  "sent_at": "2026-04-01T14:00:00Z",
  "orchestrator": {
    "name": "n8n",
    "workflow_id": "wf_123",
    "workflow_execution_id": "wfe_456",
    "node_id": "http_request_1",
    "environment": "production"
  },
  "agent": {
    "agent_id": "proposal-follow-up-enforcer",
    "agent_version": "v1.0.0",
    "policy_version": "2026-04"
  },
  "trigger": {
    "trigger_type": "proposal_silence_72h",
    "trigger_id": "trg_789",
    "trigger_time": "2026-04-01T13:59:30Z",
    "source_systems": ["crm", "proposal_platform", "email_log", "n8n"]
  },
  "entity": {
    "entity_type": "proposal",
    "entity_id": "proposal_123",
    "customer_id": "cust_456",
    "account_id": "acct_789"
  },
  "inputs": {
    "normalized_payload": {}
  },
  "options": {
    "dry_run": false,
    "require_dashboard_events": true,
    "response_detail": "full"
  }
}
```

---

## 2. Required Headers and Authentication

**Required headers**

- `Authorization: Bearer <runtime_token>`
- `Content-Type: application/json`
- `X-Request-Id: <request_id>`
- `X-Idempotency-Key: <idempotency_key>`
- `X-Orchestrator: n8n`
- `X-Orchestrator-Workflow-Id: <workflow_id>`
- `X-Signature: <hmac_sha256_signature>`
- `X-Timestamp: <ISO-8601 or unix timestamp>`

**Authentication model**

Use both:

- bearer token authentication
- HMAC request signing over raw request body plus timestamp

**Security rules**

- runtime must reject requests with missing or invalid bearer token
- runtime must reject invalid signatures
- runtime must reject stale timestamps beyond tolerance window, default `5 minutes`
- all traffic must use HTTPS
- secrets should be rotated periodically
- each environment should have separate credentials

---

## 3. Required and Optional Request Fields

## Required top-level fields

- `api_version`
- `request_id`
- `idempotency_key`
- `sent_at`
- `orchestrator`
- `agent`
- `trigger`
- `entity`
- `inputs`

## Required nested fields

**`orchestrator`**
- `name`
- `workflow_id`
- `workflow_execution_id`
- `environment`

**`agent`**
- `agent_id`

**`trigger`**
- `trigger_type`
- `trigger_time`
- `source_systems`

**`entity`**
- `entity_type`
- `entity_id`

**`inputs`**
- `normalized_payload`

## Optional fields

- `agent.agent_version`
- `agent.policy_version`
- `trigger.trigger_id`
- `entity.customer_id`
- `entity.account_id`
- `options`
- any agent-specific normalized fields inside `inputs.normalized_payload`

## Input contract rule

`n8n` must send normalized business data in `inputs.normalized_payload`. The exact payload content varies by agent, but the envelope must remain stable for all agents.

---

## 4. Idempotency Rules

**Primary idempotency key**

- `idempotency_key`

**Secondary identifiers**

- `request_id`
- `entity.entity_type`
- `entity.entity_id`
- `trigger.trigger_type`
- `trigger.trigger_time`

**Runtime requirements**

- the runtime must treat repeated requests with the same `idempotency_key` as the same decision request
- if the same request is replayed unchanged, the runtime should return the same logical response
- if a request reuses the same `idempotency_key` with materially different payload contents, the runtime must reject it as a conflict

**n8n requirements**

- retries must preserve `idempotency_key`
- retries should preserve `request_id` if the exact same delivery is being retried
- new business evaluations must generate new idempotency keys

**Recommended idempotency key pattern**

`{entity_id}:{trigger_type}:{decision_window}`

Example:

- `invoice_7001:invoice_overdue_7d:2026-04-01`
- `proposal_123:proposal_silence_72h:2026-04-01T14`
- `renewal_5501:renewal_30d_no_owner_outreach:2026-04-01`

---

## 5. Expected Synchronous Response Structure

Every runtime must return a synchronous JSON response.

**Canonical synchronous response**

```json
{
  "api_version": "1.0",
  "request_id": "req_123",
  "idempotency_key": "proposal_123_stage_2_2026-04-01T14",
  "agent_id": "proposal-follow-up-enforcer",
  "agent_version": "v1.0.0",
  "execution_id": "exec_123",
  "response_type": "success",
  "decision": {
    "decision_code": "SEND_FOLLOW_UP_2",
    "decision_label": "Queue second follow-up email",
    "decision_confidence": 0.88,
    "reason_codes": ["PROPOSAL_SENT", "NO_REPLY_72H", "NO_OUTREACH_24H"],
    "leakage_condition": "silent_proposal_decay"
  },
  "action": {
    "action_type": "send_follow_up_email",
    "action_status": "queued",
    "action_channel": "email",
    "action_target": "buyer@example.com"
  },
  "routing": {
    "route": "action",
    "priority": "normal",
    "human_review_required": false,
    "escalation_required": false
  },
  "dashboard_events": [
    {}
  ],
  "errors": [],
  "meta": {
    "latency_ms": 842,
    "terminal": false
  }
}
```

## Required response fields

- `api_version`
- `request_id`
- `idempotency_key`
- `agent_id`
- `execution_id`
- `response_type`
- `decision`
- `action`
- `routing`
- `dashboard_events`
- `errors`
- `meta`

---

## 6. Success, Suppression, Escalation, Pending-Human, and Failure Response Shapes

## A. Success

Use when the runtime recommends or confirms a normal autonomous action.

**`response_type`**
- `success`

**Routing expectation**
- `n8n` proceeds to action execution path

**Example shape**
```json
{
  "response_type": "success",
  "decision": {
    "decision_code": "SEND_REMINDER",
    "decision_label": "Send reminder",
    "decision_confidence": 0.94,
    "reason_codes": ["INVOICE_OVERDUE", "REMINDER_STAGE_1"],
    "leakage_condition": "uncollected_invoice_balance"
  },
  "action": {
    "action_type": "send_email",
    "action_status": "queued",
    "action_channel": "email",
    "action_target": "billing@customer.com"
  },
  "routing": {
    "route": "action",
    "priority": "normal",
    "human_review_required": false,
    "escalation_required": false
  }
}
```

## B. Suppression

Use when no customer-facing or owner-facing action should be taken now.

**`response_type`**
- `suppressed`

**Routing expectation**
- `n8n` records result, sends dashboard events, ends workflow or parks it

**Example shape**
```json
{
  "response_type": "suppressed",
  "decision": {
    "decision_code": "SUPPRESS_RECENT_REPLY",
    "decision_label": "Suppress follow-up due to recent reply",
    "decision_confidence": 0.98,
    "reason_codes": ["RECENT_REPLY_DETECTED"],
    "leakage_condition": "silent_proposal_decay"
  },
  "action": {
    "action_type": null,
    "action_status": "suppressed",
    "action_channel": null,
    "action_target": null
  },
  "routing": {
    "route": "suppress",
    "priority": "normal",
    "human_review_required": false,
    "escalation_required": false
  }
}
```

## C. Escalation

Use when autonomous handling is insufficient and escalation is required.

**`response_type`**
- `escalated`

**Routing expectation**
- `n8n` routes to owner alert, Slack, task creation, or manager escalation path

**Example shape**
```json
{
  "response_type": "escalated",
  "decision": {
    "decision_code": "ESCALATE_HIGH_VALUE_RISK",
    "decision_label": "Escalate high-value silent entity",
    "decision_confidence": 0.91,
    "reason_codes": ["HIGH_VALUE", "NO_OWNER_ACTION"],
    "leakage_condition": "unworked_renewal_risk"
  },
  "action": {
    "action_type": "create_escalation",
    "action_status": "escalated",
    "action_channel": "slack",
    "action_target": "sales-manager"
  },
  "routing": {
    "route": "escalation",
    "priority": "high",
    "human_review_required": true,
    "escalation_required": true
  }
}
```

## D. Pending Human

Use when the runtime requires approval or manual judgment before any further action.

**`response_type`**
- `pending_human`

**Routing expectation**
- `n8n` pauses workflow, creates approval item, and waits for human resolution

**Example shape**
```json
{
  "response_type": "pending_human",
  "decision": {
    "decision_code": "AWAIT_HUMAN_REVIEW",
    "decision_label": "Await human review before next action",
    "decision_confidence": 0.57,
    "reason_codes": ["LOW_CONFIDENCE", "VIP_ACCOUNT"],
    "leakage_condition": "silent_proposal_decay"
  },
  "action": {
    "action_type": "await_review",
    "action_status": "awaiting_human",
    "action_channel": "internal_review",
    "action_target": "revenue-ops"
  },
  "routing": {
    "route": "human_review",
    "priority": "high",
    "human_review_required": true,
    "escalation_required": false
  }
}
```

## E. Failure

Use when the runtime cannot safely complete evaluation.

**`response_type`**
- `failed`

**Routing expectation**
- `n8n` retries if retryable, otherwise dead-letters and raises ops event

**Example shape**
```json
{
  "response_type": "failed",
  "decision": {
    "decision_code": "RUNTIME_EVALUATION_FAILED",
    "decision_label": "Runtime evaluation failed",
    "decision_confidence": 0,
    "reason_codes": ["MISSING_REQUIRED_INPUT"],
    "leakage_condition": null
  },
  "action": {
    "action_type": null,
    "action_status": "failed",
    "action_channel": null,
    "action_target": null
  },
  "routing": {
    "route": "failure",
    "priority": "high",
    "human_review_required": false,
    "escalation_required": false
  },
  "errors": [
    {
      "error_code": "MISSING_REQUIRED_INPUT",
      "error_message": "contact_email is required",
      "retryable": false
    }
  ]
}
```

---

## 7. Timeout and Retry Expectations

## Runtime timeout expectations

- target response time: `< 3 seconds`
- normal hard timeout: `10 seconds`
- absolute hard timeout: `15 seconds`

If the runtime cannot decide within the hard timeout, it should fail fast rather than hold the orchestration thread open.

## n8n retry expectations

Retry only when failure is transient.

**Retryable conditions**
- HTTP `408`
- HTTP `429`
- HTTP `500`
- HTTP `502`
- HTTP `503`
- HTTP `504`
- network timeout
- connection failure
- runtime response with `response_type = failed` and `errors[].retryable = true`

**Non-retryable conditions**
- HTTP `400`
- HTTP `401`
- HTTP `403`
- HTTP `404`
- HTTP `409`
- HTTP `422`
- runtime response with `retryable = false`

**Recommended retry policy**
- 1st retry: `30 seconds`
- 2nd retry: `2 minutes`
- 3rd retry: `10 minutes`
- 4th retry: `1 hour`
- then dead-letter

---

## 8. How the Agent Should Return Dashboard-Ready Event Data

The runtime must return canonical dashboard-ready events in `dashboard_events`.

## Rules

- every runtime response must include at least one canonical event
- every event must conform to the universal enforcement event schema `v1`
- the runtime must not return dashboard-only shorthand objects
- event payloads must be complete enough for direct submission to the dashboard ingestion endpoint

## Minimum requirement

`dashboard_events` must contain:

- a primary decision event representing the current evaluation result
- additional events only when needed for explicit lifecycle representation

## Example

- `agent.evaluated`
- `agent.action_queued`
- `agent.escalated`
- `agent.failed`

If the runtime returns multiple events, they must all share the same `execution.execution_id`.

---

## 9. How n8n Should Interpret and Route Each Response Type

## `success`

**Interpretation**
- the agent has produced a valid action path

**n8n routing**
- execute the recommended action
- update business systems as needed
- send returned `dashboard_events` to dashboard ingestion
- optionally emit follow-up execution events after downstream send succeeds

## `suppressed`

**Interpretation**
- no action should happen now

**n8n routing**
- do not execute customer-facing action
- ingest dashboard events
- optionally schedule future reevaluation

## `escalated`

**Interpretation**
- escalation path required

**n8n routing**
- notify owner, team, or manager
- create CRM/Slack/task escalation artifact
- ingest dashboard events
- optionally start SLA timer workflow

## `pending_human`

**Interpretation**
- human checkpoint required before proceeding

**n8n routing**
- create approval/review task
- pause or wait
- ingest dashboard events
- resume workflow only after manual decision event or manual input

## `failed`

**Interpretation**
- runtime could not safely decide

**n8n routing**
- inspect `errors[].retryable`
- retry if transient
- dead-letter if not retryable or retries exhausted
- ingest dashboard failure event if present
- alert ops for sustained failures

---

## 10. Versioning Rules for Future Agent Runtime APIs

## API version field

- `api_version` is mandatory in both request and response

## Current version family

- `1.x`

## Compatibility policy

- `1.x` minor versions must remain backward-compatible
- new optional fields may be added in minor versions
- existing field meanings must not change within major version `1`
- breaking changes require `2.0`

## Producer and consumer expectations

- `n8n` must send the highest supported version it knows how to produce
- runtimes must reject unsupported major versions clearly
- runtimes may accept lower minor versions within the same major version
- both sides should ignore unknown optional fields when safe

## Version negotiation rule

For `v1`, negotiation is simple:

- `n8n` sends `api_version`
- runtime returns `api_version`
- if unsupported, runtime returns explicit version error

## Recommended failure for unsupported version

- HTTP `422` or `400`
- machine-readable error:
  - `UNSUPPORTED_API_VERSION`

---

## Canonical Status Mapping

To keep routing uniform, each runtime must use exactly one of these response types:

- `success`
- `suppressed`
- `escalated`
- `pending_human`
- `failed`

No agent may invent its own top-level response category.

---

## Response Error Object Contract

When `errors` are present, each error object should use:

```json
{
  "error_code": "STRING_CODE",
  "error_message": "Human-readable explanation",
  "retryable": false,
  "field": "optional.field.path"
}
```

---

## Operational Rules

- `n8n` is responsible for orchestration, retries, pauses, routing, and downstream side effects.
- agent runtimes are responsible for decisioning, consistency, and canonical event generation.
- dashboard event generation must happen at decision time in the runtime, not be reconstructed later by `n8n`.
- all events returned by the runtime must be safe to forward directly to the dashboard ingestion endpoint.
- agent runtimes must remain deterministic first; orchestration logic must not be embedded into agent response types.

## Stability Rule

No enforcement agent may define a custom `n8n` handoff shape outside this contract. Agent-specific business inputs belong inside `inputs.normalized_payload`; agent-specific analytics belong inside canonical event `metadata`. The envelope, response categories, and routing semantics must remain system-wide and stable.

If you want, I can next produce a matching:
1. `runtime-request-v1.json` schema,
2. `runtime-response-v1.json` schema, or
3. a one-page operator version of this spec.
