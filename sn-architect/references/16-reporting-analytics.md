# Reporting & Analytics Architecture (`16-reporting-analytics.md`)

> Guidance for designing the measurement layer — operational reporting, dashboards, and Performance Analytics — without designing the data model into a reporting dead-end.

---

## 1. Reporting Architecture Decision Matrix

| Need | Mechanism | Rationale |
|------|-----------|-----------|
| **Operational lists/counts on demand** | **Reports (`sys_report`) on tables/filtered lists** | Declarative, user-managed, zero build cost. |
| **Cross-table or aggregated views** | **Database views** | Pre-modeled joins for reporting; design them as schema objects (naming, ownership), not ad-hoc per report. |
| **Trend / KPI / target tracking over time** | **Performance Analytics** | Time-series indicators (`pa_indicators`), breakdowns, thresholds, and targets — reporting alone cannot show trends against targets. |
| **Executive dashboards** | **Next Experience dashboards / PA dashboards (`pa_dashboards`)** | Map dashboard persona → data source explicitly (see `03-ui-layer.md` persona mapping). |
| **Scheduled distribution** | **Report/PA scheduled generation + email delivery** | Uses the notification pipeline (`11-notifications.md`); never script custom report emails. |

---

## 2. Design-Time Reporting Standards

1. **Reporting is a data-model consequence.** The reporting layer cannot fix a bad schema: extended-table reporting gets slower with each inheritance level (see `01-data-model.md`), and `glide_list` fields cannot be aggregated or joined (a named anti-pattern in `01` §4). Run Check 4 with the reporting requirement in hand.
2. **Design reports on the leaf table, not the base:** reporting on `task` pulls every child class; scope reports to the specific extended table for both performance and correct record sets.
3. **Indicator design over report sprawl:** for recurring executive metrics, PA indicators with scheduled jobs and thresholds replace hundreds of hand-built reports. One indicator with breakdowns beats twenty reports.
4. **Dashboard persona mapping:** every dashboard names its persona, refresh expectation, and data freshness — stale dashboards presented as live are a trust-destroying failure.
5. **Permissions:** dashboards and reports respect ACLs; design the read surfaces (who sees whose data) as part of the ACL matrix (`05-security-design.md`), not as a reporting afterthought.

---

## 3. Reporting Anti-Patterns

1. **Custom reporting tables / ETL-to-custom-table:** duplicating record data into a custom "reporting" table re-creates the platform's reporting engine badly. Use database views for joins; use the engine for the rest.
2. **Reporting on `glide_list` fields** — `LIKE` scans, no aggregation (see `01-data-model.md`).
3. **Reports on the base `task` table expecting one class** — wrong record sets and slow queries.
4. **PA indicators without scheduled collection jobs** — indicators that never collect show flat lines; the collection schedule is part of the design.
5. **Dashboard sprawl:** dozens of overlapping dashboards with no owner. Every dashboard has an owner and a review cadence, like articles (see `12-knowledge.md`).
6. **Building "real-time" expectations onto scheduled PA collection** — communicate data freshness; PA is trend analytics, not a live feed.

---

## 4. Output in the Architecture Document

Every architecture document must include a **Reporting & Analytics Map**:

| Report / Indicator | Type (Report / PA Indicator / Dashboard) | Persona | Source (table / view / indicator) | Refresh / Collection | Owner |
|--------------------|------------------------------------------|---------|-----------------------------------|----------------------|-------|

Plus: database views required (with join definitions), and the schema notes that protect reporting (indexed reference fields, no `glide_list` on report-filtered fields).