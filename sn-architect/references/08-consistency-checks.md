# Architectural Consistency Checks (`08-consistency-checks.md`)

> Automated validation rules that `sn-architect` must run against any architecture proposal before presenting it to the user.

---

## The 6 Mandatory Consistency Checks

Before presenting an architecture specification, the architect must validate the proposal against these 6 checks. Any failure indicates a design flaw that must be corrected immediately.

---

### Check 1: Orphan Component Check
*Objective: Ensure every proposed component has an active caller, consumer, or trigger.*

- **Rule 1.1:** Every Script Include must be invoked by at least one Business Rule, Flow Action, Scheduled Job, Scripted REST API, Client Script (via GlideAjax), or another Script Include that itself passes this check. *Library & Public API Provision:* If a Script Include is intentionally designed as an exported library/API for external callers or implements a platform Extension Point (`sys_extension_point`), its public API contract, callers, and Restricted Caller Access (RCA) permissions must be explicitly documented in Section 6 and Section 8. No unreferenced "dead code" in design.
- **Rule 1.2:** Every Flow Designer flow must have an explicit trigger (Record Insert/Update, Scheduled, Service Catalog, or Subflow called from script).
- **Rule 1.3:** Every custom event registered in `sysevent_register` must be fired by a script (`gs.eventQueue`) and consumed by an Email Notification or Script Action.
- **Rule 1.4:** Every custom role must be mapped to at least one ACL, Module, or Catalog Item.

---

### Check 2: Security & ACL Completeness Check
*Objective: Prevent data leaks, empty ACLs, or inaccessible tables.*

- **Rule 2.1:** Every custom table must have explicit ACLs defined for `create`, `read`, `write`, and `delete`.
- **Rule 2.2:** Every table with field-level sensitivity must have both table-level ACL (`table`) and wildcard field ACL (`table.*`) before defining specific field rules.
- **Rule 2.3 (Zero Empty ACLs):** No proposed ACL may omit roles, conditions, and scripts. Every ACL must require at least one scoped role or an explicit script evaluation.
- **Rule 2.4:** Client-callable Script Includes must explicitly validate caller session and roles within the script methods.
- **Rule 2.5:** Cross-Scope Privileges and Restricted Caller Access (RCA) records must be documented for any component reading/writing outside its scope.

---

### Check 3: Execution & Transaction Integrity Check
*Objective: Prevent thread blocks, recursion loops, and race conditions.*

- **Rule 3.1:** `current.update()` must NEVER be specified in Before Business Rules (causes recursive loop failure).
- **Rule 3.2:** `current.setAbortAction(true)` must ONLY be placed in Before Business Rules (it does not abort database writes in After, Async, or Flow Designer).
- **Rule 3.3:** Outbound HTTP requests or heavy processing must NEVER run in synchronous Business Rules — this includes **Before, After, and Display** rules (all three execute on the user's session thread and block the transaction). Use Async BR, Flow Designer, or Scheduled Jobs.
- **Rule 3.4:** No client-side synchronous GlideRecord calls. Any client-to-server data fetch must specify `GlideAjax` with an asynchronous callback.

---

### Check 4: Data Model Integrity Check
*Objective: Enforce schema hygiene, naming standards, and indexing.*

- **Rule 4.1:** Table technical name must be singular (`x_<vendor>_<app>_<name>`); table display label must be plural (`Names`).
- **Rule 4.2:** Custom fields in scoped apps must follow `snake_case`. System prefix `sys_` is strictly forbidden for custom fields.
- **Rule 4.3:** Custom applications must NEVER extend `sys_user` (store application-specific user data in a profile table referencing `sys_user`).
- **Rule 4.4:** Reference fields must target valid, existing tables and must be flagged for indexing.
- **Rule 4.5:** If a relationship requires querying, filtering, or reporting joins, it must use an M2M junction table, not a `glide_list`.
- **Rule 4.6:** Scoped custom roles must follow the platform naming standard: `x_<vendor>_<app>.<role_name>` (dot notation separating scope prefix and role name, e.g., `x_acme_loan.admin`).

---

### Check 5: Build Order Dependency Check
*Objective: Guarantee that components can be constructed in linear sequence without forward-referencing errors.*

- **Rule 5.1:** Custom Roles must be defined before ACLs.
- **Rule 5.2:** Parent tables must be created before child tables.
- **Rule 5.3:** Target tables must exist before Reference fields pointing to them.
- **Rule 5.4:** Tables and fields must exist before Business Rules, Flow triggers, and UI Policies that reference them.
- **Rule 5.5:** Script Includes must be created before the Business Rules or Flows that call them.
- **Rule 5.6:** Credentials and Connection Aliases must be created before Outbound REST Messages or Spokes.

---

### Check 6: Out-of-the-Box (OOTB) Duplication Check
*Objective: Protect upgradeability and prevent redundant custom code — and enforce CMDB write integrity and AI entitlement diligence.*

- **Rule 6.1:** Verify that the proposal does not rebuild existing task features (e.g., custom assignment groups, custom work notes, custom SLA tables) if extending `task` satisfies the requirement.
- **Rule 6.2:** Verify that standard platform approval engines (`sysapproval_approver`) are utilized instead of custom approval tables or status flag workarounds.
- **Rule 6.3:** Verify that an official Store Spoke is evaluated before proposing a bespoke Scripted REST or outbound REST integration.
- **Rule 6.4 (IRE Enforcement):** Any design creating or updating Configuration Items must route through `sn_cmdb.IdentificationEngine` (or `createOrUpdateCIEnhanced`, or an IRE-backed transform map via `global.CMDBTransformUtil().identifyAndReconcile()`). Direct inserts into `cmdb` tables — via GlideRecord, transform maps, or REST — **fail review unconditionally.** Verify the scoped IRE prerequisite (`com.snc.cmdb.scoped`) when the caller is a scoped app. (Standard: `14-cmdb.md` §1.)
- **Rule 6.5 (AI Entitlement Gate):** Any design specifying GenAI / Now Assist / AI Agents / Virtual Agent / Predictive Intelligence must include an explicit **entitlement and release verification check** — verified plugin activation and licensed entitlement on the target instance, recorded in the AI Agent Tool & Entitlement Map (`07-architecture-doc-template.md` §6B). Designing against an unverified AI capability **fails review.** (Standard: `13-ai-automation.md` Hard Rule.)

---

## Authoritative Sources
- [ServiceNow Documentation: Coding Best Practices](https://docs.servicenow.com/bundle/xanadu-application-development/page/script/general-scripting/concept/c_CodingBestPractices.html)
- [ServiceNow Documentation: Instance Scan Checks and Suites](https://docs.servicenow.com/bundle/xanadu-application-development/page/administer/instance-scan/concept/instance-scan-suites.html)
