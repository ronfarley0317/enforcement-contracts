# Enforcement Agent Library Universal Event Schema

This is the system-wide event contract for all enforcement agents, `n8n` workflows, and the live dashboard. It is designed to stay stable across future agents while allowing controlled extension.

## 1. Canonical JSON Schema Definition

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://enforcement-agent-library/schemas/event/v1.0.0.json",
  "title": "Enforcement Agent Event",
  "type": "object",
  "additionalProperties": false,
  "required": [
    "schema_version",
    "event_id",
    "event_time",
    "event_type",
    "agent",
    "execution",
    "entity",
    "decision",
    "action",
    "revenue",
    "escalation",
    "human_review",
    "status"
  ],
  "properties": {
    "schema_version": {
      "type": "string",
      "pattern": "^1\\.\\d+\\.\\d+$"
    },
    "event_id": {
      "type": "string",
      "minLength": 1
    },
    "event_time": {
      "type": "string",
      "format": "date-time"
    },
    "event_type": {
      "type": "string",
      "enum": [
        "agent.evaluated",
        "agent.action_queued",
        "agent.action_sent",
        "agent.action_executed",
        "agent.action_suppressed",
        "agent.escalated",
        "agent.human_review_requested",
        "agent.human_review_resolved",
        "agent.resolved",
        "agent.failed"
      ]
    },
    "agent": {
      "type": "object",
      "additionalProperties": false,
      "required": [
        "agent_id",
        "agent_name",
        "agent_version"
      ],
      "properties": {
        "agent_id": { "type": "string", "minLength": 1 },
        "agent_name": { "type": "string", "minLength": 1 },
        "agent_version": { "type": "string", "minLength": 1 },
        "library_version": { "type": "string" },
        "owner": { "type": "string" }
      }
    },
    "execution": {
      "type": "object",
      "additionalProperties": false,
      "required": [
        "execution_id",
        "trigger_type",
        "source_systems"
      ],
      "properties": {
        "execution_id": { "type": "string", "minLength": 1 },
        "workflow_id": { "type": ["string", "null"] },
        "trigger_type": { "type": "string", "minLength": 1 },
        "trigger_id": { "type": ["string", "null"] },
        "source_systems": {
          "type": "array",
          "items": { "type": "string", "minLength": 1 }
        },
        "idempotency_key": { "type": ["string", "null"] },
        "latency_ms": {
          "type": ["integer", "null"],
          "minimum": 0
        }
      }
    },
    "entity": {
      "type": "object",
      "additionalProperties": false,
      "required": [
        "entity_type",
        "entity_id"
      ],
      "properties": {
        "entity_type": {
          "type": "string",
          "enum": [
            "account",
            "customer",
            "invoice",
            "opportunity",
            "proposal",
            "estimate",
            "subscription",
            "contract",
            "renewal",
            "ticket",
            "payment",
            "job",
            "other"
          ]
        },
        "entity_id": { "type": "string", "minLength": 1 },
        "parent_entity_type": { "type": ["string", "null"] },
        "parent_entity_id": { "type": ["string", "null"] },
        "customer_id": { "type": ["string", "null"] },
        "account_id": { "type": ["string", "null"] },
        "external_refs": {
          "type": "array",
          "items": {
            "type": "object",
            "additionalProperties": false,
            "required": ["system", "id"],
            "properties": {
              "system": { "type": "string" },
              "id": { "type": "string" }
            }
          }
        }
      }
    },
    "decision": {
      "type": "object",
      "additionalProperties": false,
      "required": [
        "decision_code",
        "decision_label",
        "decision_confidence",
        "reason_codes"
      ],
      "properties": {
        "decision_code": { "type": "string", "minLength": 1 },
        "decision_label": { "type": "string", "minLength": 1 },
        "decision_confidence": {
          "type": "number",
          "minimum": 0,
          "maximum": 1
        },
        "reason_codes": {
          "type": "array",
          "items": { "type": "string", "minLength": 1 }
        },
        "policy_version": { "type": ["string", "null"] },
        "leakage_condition": { "type": ["string", "null"] }
      }
    },
    "action": {
      "type": "object",
      "additionalProperties": false,
      "required": [
        "action_status"
      ],
      "properties": {
        "action_type": { "type": ["string", "null"] },
        "action_status": {
          "type": "string",
          "enum": [
            "none",
            "proposed",
            "queued",
            "sent",
            "executed",
            "suppressed",
            "escalated",
            "awaiting_human",
            "failed",
            "resolved"
          ]
        },
        "action_channel": { "type": ["string", "null"] },
        "action_target": { "type": ["string", "null"] },
        "scheduled_for": {
          "type": ["string", "null"],
          "format": "date-time"
        },
        "completed_at": {
          "type": ["string", "null"],
          "format": "date-time"
        }
      }
    },
    "revenue": {
      "type": "object",
      "additionalProperties": false,
      "required": [
        "currency",
        "revenue_at_risk",
        "revenue_protected",
        "revenue_recovered",
        "expected_value"
      ],
      "properties": {
        "currency": {
          "type": "string",
          "pattern": "^[A-Z]{3}$"
        },
        "revenue_at_risk": { "type": "number", "minimum": 0 },
        "revenue_protected": { "type": "number", "minimum": 0 },
        "revenue_recovered": { "type": "number", "minimum": 0 },
        "expected_value": { "type": "number", "minimum": 0 },
        "attribution_window_days": {
          "type": ["integer", "null"],
          "minimum": 0
        },
        "revenue_formula_ref": { "type": ["string", "null"] }
      }
    },
    "escalation": {
      "type": "object",
      "additionalProperties": false,
      "required": [
        "escalation_flag"
      ],
      "properties": {
        "escalation_flag": { "type": "boolean" },
        "escalation_level": { "type": ["string", "null"] },
        "escalation_target": { "type": ["string", "null"] },
        "escalation_reason": { "type": ["string", "null"] },
        "escalation_due_at": {
          "type": ["string", "null"],
          "format": "date-time"
        }
      }
    },
    "human_review": {
      "type": "object",
      "additionalProperties": false,
      "required": [
        "human_review_required",
        "human_review_status"
      ],
      "properties": {
        "human_review_required": { "type": "boolean" },
        "human_review_status": {
          "type": "string",
          "enum": [
            "not_required",
            "pending",
            "approved",
            "rejected",
            "overridden",
            "timed_out"
          ]
        },
        "reviewer_id": { "type": ["string", "null"] },
        "review_notes": { "type": ["string", "null"] },
        "reviewed_at": {
          "type": ["string", "null"],
          "format": "date-time"
        }
      }
    },
    "status": {
      "type": "object",
      "additionalProperties": false,
      "required": [
        "outcome"
      ],
      "properties": {
        "outcome": {
          "type": "string",
          "enum": [
            "success",
            "suppressed",
            "escalated",
            "pending_human",
            "failed",
            "resolved"
          ]
        },
        "terminal": { "type": "boolean" },
        "error_code": { "type": ["string", "null"] },
        "error_message": { "type": ["string", "null"] }
      }
    },
    "metadata": {
      "type": "object"
    }
  }
}
```

---

## 2. Field-by-Field Explanation

**Top level**
- `schema_version`: Contract version for consumers and producers.
- `event_id`: Unique identifier for this event instance.
- `event_time`: When the event was emitted.
- `event_type`: Lifecycle event category.

**`agent`**
- `agent_id`: Stable system slug.
- `agent_name`: Human-readable name.
- `agent_version`: Runtime/build version.
- `library_version`: Optional Enforcement Agent Library version.
- `owner`: Optional owning team or function.

**`execution`**
- `execution_id`: Unique run ID for one agent evaluation.
- `workflow_id`: Upstream orchestration ID, usually from `n8n`.
- `trigger_type`: Why the run happened.
- `trigger_id`: Source event or schedule ID if available.
- `source_systems`: Systems used in the evaluation.
- `idempotency_key`: Deduplication key.
- `latency_ms`: End-to-end decision latency.

**`entity`**
- `entity_type`: Business object being enforced.
- `entity_id`: Primary object ID.
- `parent_entity_type` / `parent_entity_id`: Optional parent relationship.
- `customer_id`: Customer reference if distinct.
- `account_id`: Account reference if distinct.
- `external_refs`: Cross-system IDs.

**`decision`**
- `decision_code`: Stable machine code for logic branch.
- `decision_label`: Human-readable decision summary.
- `decision_confidence`: Confidence from `0` to `1`.
- `reason_codes`: Machine-readable reasons supporting the decision.
- `policy_version`: Optional decision policy/ruleset version.
- `leakage_condition`: Leakage condition being enforced.

**`action`**
- `action_type`: Intended or executed action.
- `action_status`: State of the action.
- `action_channel`: Email, SMS, task, Slack, API, etc.
- `action_target`: Recipient, queue, team, endpoint, or target ID.
- `scheduled_for`: When the action is planned.
- `completed_at`: When the action completed.

**`revenue`**
- `currency`: ISO 4217 currency code.
- `revenue_at_risk`: Total exposed value.
- `revenue_protected`: Modeled protected value.
- `revenue_recovered`: Realized recovered value.
- `expected_value`: Expected economic value of this intervention.
- `attribution_window_days`: Attribution window.
- `revenue_formula_ref`: Reference to formula name or doc.

**`escalation`**
- `escalation_flag`: Whether escalation occurred or is required.
- `escalation_level`: Severity/tier.
- `escalation_target`: Human or team target.
- `escalation_reason`: Why escalation happened.
- `escalation_due_at`: SLA deadline.

**`human_review`**
- `human_review_required`: Whether human review is needed.
- `human_review_status`: Current review state.
- `reviewer_id`: Approver/reviewer identifier.
- `review_notes`: Optional notes.
- `reviewed_at`: Timestamp of review decision.

**`status`**
- `outcome`: Final current outcome of the event.
- `terminal`: Whether this event represents a terminal state.
- `error_code`: Structured failure code.
- `error_message`: Human-readable failure detail.

**`metadata`**
- Free-form extension area for agent-specific fields.

---

## 3. Required vs Optional Fields

**Required top-level fields**
- `schema_version`
- `event_id`
- `event_time`
- `event_type`
- `agent`
- `execution`
- `entity`
- `decision`
- `action`
- `revenue`
- `escalation`
- `human_review`
- `status`

**Required nested fields**
- `agent.agent_id`
- `agent.agent_name`
- `agent.agent_version`
- `execution.execution_id`
- `execution.trigger_type`
- `execution.source_systems`
- `entity.entity_type`
- `entity.entity_id`
- `decision.decision_code`
- `decision.decision_label`
- `decision.decision_confidence`
- `decision.reason_codes`
- `action.action_status`
- `revenue.currency`
- `revenue.revenue_at_risk`
- `revenue.revenue_protected`
- `revenue.revenue_recovered`
- `revenue.expected_value`
- `escalation.escalation_flag`
- `human_review.human_review_required`
- `human_review.human_review_status`
- `status.outcome`

**Optional fields**
- Everything else, unless a validation rule below makes it conditionally required.

---

## 4. Validation Rules

**Core**
- `event_id` must be globally unique.
- `event_time` must be valid ISO-8601 UTC or offset datetime.
- `schema_version` must follow `major.minor.patch`.
- `decision_confidence` must be between `0` and `1`.
- Revenue fields must be numeric and `>= 0`.
- `currency` must be a 3-letter uppercase code like `USD`.

**Conditional**
- If `escalation.escalation_flag = true`, then these should be present:
  - `escalation.escalation_level`
  - `escalation.escalation_reason`
- If `human_review.human_review_required = true`, then `human_review.human_review_status` cannot be `not_required`.
- If `status.outcome = failed`, then `status.error_code` should be present.
- If `action.action_status = failed`, then `status.outcome` should usually be `failed` or `escalated`.
- If `action.action_status = sent|executed|resolved`, then `action.action_type` should be present.
- If `event_type = agent.human_review_resolved`, then `human_review.reviewed_at` should be present.
- If `status.terminal = true`, event should represent a terminal business state such as `resolved`, `failed`, or permanent `suppressed`.

**Consistency**
- `event_type`, `action.action_status`, and `status.outcome` must not contradict each other.
- `revenue_recovered` should not exceed `revenue_at_risk` unless explicitly allowed by a future schema version.
- `reason_codes` should never be empty in production events.
- `source_systems` should list every system materially involved in the decision.

---

## 5. Versioning Strategy

Use semantic versioning on `schema_version`.

**Major**
- Breaking changes only.
- Examples: renaming fields, changing meanings, removing fields, tightening enums incompatibly.

**Minor**
- Backward-compatible additions.
- Examples: adding optional fields, adding non-breaking enum values, adding new entity types.

**Patch**
- Clarifications or validation fixes with no structural break.
- Examples: documentation fixes, regex corrections, wording updates.

**Stability rule**
- Consumers must ignore unknown fields inside `metadata`.
- Consumers should tolerate new optional fields in future minor versions.
- Core field names and meanings must remain stable across all `1.x.x` versions.

---

## 6. Extension Rules for Agent-Specific Metadata

Agent-specific data belongs in `metadata`.

**Rules**
- `metadata` may extend but never redefine canonical meaning.
- Do not duplicate canonical fields inside `metadata`.
- Use flat or lightly nested keys with agent namespace prefixes if needed.
- Keep canonical analytics dependent only on the standard fields, not `metadata`.
- If a metadata field becomes broadly useful across multiple agents, promote it into the canonical schema in a future minor release.

**Good examples**
- `metadata.follow_up_stage`
- `metadata.days_since_sent`
- `metadata.invoice_due_days`
- `metadata.renewal_window_days`
- `metadata.segment`
- `metadata.view_count`

**Bad examples**
- `metadata.agent_id`
- `metadata.revenue_at_risk`
- `metadata.action_status`

---

## 7. Example Events

### Invoice Enforcement

```json
{
  "schema_version": "1.0.0",
  "event_id": "evt_inv_001",
  "event_time": "2026-04-01T13:00:00Z",
  "event_type": "agent.action_sent",
  "agent": {
    "agent_id": "invoice-enforcement-agent",
    "agent_name": "Invoice Enforcement Agent",
    "agent_version": "v1.0.0",
    "library_version": "v1.0.0",
    "owner": "Revenue Ops"
  },
  "execution": {
    "execution_id": "exec_inv_001",
    "workflow_id": "n8n_wf_101",
    "trigger_type": "invoice_overdue_7d",
    "trigger_id": "trg_inv_001",
    "source_systems": ["billing", "crm", "n8n"],
    "idempotency_key": "invoice_7001_stage2",
    "latency_ms": 645
  },
  "entity": {
    "entity_type": "invoice",
    "entity_id": "inv_7001",
    "parent_entity_type": "account",
    "parent_entity_id": "acct_2001",
    "customer_id": "cust_2001",
    "account_id": "acct_2001",
    "external_refs": [
      { "system": "quickbooks", "id": "QB-INV-7001" }
    ]
  },
  "decision": {
    "decision_code": "SEND_PAYMENT_REMINDER_2",
    "decision_label": "Send second overdue payment reminder",
    "decision_confidence": 0.95,
    "reason_codes": ["INVOICE_OVERDUE", "NO_PAYMENT", "REMINDER_STAGE_2"],
    "policy_version": "invoice-policy-2026-04",
    "leakage_condition": "uncollected_invoice_balance"
  },
  "action": {
    "action_type": "send_payment_reminder",
    "action_status": "sent",
    "action_channel": "email",
    "action_target": "ap@customer.com",
    "scheduled_for": null,
    "completed_at": "2026-04-01T13:00:02Z"
  },
  "revenue": {
    "currency": "USD",
    "revenue_at_risk": 2400,
    "revenue_protected": 1200,
    "revenue_recovered": 0,
    "expected_value": 1200,
    "attribution_window_days": 30,
    "revenue_formula_ref": "invoice_recovery_ev_v1"
  },
  "escalation": {
    "escalation_flag": false,
    "escalation_level": null,
    "escalation_target": null,
    "escalation_reason": null,
    "escalation_due_at": null
  },
  "human_review": {
    "human_review_required": false,
    "human_review_status": "not_required",
    "reviewer_id": null,
    "review_notes": null,
    "reviewed_at": null
  },
  "status": {
    "outcome": "success",
    "terminal": false,
    "error_code": null,
    "error_message": null
  },
  "metadata": {
    "invoice_due_days": 7,
    "reminder_stage": 2
  }
}
```

### Proposal Follow-Up

```json
{
  "schema_version": "1.0.0",
  "event_id": "evt_prop_001",
  "event_time": "2026-04-01T14:00:00Z",
  "event_type": "agent.action_queued",
  "agent": {
    "agent_id": "proposal-follow-up-enforcer",
    "agent_name": "Proposal Follow-Up Enforcer",
    "agent_version": "v1.0.0",
    "library_version": "v1.0.0",
    "owner": "Revenue Ops"
  },
  "execution": {
    "execution_id": "exec_prop_001",
    "workflow_id": "n8n_wf_202",
    "trigger_type": "proposal_silence_72h",
    "trigger_id": "trg_prop_001",
    "source_systems": ["crm", "proposal_platform", "email_log", "n8n"],
    "idempotency_key": "proposal_9001_stage2",
    "latency_ms": 812
  },
  "entity": {
    "entity_type": "proposal",
    "entity_id": "proposal_9001",
    "parent_entity_type": "account",
    "parent_entity_id": "acct_3100",
    "customer_id": "cust_3100",
    "account_id": "acct_3100",
    "external_refs": [
      { "system": "hubspot", "id": "deal_8831" }
    ]
  },
  "decision": {
    "decision_code": "SEND_FOLLOW_UP_2",
    "decision_label": "Queue second proposal follow-up email",
    "decision_confidence": 0.88,
    "reason_codes": [
      "PROPOSAL_SENT",
      "NO_REPLY_72H",
      "NO_OUTREACH_24H"
    ],
    "policy_version": "proposal-policy-2026-04",
    "leakage_condition": "silent_proposal_decay"
  },
  "action": {
    "action_type": "send_follow_up_email",
    "action_status": "queued",
    "action_channel": "email",
    "action_target": "buyer@example.com",
    "scheduled_for": "2026-04-01T14:05:00Z",
    "completed_at": null
  },
  "revenue": {
    "currency": "USD",
    "revenue_at_risk": 4200,
    "revenue_protected": 630,
    "revenue_recovered": 0,
    "expected_value": 630,
    "attribution_window_days": 30,
    "revenue_formula_ref": "proposal_followup_ev_v1"
  },
  "escalation": {
    "escalation_flag": false,
    "escalation_level": null,
    "escalation_target": null,
    "escalation_reason": null,
    "escalation_due_at": null
  },
  "human_review": {
    "human_review_required": false,
    "human_review_status": "not_required",
    "reviewer_id": null,
    "review_notes": null,
    "reviewed_at": null
  },
  "status": {
    "outcome": "success",
    "terminal": false,
    "error_code": null,
    "error_message": null
  },
  "metadata": {
    "follow_up_stage": 2,
    "days_since_sent": 3,
    "view_count": 2
  }
}
```

### Renewal Enforcement

```json
{
  "schema_version": "1.0.0",
  "event_id": "evt_ren_001",
  "event_time": "2026-04-01T15:30:00Z",
  "event_type": "agent.escalated",
  "agent": {
    "agent_id": "renewal-enforcement-agent",
    "agent_name": "Renewal Enforcement Agent",
    "agent_version": "v1.0.0",
    "library_version": "v1.0.0",
    "owner": "Customer Success Ops"
  },
  "execution": {
    "execution_id": "exec_ren_001",
    "workflow_id": "n8n_wf_303",
    "trigger_type": "renewal_30d_no_owner_outreach",
    "trigger_id": "trg_ren_001",
    "source_systems": ["crm", "billing", "product_usage", "n8n"],
    "idempotency_key": "renewal_5501_escalation1",
    "latency_ms": 990
  },
  "entity": {
    "entity_type": "renewal",
    "entity_id": "renewal_5501",
    "parent_entity_type": "subscription",
    "parent_entity_id": "sub_5501",
    "customer_id": "cust_8800",
    "account_id": "acct_8800",
    "external_refs": [
      { "system": "stripe", "id": "sub_live_5501" }
    ]
  },
  "decision": {
    "decision_code": "ESCALATE_OWNER_RENEWAL_RISK",
    "decision_label": "Escalate renewal risk to owner",
    "decision_confidence": 0.91,
    "reason_codes": [
      "RENEWAL_WITHIN_30D",
      "NO_OWNER_OUTREACH",
      "HIGH_ARR",
      "USAGE_DECLINE"
    ],
    "policy_version": "renewal-policy-2026-04",
    "leakage_condition": "unworked_renewal_risk"
  },
  "action": {
    "action_type": "create_escalation",
    "action_status": "escalated",
    "action_channel": "slack",
    "action_target": "sales-manager",
    "scheduled_for": null,
    "completed_at": "2026-04-01T15:30:02Z"
  },
  "revenue": {
    "currency": "USD",
    "revenue_at_risk": 18000,
    "revenue_protected": 3600,
    "revenue_recovered": 0,
    "expected_value": 3600,
    "attribution_window_days": 45,
    "revenue_formula_ref": "renewal_save_ev_v1"
  },
  "escalation": {
    "escalation_flag": true,
    "escalation_level": "high",
    "escalation_target": "sales-manager",
    "escalation_reason": "High ARR renewal at risk with no owner motion",
    "escalation_due_at": "2026-04-01T19:30:00Z"
  },
  "human_review": {
    "human_review_required": true,
    "human_review_status": "pending",
    "reviewer_id": null,
    "review_notes": null,
    "reviewed_at": null
  },
  "status": {
    "outcome": "pending_human",
    "terminal": false,
    "error_code": null,
    "error_message": null
  },
  "metadata": {
    "renewal_window_days": 30,
    "arr_band": "10000_25000",
    "usage_change_pct": -22
  }
}
```

This schema is a good stable `v1` base because it cleanly covers agent identity, entity tracking, decision output, action state, revenue metrics, escalation, human review, and failures without baking in one specific agent’s logic.

If you want, I can turn this into an `EVENT_SCHEMA.md` spec plus a separate machine-readable `event-schema.v1.json` file layout next.
