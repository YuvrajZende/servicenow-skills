# SLA & Service Commitment Architecture (`10-requirements.md`)

> Guidance for designing Service Level Agreements (SLAs), OLAs, and measurement models on `task`-extending tables — before promising delivery timelines in an architecture.

---

## 1. SLA Design Decision Matrix

| Scenario | Mechanism | Rationale |
|----------|-----------|-----------|
| **Formal response/resolution commitments with breach tracking** | **SLA Definition (`contract_sla`) + Task SLA engine** | The platform-native measurement engine: attaches to any `task`-extending table, tracks stages, and drives breach reporting automatically. |
| **Internal team commitments (support tiers)** | **OLA** (SLA definition with type OLA) | Same engine, distinct commitment class so reporting separates customer-facing from internal targets. |
| **Vendor-backed commitments** | **UC** (underpinning contract) | Ties internal obligations to supplier obligations. |
| **Simple "due date" with no formal measurement** | **Due date field + flow timer** | Do not spin up an SLA for a cosmetic due date; SLAs add engine load and `task_sla` records per task. |
| **Step-level process deadlines inside a flow** | **Flow Designer timer/wait conditions** | Process-internal deadlines belong to the process engine, not the SLA engine. |

**Rule of thumb:** every SLA creates a `task_sla` record per task per SLA. On high-volume tables, each additional SLA multiplies storage and engine evaluations — design the minimum set that satisfies the contract.

---

## 2. SLA Definition Anatomy (checklist for every `contract_sla`)

| Element | Design Standard |
|---------|-----------------|
| **Table** | Must be a `task`-extending table (SLA engine is pre-wired to `task`). |
| **Type** | SLA (customer-facing), OLA (internal), or UC (vendor) — pick deliberately; reporting filters on it. |
| **Target** | Duration-based (fixed) or schedule-based (business hours). Schedule-based targets require a `sys_schedule` and a deliberate **timezone strategy** — whose clock does the commitment run on? |
| **Start conditions** | Defined by record conditions. Use retroactive start deliberately when the SLA should begin at an earlier timestamp (e.g., email receipt time) — it can backdate breach risk. |
| **Pause conditions** | Every pause condition must have a matching unpause path; a SLA that pauses forever silently never breaches (a classic audit failure). |
| **Stop conditions** | Typically terminal state; align with the table's close semantics (closed vs resolved vs cancelled). |
| **Stage/breach notifications** | Drive via SLA stages and events or flows (see `11-notifications.md`), not custom scripts polling `task_sla`. |
| **Workflow** | SLA workflows attach actions to SLA stages; for modern designs prefer Flows triggered on `task_sla` stage changes over legacy SLA workflows where the release supports it. |

---

## 3. SLA Anti-Patterns

1. **Pause-forever SLAs:** pause conditions with no exit — the SLA never breaches and the commitment is fictional.
2. **SLA sprawl:** dozens of overlapping SLAs on one table. Consolidate per persona/commitment; one response + one resolution SLA usually covers the contract.
3. **SLAs on non-task tables:** the engine does not measure them; do not hand-roll "SLA-like" date math for custom tables that could extend `task` instead (see `01-data-model.md` — this is an OOTB-first violation).
4. **Timezone ambiguity:** schedule-based SLAs without a documented timezone decision produce "whose midnight?" disputes. Decide and document.
5. **Retroactive start without stakeholder sign-off:** backdating start times moves breach risk before the user saw the ticket; get business agreement in writing.
6. **Reporting on `task_sla` directly instead of using SLA reporting:** the engine provides stage-based reporting; custom aggregations drift from platform semantics.

---

## 4. Output in the Architecture Document

Every architecture document for a work-management app must include an **SLA Matrix**:

| SLA Name | Type | Table | Target | Schedule / Timezone | Start | Pause | Stop | Breach Action |
|----------|------|-------|--------|---------------------|-------|-------|------|---------------|
| `Response Time` | SLA | `x_acme_loan_request` | 4h business | 8-5 Mon-Fri, tenant TZ | Created | Awaiting customer | In progress / Closed | Notify group + escalate |

Plus: the timezone decision, the pause/unpause pairings, and the expected `task_sla` volume impact (tasks/day × SLAs per task) feeding the archiving strategy in `01-data-model.md`.