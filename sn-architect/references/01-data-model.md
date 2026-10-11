# Data Model Architecture (`01-data-model.md`)

> Guidance for designing ServiceNow schemas, relationships, table inheritance, and naming standards.

---

## 1. Naming Standards & Conventions

Adhere strictly to platform naming conventions:

| Artifact | Convention | Example |
|----------|------------|---------|
| **Scoped Table (Technical)** | `x_<vendor>_<app>_<singular_noun>` | `x_acme_loan_request` |
| **Global Table (Technical)** | `u_<singular_noun>` | `u_building_access_pass` |
| **Table Display Label** | **Plural noun** | `Loan Requests`, `Building Access Passes` |
| **Scoped Field (Technical)** | Plain `snake_case`, **no prefix** — the scope lives in the table name, so scoped columns are not auto-prefixed | `requested_amount`, `target_completion_date` |
| **Reference Field** | Singular noun representing target record | `assigned_to` (targets `sys_user`), `department` |
| **System Columns** | Reserved prefix `sys_` (never use for custom fields) | `sys_id`, `sys_created_on`, `sys_updated_by` |

*Note on table labels:* Platform convention pairs a singular table name with a plural display label. For example: table `incident` has label "Incidents"; table `change_request` has label "Change Requests".

---

## 2. Extend Task vs Standalone Table — Decision Matrix

Inheriting from `task` brings enormous built-in power but also carries table weight and inherited logic. Use this matrix:

| Architectural Signal | Extend `task` | Standalone Table | Rationale |
|----------------------|:-------------:|:----------------:|-----------|
| Represents actionable work with an assignment lifecycle | ✅ | | Inherits `assignment_group`, `assigned_to`, work notes, and activity formatter. |
| Requires Service Level Agreements (SLAs) | ✅ | | SLA engine is pre-wired to `task` tables. |
| Needs multi-stage approvals | ✅ | | Approval engines integrate directly with `task`. |
| Must appear in unified fulfiller views ("My Work", "My Groups Work") | ✅ | | Fulfiller modules query the base `task` table. |
| Represents master, reference, or lookup data | | ✅ | Lookup data has no lifecycle, states, or fulfillment. |
| High-volume transactional logging or external event ingest | | ✅ | High volume (e.g., IoT logs, API audit trails) will bloat the shared `task` table. |
| Clean, unburdened schema without task business rules | | ✅ | `task` has hundreds of out-of-the-box business rules that execute on every insert/update. |
| Pure staging table for ETL/Import Sets | | ✅ | Staging tables must extend `sys_import_set_row`, not `task`. |

---

## 3. Platform Base Tables to Reuse vs Avoid

| Base Table | Extensibility Rule | Notes & Constraints |
|------------|-------------------|---------------------|
| `task` | Extensible for work items | Avoid deep hierarchies (e.g., `task` ➔ `incident` ➔ custom). Max depth recommended is 1 level below `task`. |
| `cmdb_ci` | Extensible for Configuration Items | Always extend via **CI Class Manager**. Follow Common Service Data Model (CSDM) standards. |
| `interaction` | Extensible or reusable for intake | Standard landing table for omnichannel chat, phone, and walk-up intake before converting to a task. |
| `sys_user` | **NEVER EXTEND** | `sys_user` is a core system table. Extending it breaks authentication, licensing, and reference integrity. To add user attributes, add custom fields or create a related 1:1 `user_profile` table. |
| `core_company` | Reusable master data table | Store vendors, customers, manufacturers. Do not create separate "vendor" or "client" tables. |

---

## 4. Relationship Patterns

Select relationship mechanisms based on dot-walking, aggregation, and volume needs:

```
+--------------------------------------------------------------------------------+
|                             Relationship Selector                              |
+--------------------------------------------------------------------------------+
| Need dot-walking & reporting?                                                  |
|   ├── YES: 1-to-Many?                                                          |
|   │     └── Use Reference Field (stores sys_id on child record)                |
|   └── YES: Many-to-Many with attributes or reporting needs?                    |
|         └── Use M2M Junction Table (two reference fields + relationship fields)|
|                                                                                |
| Simple multi-select display only (no dot-walking or reporting joins needed)?    |
|   └── Use glide_list (comma-separated sys_ids, e.g., "watch_list")             |
+--------------------------------------------------------------------------------+
```

### Relationship Comparison

| Type | Platform Representation | Dot-Walking | Reporting Join | Indexing Recommendation |
|------|-------------------------|:-----------:|:--------------:|-------------------------|
| **Reference Field** | Foreign key (`sys_id`) | ✅ | ✅ | Always index reference fields via Table Dictionary. |
| **M2M Junction Table** | Dedicated table with 2 reference fields | Via junction | ✅ | Compound unique index on (`parent_sys_id`, `child_sys_id`). |
| **glide_list** | Comma-delimited string of `sys_ids` | ❌ | ❌ | Cannot be indexed; queries use `CONTAINS` (table scan). |

---

## 5. Data Model Anti-Patterns to Prevent

1. **Deep Inheritance Chains:** Creating `custom_task` extending `incident` which extends `task`. Every query on the child joins all ancestor tables. Limit inheritance to 1 level from `task`.
2. **Extending `sys_user`:** Breaks identity provider sync, user lookups, and licensing audits.
3. **Using `glide_list` for Core Business Logic:** If business rules or reports must filter on the relationship, a `glide_list` requires expensive `LIKE` or `CONTAINS` queries. Use an M2M table instead.
4. **Unindexed Foreign Keys:** Forgetting to index reference fields causes table scans on every related list query.
5. **Over-Normalizing Simple Attributes:** Creating separate lookup tables for 3 static choices. Use Choice Lists (`sys_choice`) for static options; use tables only for dynamically administered master data.

---

## 6. Data Lifecycle & Scale

The data model is not done when the schema is drawn — design how data ages:

| Concern | Mechanism | Design Standard |
|---------|-----------|-----------------|
| **Record volume growth** | Data archiving / retention policies | High-volume tables (see `10-requirements.md` for `task_sla` volume math) need an archiving decision at design time, not after the first performance incident. |
| **History & audit** | Platform audit tables (`sys_audit`) | Audit is controlled per table/field; high-volume tables may need audit exclusions — decide deliberately, and never query `sys_audit` directly in scripts (see `09-anti-patterns.md`). |
| **Multilingual data** | Translated fields | If personas work in multiple languages, field labels *and* data (choices, text) need translated-field design up front — retrofitting is painful. |
| **Date/time correctness** | `GlideDateTime` | All server date math in UTC-backed Glide APIs; never naive `new Date()` (see `09-anti-patterns.md`). Document the timezone strategy alongside SLAs (`10-requirements.md`). |
| **Cross-table reporting** | Database views | Pre-modeled joins for reporting (see `16-reporting-analytics.md`) — designed as schema objects with owners. |
