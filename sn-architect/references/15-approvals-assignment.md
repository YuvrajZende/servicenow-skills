# Approvals, Routing & Assignment Architecture (`15-approvals-assignment.md`)

> Guidance for designing approval flows, record routing, and assignment strategy — the mechanics that decide who acts on a record and when.

---

## 1. Approval Mechanism Selection Matrix

| Scenario | Mechanism | Rationale |
|----------|-----------|-----------|
| **Approval on a `task`-extending table, standard patterns** | **Approval engine (`sysapproval_approver`) driven by approval rules** | Declarative rule-based approver determination; approvals appear in the standard My Approvals surface and mobile. |
| **Approval inside a multi-step process** | **Flow Designer approval actions (Ask for Approval)** | Process-native: wait states, reminders, and branching are flow steps; execution history is auditable. |
| **Approval on a non-task custom table** | **Flow Designer approval actions** (or the approval engine if the table is approval-enabled) | Flow approvals work on any table; the classic engine is task-oriented. |
| **Ad-hoc/scripted approval generation** | **Script creating `sysapproval_approver` records** | Justified only when approver logic is too dynamic for rules/flows — keep in a Script Include (see `02-logic-placement.md`). |
| **Multi-level / chained approvals** | **Flow Designer with approval stages or approval rules with sequence** | Avoid script loops that poll for approval state — use the engine's wait semantics. |

**Check 6 tie-in:** custom approval tables and status-flag workarounds are explicitly rejected by Rule 6.2 — the platform approval engine (`sysapproval_approver`) is the default.

---

## 2. Approval Design Standards

1. **Approvers resolve dynamically:** by manager chain, group membership, or record fields — never hardcoded named users (environment-dependent, unmaintainable; see `09-anti-patterns.md` §5).
2. **Delegation:** design for approver absence — the platform supports approver delegation; document the delegation policy (duration, self-approval prevention).
3. **Reminders and escalation:** unapproved requests need a time-based path (reminders → escalation → auto-approve/auto-reject policy decision). Silence is a design decision and must be made explicitly.
4. **Approval state machine:** approved/rejected/requested/cancelled — never encode approval state into custom string fields; the approver record *is* the state.
5. **Consolidation:** multiple approvals on one record should be visible as a related list on the target record — design the approval view, not just the generation.

---

## 3. Assignment & Routing Architecture

```
[Record Created]
      │
      ▼
[Assignment rules?] ──► condition-matched ──► set group/assignee
      │ no match
      ▼
[Data lookup / skills-based routing?] ──► route by category/skill/region
      │ no match
      ▼
[Group queue with manual triage] ──► documented dispatch process
```

| Mechanism | Use When | Watch Out |
|-----------|----------|-----------|
| **Assignment rules** | Deterministic condition→group mapping | Rule explosion; order matters — document precedence. |
| **Data lookup rules** | Category/subcategory → group/assignment mapping tables | Prefer over assignment rules when mapping is data-driven and administered by process owners. |
| **Skills-based routing** | Fulfiller skills matter (language, expertise) | Requires skills data hygiene; unmanaged skills lists rot. |
| **On-call scheduling + escalation** | Time-of-day coverage obligations | Tie to escalation timers (see `11-notifications.md` for the event pipeline that drives escalations). |
| **Manual dispatch** | Complex triage, low volume | Explicitly design the queue view and dispatch SLA; don't leave it undefined. |

**Architectural rule:** every record type must have exactly one defined routing outcome — no record should land in an undefined state. Validate this with Check 1 (orphan check) logic: every queue/group the design references must have a defined intake path.

---

## 4. Approval & Routing Anti-Patterns

1. **Custom approval tables / status flags** — violates Rule 6.2; breaks My Approvals, delegation, and mobile approval surfaces.
2. **Polling scripts for approval completion** — use engine wait semantics (flow approvals / approval rules), not scheduled scripts checking state.
3. **Hardcoded approver names or sys_ids** — breaks cross-environment (see `09-anti-patterns.md` §5).
4. **Assignment rules without precedence documentation** — first-match order is load-order dependent; document it.
5. **Approvals with no timeout path** — requests that stall forever in "requested" state; always pair approvals with reminder/escalation policy.
6. **Assigning to individuals by rule** — routes to people, not roles; assign to groups, let group load-balancing/fulfiller choice pick the person.

---

## 5. Output in the Architecture Document

Every architecture document must include:

- **Approval Matrix:** each approval step — target table, mechanism (rule/flow/script), approver resolution (dynamic source), timeout/escalation policy, auto-approve/reject decision.
- **Routing map:** record type → routing mechanism chain → fallback queue; plus the skills/on-call requirements if used.