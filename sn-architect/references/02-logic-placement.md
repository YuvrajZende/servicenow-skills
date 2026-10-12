# Logic Placement & Execution Architecture (`02-logic-placement.md`)

> Guidance for determining where code and configuration should execute across server, client, flow engine, and scheduler tiers.

---

## 1. Logic Placement Decision Matrix

| Requirement / Scenario | Recommended Mechanism | Secondary Choice | Architectural Rationale |
|------------------------|-----------------------|------------------|-------------------------|
| **Data Validation & Aborting Save** | **Before Business Rule** | Data Policy | `current.setAbortAction(true)` only stops the database transaction during Before rules. |
| **Deriving Values on Same Record** | **Before Business Rule** | — | Updates the in-memory record directly; avoids a second database round-trip or recursion. |
| **Updating Related Records** | **After Business Rule** | Flow Designer | Runs after the primary record commits successfully; ensures transactional integrity. |
| **Long-Running / Heavy Calculations** | **Async Business Rule** | Flow Designer / Scheduled Job | Frees the end-user thread immediately; runs asynchronously via the platform scheduler queue. |
| **Multi-Stage Approvals & Orchestration** | **Flow Designer** | — | Visual, auditable, built-in approval engines, SLA timers, and execution history. |
| **Reusable Business Logic** | **Script Include** | — | Single source of truth. Business Rules should be 5-line wrappers delegating to Script Includes. |
| **Client Form Interaction & Read-Only Logic** | **UI Policy** | Client Script | No-code UI Policies run faster and adapt to mobile and configurable workspaces natively. |
| **Complex Client-Side Logic & Dynamic Prompts** | **Client Script (onChange/onSubmit)** | — | Use `g_form` and `g_user` APIs. Never make synchronous GlideRecord calls from client scripts. |
| **Client-to-Server Asynchronous Lookups** | **GlideAjax + Client-Callable Script Include** | — | Standard secure pattern for fetching server data from client forms without freezing the UI. |
| **Batch Background Processing (Bulk Records)** | **Scheduled Script Execution** | — | Batched processing with `gr.setLimit()` prevents memory exhaustion and worker lockups. (For external bulk *ingestion*, use Import Sets / RTE — see `04-integration-design.md`.) |
| **High-Volume Asynchronous Decoupling** | **Event Queue (`gs.eventQueue`)** | Flow Designer | Dedicated event processor isolates heavy operations from user transaction threads. |

---

## 2. Synchronous vs Asynchronous Nuances

Understanding execution thread models is critical for avoiding race conditions and performance degradation.

### Business Rule Execution Flow
```
User clicks Save
       │
       ▼
[Before Business Rules] ── (Synchronous: Can modify current.*, can setAbortAction)
       │
       ▼
[Database Write] ──────── (Record written to database buffer, not yet committed)
       │
       ├──► [After Business Rules] ── (Synchronous: Runs in session thread after write, inside the SAME transaction before commit)
       │
       ▼
[Database Commit] ─────── (Transaction committed to database)
       │
       └──► [Async Business Rules] ── (Asynchronous: Queued into sys_trigger / scheduler after commit)
```

### Flow Designer Execution Nuance (Not Strictly Async)
Many architects mistakenly assume Flow Designer is always asynchronous. In reality:
1. **Foreground vs Background Triggers:** Record-triggered flows can be configured to run in **Foreground** (runs synchronously within the session transaction) or **Background** (queued to the flow engine worker pool).
2. **Subflow Synchronous Execution:**
   - Scripted execution via `sn_fd.FlowAPI.executeSubflow('scope.subflow_name', inputs)` executes **synchronously** in-thread and returns outputs immediately.
   - Scripted execution via `sn_fd.FlowAPI.startSubflow('scope.subflow_name', inputs)` executes **asynchronously** in the background.
3. **Flow Designer Action Steps:** Steps within a subflow execute sequentially, with parent steps pausing until child steps complete.

---

## 3. Flow Designer vs Business Rule Selection

| Evaluation Factor | Flow Designer | Business Rule |
|-------------------|:-------------:|:-------------:|
| **Maintainability for Process Owners** | ✅ Visual, readable steps | ❌ Requires developer code review |
| **Audit Trail & Execution Tracking** | ✅ Detailed Execution Context | ❌ Requires custom `gs.info` logs |
| **Approval Life Cycles & Wait Conditions** | ✅ Native actions & timers | ❌ Cumbersome procedural scripts |
| **High-Throughput Microsecond Updates** | ❌ Engine overhead per execution | ✅ Lightweight, near-zero latency |
| **Transaction Abort (`setAbortAction`)** | ❌ Cannot abort database write | ✅ Native capability in Before BR |
| **IntegrationHub Spoke Actions** | ✅ Native drag-and-drop actions | ❌ Requires manual RESTMessageV2 scripts |

**Architectural Rule of Thumb:**  
- Use **Before Business Rules** for synchronous data hygiene, field defaults, and transaction validation.
- Use **Flow Designer** for multi-step approvals, fulfillment tasks, notifications, and cross-system orchestration.

---

## 4. Script Include Architectural Guidelines

1. **Class-Based Structure:** Always create Script Includes as prototype classes:
   ```javascript
   var LoanRequestUtils = Class.create();
   LoanRequestUtils.prototype = {
       initialize: function() {},
       
       calculateRiskScore: function(loanGr) {
           // Business logic here
       },
       
       type: 'LoanRequestUtils'
   };
   ```
2. **Access Control & Client-Callable:**
   - Mark `Client-callable: false` for internal server-side utilities.
   - For `Client-callable: true` (GlideAjax), inherit from `AbstractAjaxProcessor` and explicitly validate permissions/roles inside every public method.
3. **Application Scope Access:** Set `Accessible from: All application scopes` only if cross-scope consumption is strictly required; otherwise restrict to `This application scope only`.

---

## 5. Missing Actors: UI Actions & Flow Resilience

**UI Actions** are the platform's button/link/context-menu mechanism and a first-class logic placement candidate:
- **Use for:** form/list actions that change record state (e.g., "Approve", "Escalate", "Generate Report") — visible affordances with conditions.
- **Placement rule:** the action's script should be a thin wrapper delegating to a Script Include (same rule as business rules); its condition field governs visibility; design both a form button and a list action where both contexts make sense.
- **Workspace caveat:** classic UI Action buttons do not automatically appear in Configurable Workspace — verify the Workspace surface for every UI Action in the design (see `03-ui-layer.md`).

**Flow Designer error handling** is part of logic placement, not an afterthought:
- Every flow with outbound calls must define an **error path** (flow error handling / error subflows) — what happens when the REST step fails after retries (see `04-integration-design.md`).
- **Decision tables** belong in flows where mapping logic is data-administered (routing, prioritization) rather than scripted.

**Caching:** scripts caching data in `gs.getProperty`-backed properties or custom caches must consider multi-node instances — cache invalidation across nodes is not instant; design for stale-window tolerance instead of assuming instant consistency.

---

## Authoritative Sources
- [ServiceNow Documentation: Business Rules](https://docs.servicenow.com/bundle/xanadu-application-development/page/script/business-rules/concept/c_BusinessRules.html)
- [ServiceNow Documentation: Flow Designer Architecture](https://docs.servicenow.com/bundle/xanadu-build-workflows/page/administer/flow-designer/concept/flow-designer.html)
- [ServiceNow Developer Guide: GlideAjax API](https://developer.servicenow.com/dev.do#!/reference/api/latest/client/c_GlideAjaxAPI)
