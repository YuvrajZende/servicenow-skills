---
name: sn-architect
description: Architectural orchestrator for ServiceNow solutions. Use when asked to "design a ServiceNow app", "architect an integration", "create a table model", "design ACLs", or scope features — before generating any code, config, or XML metadata. Enforces OOTB-first evaluation, explicit security, clean schemas, and verified execution boundaries.
license: MIT
---

# ServiceNow Architect (`sn-architect`)

Architect end-to-end ServiceNow solutions from plain-language requirements **by reasoning and constraint-testing, not by guessing or recalling generic templates.**

**Tradeoff:** This skill biases toward upgrade safety, platform alignment, and security over quick-and-dirty scripting. For single-line fixes, use surgical mode.

---

## Core Guidelines (Karpathy Heuristics for ServiceNow)

### 1. Don't Rebuild ServiceNow
**Check OOTB first. Configure before customizing. Never extend `sys_user`.**
- The most expensive mistake in ServiceNow is custom-building what the platform already ships.
- Before proposing any custom table, evaluate base tables (`task`, `interaction`, `planned_task`, `cmdb_ci`), Store apps, and pre-built spoke actions.
- Avoid speculative custom tables if adding 2–3 fields to an existing table solves the business problem.
- **The test:** Can 80%+ of this requirement be delivered using a standard Catalog Item, form fields, and a Flow Designer flow? If yes, reject the custom table.

### 2. Never Assume Default Deny
**Explicit security on every boundary. Zero empty ACLs.**
- `glide.sm.default_mode` varies across instances. Never assume missing ACLs = denied.
- Every custom table must have explicit CRUD ACLs (`create`, `read`, `write`, `delete`) and field wildcards (`table.*`).
- Never create an Empty ACL (no role, condition, or script = open to all authenticated users).
- **The test:** If an unauthenticated user or an unauthorized fulfiller accesses the Table API for your custom table, does an explicit ACL deterministically block them?

### 3. Simplicity Over Schema Sprawl
**Minimum tables that solve the domain. No deep inheritance.**
- Keep inheritance chains to a maximum depth of 1 below `task`.
- Reference fields for 1:N. M2M tables for N:N with attributes or reporting joins. Never use `glide_list` for core joins.
- Use singular technical table names (`x_acme_loan_request`) with plural display labels ("Loan Requests").
- **The test:** Would a Certified Master Architect (CMA) look at this schema and say it creates unnecessary maintenance or upgrade collisions?

### 4. Respect the Execution Engine
**Sync for transactional validation. Async for heavy lift. Never `current.update()` in Before BR.**
- `Before` Business Rules = synchronous validation & defaults (only place `setAbortAction` works). Never call `current.update()` inside a Before rule.
- `Flow Designer` / `Async BR` = multi-step approvals, notifications, and cross-system orchestration.
- Never make synchronous outbound REST calls or long queries in user session transactions.
- Never make synchronous client-side `GlideRecord` calls; always use asynchronous `GlideAjax`.
- **The test:** Will this execution design freeze the end-user's browser or exhaust database worker threads under peak concurrency?

### 5. Surgical Delivery & Verified Tooling
**SDK-native, Git-backed, verifiable with ATF.**
- Use verified `@servicenow/sdk` CLI commands: `now-sdk build`, `now-sdk install` (*install*, not *deploy*). Verify every CLI command against `now-sdk --help` or the official reference before emitting it — the CLI evolves between releases.
- Delineate Fluent declarative metadata (`.now.ts`) from implementation logic (`.js`/`.ts`).
- Local filesystem project layouts are SDK build trees, not `sys_metadata` runtime schemas.
- **The test:** Can another engineer clone the Git repository, execute `now-sdk install`, and pass automated ATF suites with zero manual clicks?

---

## When to Use Surgical Mode vs Full Architecture

- **New Application or Cross-Layer Feature:** Execute the full 6-step reasoning loop and run all 6 consistency checks.
- **Surgical Component Change (1 BR, 1 ACL fix, 1 Script Include method):** Skip the full loop. Load only the targeted reference file (e.g., `02-logic-placement.md` or `05-security-design.md`) and run only the relevant check (e.g., Check 3 for BRs, Check 2 for ACLs). Touch only what you must.

---

## The Reasoning Loop

Work through these steps out loud before producing any code or configuration:

```
[1. Clarify & Scope] ──► [2. OOTB Assessment] ──► [3. Model Schema]
                                                           │
                                                           ▼
[6. Verify & Format] ◄── [5. Security & Delivery] ◄── [4. Place Logic & UI]
```

1. **Clarify & Scope:** Extract user personas, transactional volume, integrations, and strict out-of-scope boundaries.
2. **OOTB Check:** Load `references/00-ootb-first.md`. Identify existing platform tables, plugins, or catalog workflows that fulfill the domain.
3. **Data Model Design:** Load `references/01-data-model.md`. Decide extend `task` vs standalone. Enforce singular name / plural label. Define indexed reference keys.
4. **Logic Placement & UI:** Load `references/02-logic-placement.md` and `references/03-ui-layer.md`. Map sync vs async execution. Assign fulfillers to Configurable Workspace and end-users to Employee Center.
5. **Security & Integration:** Load `references/04-integration-design.md` and `references/05-security-design.md`. Create explicit CRUD ACLs. Define scoped roles, RCA, and REST idempotency.
6. **Verify & Format:** Load `references/07-architecture-doc-template.md` and `references/08-consistency-checks.md`. Run all 6 automated consistency checks. Generate the complete architecture document including the Mermaid diagram and phased build order.

---

## Topic Reference Routing Table

Load these modular references on demand as each concern becomes active:

| Concern | When Active | Reference File |
|---------|-------------|----------------|
| **OOTB Evaluation** | Evaluating whether to build custom vs use existing tables (`task`, `cmdb_ci`, etc.) | [`references/00-ootb-first.md`](references/00-ootb-first.md) |
| **Data Model** | Defining tables, fields, relationships (Ref vs M2M vs glide_list), naming standards | [`references/01-data-model.md`](references/01-data-model.md) |
| **Logic Placement** | Choosing BR (Before/After/Async) vs Flow Designer vs Script Include vs Scheduled Job | [`references/02-logic-placement.md`](references/02-logic-placement.md) |
| **UI & Experience** | Choosing Configurable Workspace vs UI Builder vs Employee Center vs Platform UI | [`references/03-ui-layer.md`](references/03-ui-layer.md) |
| **Integrations & ETL** | Choosing Scripted REST vs Table API vs IntegrationHub vs Import Sets / RTE vs MID | [`references/04-integration-design.md`](references/04-integration-design.md) |
| **Security & ACLs** | Writing explicit CRUD ACLs, evaluation order, scoping roles, RCA, guest hardening | [`references/05-security-design.md`](references/05-security-design.md) |
| **Notifications & Events** | Email/SMS/push/chat notifications, event queue design, inbound email actions, notification matrix | [`references/11-notifications.md`](references/11-notifications.md) |
| **Approvals & Routing** | Approval engine vs flow approvals, delegation, assignment rules, skills-based routing, on-call | [`references/15-approvals-assignment.md`](references/15-approvals-assignment.md) |
| **SLA & Commitments** | SLA/OLA/UC definitions, pause/stop conditions, timezone strategy, breach actions | [`references/10-requirements.md`](references/10-requirements.md) |
| **CMDB & CSDM** | IRE-based CI loading, CSDM domains, data precedence, CMDB health gates | [`references/14-cmdb.md`](references/14-cmdb.md) |
| **AI & Agents** | AI Agents, Now Assist/Otto, Virtual Agent, Predictive Intelligence, AI Search, entitlement checks | [`references/13-ai-automation.md`](references/13-ai-automation.md) |
| **Knowledge** | Knowledge bases, article lifecycle, KCS, search surfaces | [`references/12-knowledge.md`](references/12-knowledge.md) |
| **Reporting & Analytics** | Reports, database views, Performance Analytics indicators, dashboards | [`references/16-reporting-analytics.md`](references/16-reporting-analytics.md) |
| **Delivery & SDK** | ServiceNow SDK structure, verified CLI (`now-sdk install`), ATF suites, Git vs Update Sets | [`references/06-delivery-design.md`](references/06-delivery-design.md) |
| **Document Template** | Structuring the final architecture specification with Mermaid diagram and build order | [`references/07-architecture-doc-template.md`](references/07-architecture-doc-template.md) |
| **Consistency Checks** | Running the 6 pre-delivery checks (orphans, ACL completeness, recursion, dependencies) | [`references/08-consistency-checks.md`](references/08-consistency-checks.md) |
| **Anti-Patterns** | Reviewing known gotchas (`getValue` pointer trap, `current.update()` in Before BR, GlideAjax) | [`references/09-anti-patterns.md`](references/09-anti-patterns.md) |
