# CMDB & CSDM Architecture (`14-cmdb.md`)

> Guidance for any architecture that reads from or loads into the CMDB — Identification & Reconciliation, CSDM alignment, and data source strategy.

---

## 1. The Iron Rule: Never Write to the CMDB Without the IRE

Loading CIs by directly inserting into `cmdb` tables creates duplicate CIs, destroys data-source accountability, and corrupts reconciliation. Every CI load path must go through the **Identification and Reconciliation Engine (IRE)**.

### Verified IRE Access Patterns

| Pattern | When | Mechanism |
|---------|------|-----------|
| **Import Set + IRE (transform)** | Single-table loads, no complex relationships | Transform map calling `global.CMDBTransformUtil().identifyAndReconcile()` |
| **Scripted IRE** | Multi-class payloads with relationships | Scoped API `sn_cmdb.IdentificationEngine.createOrUpdateCI(source, input)` — payload: `{"items": [{"className": "cmdb_ci_win_server", "values": {...}}]}` with `relations` arrays |
| **Enhanced IRE** | Partial payloads, duplicate handling, commit summaries | `createOrUpdateCIEnhanced(source, input, options)` |
| **IRE REST API** | External systems pushing directly | Push-style ingestion; note the platform only certifies *pull* mechanisms — prefer pull architectures where certification matters |

**Scoped-app prerequisite:** the scoped IRE API requires the **`com.snc.cmdb.scoped`** plugin (Configuration Management for Scoped Apps) — verify it is activated before designing a scoped CMDB integration.

**Source discipline:** the `source` string passed to IRE is the data-source identity that drives reconciliation precedence. Choose it deliberately and never reuse a source across unrelated integrations.

---

## 2. CSDM Alignment

The Common Service Data Model defines where data lives and who owns it:

| CSDM Domain | Content | Architect's Responsibility |
|-------------|---------|---------------------------|
| **Foundation** | Location, vendor, cost centers, generic entities | Verify foundation data quality before any dependent design. |
| **Categorization** | Application/service/infrastructure class structure | New CI classes are created via **CI Class Manager**, following CSDM — never by ad-hoc table extension. |
| **Presentational** | Business/service/technical views (service mappings) | Design which service views stakeholders need — this determines required relationships. |

If the architecture touches service mapping or application services, treat the service model design as a first-class deliverable, not a reporting afterthought.

---

## 3. Data Source Strategy Matrix

| Source | Mechanism | Notes |
|--------|-----------|-------|
| **Discovery (network)** | Mid Server + platform Discovery | Certified pull; owns infrastructure CIs — coordinate precedence before loading overlapping data. |
| **Service Graph Connectors** | Certified third-party ETL into IRE | Preferred vendor-data path; check the Store for a certified connector before building custom. |
| **External push (REST)** | IRE REST API | Works, but platform does not certify external push — document the trade-off. |
| **Manual / spreadsheet** | Import Set + IRE transform | Acceptable for reference data; unacceptable as an ongoing operational feed. |
| **Custom application data** | Scoped IRE API from app logic | Only for CIs the application genuinely owns; otherwise read, don't write. |

---

## 4. Reconciliation & Data Precedence

1. **Data precedence rules** decide which source wins per attribute. Before any new source starts writing, verify its precedence position — a low-authority source silently updating critical attributes (or a high-authority source overwriting Discovery) is the classic CMDB corruption path.
2. **Attribute-level, not table-level:** precedence is designed per attribute class; document the precedence table for every integration in the architecture document.
3. **Identification rules:** CI identity is established by class-specific identifiers (e.g., serial number, name+IP). Feeds that cannot supply the class's identification criteria will duplicate — validate payloads against identification criteria during design, not in production.

---

## 5. CMDB Health as an Architectural Deliverable

An architecture that loads the CMDB owns its health:

- **Completeness:** required fields populated per class.
- **Compliance:** classes follow class standards (CI Class Manager rules).
- **Correctness:** relationships match reality; no orphan or stale CIs.
- Use the platform's CMDB health dashboards as the ongoing measure; define the health gate for go-live in the architecture document (e.g., zero duplicate CIs from the new feed for 2 weeks).

---

## 6. CMDB Anti-Patterns

1. **Direct GlideRecord inserts into `cmdb` tables** from scripts or transform maps — bypasses identification, creates duplicates, breaks reconciliation. Always IRE.
2. **Loading relationships as custom fields** instead of using standard CI relationships (`Contains`, `Runs on::Runs on`, etc.).
3. **Custom tables extending `cmdb_ci` outside CI Class Manager** — violates CSDM and breaks class-based identification (see `01-data-model.md`).
4. **Ignoring source precedence:** new feed quietly overwrites authoritative data.
5. **Treating CMDB health as an ops problem, not a design problem:** the go-live gate and health ownership must appear in the architecture document.

---

## 7. Output in the Architecture Document

Every architecture document touching the CMDB must include:

- **Load path table:** source → mechanism (IRE pattern) → target classes → identification criteria available → precedence position.
- **Plugin/entitlement verification:** `com.snc.cmdb.scoped` (scoped IRE), Discovery, SG Connectors — each with a checked/verified box.
- **CSDM domain mapping:** which classes are created/modified, via CI Class Manager, mapped to CSDM domains.
- **Health gate:** the measurable CMDB health criteria for go-live acceptance.