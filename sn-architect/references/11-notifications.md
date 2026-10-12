# Notifications, Events & Messaging Architecture (`11-notifications.md`)

> Guidance for designing outbound notifications (email/SMS/push/chat), the event-driven pipeline behind them, and inbound email response architecture.

---

## 1. Notification Mechanism Selection Matrix

| Scenario | Recommended Mechanism | Rationale |
|----------|-----------------------|-----------|
| **Record lifecycle messages** (created, assigned, state changed) | **Notification record (`sys_notification`) with record trigger** | Declarative, condition-driven, no code. The platform default for standard lifecycle emails. |
| **Cross-record or decoupled messaging** | **Custom event (`sysevent_register`) + `gs.eventQueue()` + event-triggered notification** | Fire-and-forget: the user transaction never waits on email generation. Event carries parameters (`parm1`, `parm2`) to the notification. |
| **Delayed / scheduled messages** | **`gs.eventQueueScheduled()` with a future timestamp** | Queues an event for a future processing time (e.g., follow-up reminder 3 days after creation) without a Scheduled Job. |
| **Multi-step process messaging** (approval requested, fulfillment updates) | **Flow Designer steps/notifications** | Ties messaging to the process engine so it follows flow execution history and error handling. |
| **Messages on tables without standard lifecycle** | **Event-triggered notification** | Record triggers only fire on the record's own changes; events decouple the trigger from the target table. |

**Architectural rule of thumb:** notifications must never be generated inside the synchronous user transaction. Outbound email is already asynchronous by design (queued in `sys_email` and delivered by the scheduler), but the *trigger* must also be cheap — record conditions or events, not script loops.

---

## 2. The Event-Driven Pipeline

```
[Business Rule / Flow / Script]
        │  gs.eventQueue('x_acme_loan.approval_requested', gr, parm1, parm2)
        ▼
[Event Queue (sysevent)]  ── asynchronous, survives transaction abort decisions
        │
        ▼
[Event Registry (sysevent_register)]  ── declares the event and its processing queue
        │
        ├──► [Notification (sys_notification), event-triggered] ──► sys_email ──► SMTP
        └──► [Script Action / other processors] ──► custom server-side reactions
```

### Event Design Standards
1. **Register every custom event** in `sysevent_register` with a scoped name (`x_<vendor>_<app>.<event_name>`).
2. **Every event must have both a producer and a consumer** (enforced by Check 1.3 in `08-consistency-checks.md`): fired by script (`gs.eventQueue`) and consumed by a Notification, Script Action, or Flow trigger.
3. **Pass context via `parm1`/`parm2`, not by re-querying:** events queue record `sys_id` implicitly; use parameters for extra context so the consumer does not depend on the record still existing in the same state.
4. **Do not fire events from Before Business Rules:** the record is not committed; the consumer may read data that never persisted. Fire from After rules, async rules, or Flows.
5. **Event weights:** use `gs.eventQueueScheduled` or consumer-side conditions to prevent notification storms from bulk operations (imports, mass updates).

---

## 3. Notification Record Design Standards (`sys_notification`)

| Design Element | Standard |
|----------------|----------|
| **Trigger type** | Record insert/update with conditions (declarative) or event-based. Prefer record triggers for simple lifecycle messages; events for decoupled flows. |
| **Recipients** | Never hardcode individual users. Use: groups, users referenced by record fields (`assigned_to`, `opened_by`), or event parameters. Hardcoded addresses in scripts or notification fields are an anti-pattern (environment-dependent, unmaintainable). |
| **Subject / Body** | Use field name variables (`${number}`, `${assigned_to.name}`) rather than scripted strings wherever possible. Keep templates declarative for upgrade safety. |
| **Table binding** | The notification's table must match the table of the triggering record or event. |
| **Who receives** | Respect the platform's subscription and preference model — users can disable notifications they receive; do not attempt to bypass user preferences with custom send scripts. |

### Channels Beyond Email
- **SMS / voice:** delivered to a user's notification devices; requires a telephony/notify provider integration. Evaluate the device model before designing custom SMS scripts.
- **Mobile push:** platform push notifications to the Now Mobile / Agent app.
- **Microsoft Teams / Slack:** prefer official Store spokes or notification integrations over custom webhook scripts.
- Design the channel per persona, not per message: fulfillers on mobile need push; external customers need email; approvers may need chat nudges plus email for the formal record.

---

## 4. Inbound Email Architecture (`sys_inbound_email_action`)

Inbound email actions script how the instance responds to incoming email — analogous to business rules with conditions on a target table.

1. **Watermark-based threading:** emails carrying an instance watermark (`ref:` tokens) update the existing record; emails without one follow the default create path (by default, new emails attempt to create an incident). Design your custom inbound actions around this contract.
2. **Two action types:** Record action (create/update a record, set fields) and Email reply (respond to the sender).
3. **Classification:** the platform classifies inbound mail as Forward, Reply, or New using watermark, reply prefixes, and thread-index headers.
4. **Inbound flows take priority:** if an email matches an inbound email *flow*, that flow handles it instead of inbound email actions. When designing intake, choose one mechanism per entry point and document which.
5. **Architecture guardrail:** inbound email is an *integration entry point* — apply the same security review as any API: validate sender identity, restrict which addresses can trigger which actions, and never trust unvalidated email body content for field updates without sanitization.

---

## 5. Notification Architecture Anti-Patterns

1. **Custom email-sending scripts:** calling email APIs directly from scripts instead of using the notification engine bypasses templates, preferences, tracking, and audit. Use notifications/events.
2. **Notifications in synchronous transactions:** generating heavy notification logic inside Before rules or user-triggered scripts adds latency to every save.
3. **Hardcoded recipient addresses:** breaks across environments and ignores the preference/subscription model.
4. **Notification sprawl:** every state change × every role = email fatigue and opt-outs. Design a per-persona message map; if two notifications target the same persona about the same lifecycle event, merge them.
5. **Ignoring digest/quiet-hours:** high-frequency operational tables should leverage user digest and quiet hours settings instead of custom throttling scripts.
6. **Inbound actions without sender validation:** any inbound action that creates/updates records from unvalidated senders is an unauthenticated write surface — treat it like a public API (see `05-security-design.md`).

---

## 6. Output in the Architecture Document

When the design includes notifications or messaging, the architecture document must include a **Notification Matrix** (template §6A):

| Notification Name | Trigger (Record condition / Event) | Recipients (dynamic/group/field) | Channel | Persona |
|-------------------|-----------------------------------|----------------------------------|---------|---------|
| `Approval Requested` | Event `x_acme_loan.approval_requested` | Approver (field), Loan group | Email + Chat | Fulfiller / Approver |
| `Request Submitted Confirmation` | Record inserted | `opened_by` | Email | End User |

Plus: list of custom events with producer and consumer columns (satisfies Check 1.3), and the inbound email entry points with their security validation strategy.

---

## Authoritative Sources
- [ServiceNow Documentation: Notifications](https://docs.servicenow.com/bundle/xanadu-platform-administration/page/administer/notification/concept/c_Notifications.html)
- [ServiceNow Documentation: Event Registry](https://docs.servicenow.com/bundle/xanadu-platform-administration/page/administer/platform-events/concept/events.html)
- [ServiceNow Documentation: Inbound Email Actions](https://docs.servicenow.com/bundle/xanadu-platform-administration/page/administer/notification/concept/c_InboundEmailActions.html)