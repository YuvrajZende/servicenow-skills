<div align="center">

<img src="assets/banner.svg" alt="sn-architect — architecture-first ServiceNow skills for AI coding agents" width="100%">

[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Format](https://img.shields.io/badge/format-agentskills.io-blue)](https://agentskills.io)
[![References](https://img.shields.io/badge/references-18-blueviolet)](#how-it-works)
[![Consistency checks](https://img.shields.io/badge/consistency_checks-6-orange)](sn-architect/references/08-consistency-checks.md)
[![Tested](https://img.shields.io/badge/behavior_tested-RED%2FGREEN%2C_n%3D1-yellow)](#test-evidence)

</div>

# sn-architect — architecture-first ServiceNow skills for AI coding agents

`sn-architect` is a skill that turns your AI coding agent (Claude Code, Cursor, Antigravity, and other agent runtimes) into a ServiceNow **enterprise architect**. Instead of generating tables, business rules, and scripts on demand, the agent first designs the solution — evaluates what the platform already provides, models the schema, places logic, wires security, and validates the design against 6 mandatory consistency checks — **before** any code or configuration is generated.

```mermaid
flowchart LR
    A["Business requirement"] --> B{"SKILL.md<br/>thin router"}
    B -- "full mode" --> C["6-step reasoning loop"]
    B -- "surgical mode" --> D["targeted reference only"]
    C --> E["On-demand references<br/>00–17"]
    D --> E
    E --> F["6 consistency checks<br/>incl. IRE + AI entitlement gates"]
    F --> G["Architecture spec<br/>+ conditional appendices"]
    G --> H["Now: code generation"]
```

---

## The problem it solves

If you have asked an AI agent to "build a ServiceNow app for loan requests," you have likely received:

- a custom table that duplicates `task` (losing SLAs, approvals, and unified fulfiller views),
- no ACLs at all — or worse, an **empty ACL** open to every authenticated user,
- `current.update()` inside a Before Business Rule (recursive loop),
- synchronous outbound REST calls freezing every user transaction,
- a custom approval table instead of the platform's approval engine,
- CLI commands that don't exist in the current ServiceNow SDK.

These are not model-knowledge problems — they are **workflow** problems. Agents generate code first and think about platform consequences never. This skill restructures that workflow.

## How it works

The skill is a **thin router, not a documentation dump**. `SKILL.md` (~100 lines) carries hard rules, a surgical-vs-full-mode decision, a 6-step reasoning loop, and a routing table. Eighteen modular reference files load **on demand**, only when a concern becomes active — and after the architecture is approved, the agent generates ServiceNow SDK (Fluent) code from **verified official patterns**, not from memory:

<details>
<summary><strong>All 18 reference files</strong> (click to expand)</summary>

| Reference | Concern |
|-----------|---------|
| `00-ootb-first.md` | Step 0 — never rebuild what the platform ships; OOTB evaluation protocol |
| `01-data-model.md` | Extend `task` vs standalone, naming standards, indexing, data lifecycle & scale |
| `02-logic-placement.md` | Before/After/Async BRs, Flow Designer, Script Includes, UI Actions, sync vs async |
| `03-ui-layer.md` | Configurable Workspace, UI Builder, Employee Center/Portal, Platform UI, mobile, Playbooks |
| `04-integration-design.md` | REST, IntegrationHub, Import Sets/RTE, MID Server, e-bonding, webhooks, identity |
| `05-security-design.md` | Explicit CRUD ACLs, ACL evaluation order, empty-ACL threat, instance hardening |
| `06-delivery-design.md` | ServiceNow SDK (Fluent) project layout, verified CLI, ATF, Git, upgrade management |
| `07-architecture-doc-template.md` | The specification template with conditional appendices (SLA, notifications, CMDB, AI) |
| `08-consistency-checks.md` | The 6-check validation gate (orphans, ACL completeness, transaction integrity, IRE enforcement, AI entitlement, ...) |
| `09-anti-patterns.md` | `getValue()` GlideElement trap, `current.update()` recursion, client GlideRecord, timezone bugs, ... |
| `10-sla.md` | SLA / OLA / UC design |
| `11-notifications.md` | Notification records, event pipeline, inbound email |
| `12-knowledge.md` | Knowledge bases, article lifecycle, KCS |
| `13-ai-automation.md` | AI Agents, Now Assist, Virtual Agent, Predictive Intelligence, AI Search |
| `14-cmdb.md` | IRE enforcement, CSDM, data precedence, CMDB health |
| `15-approvals-assignment.md` | Approval engine, routing, assignment, on-call |
| `16-reporting-analytics.md` | Reports, database views, Performance Analytics |
| `17-fluent-cookbook.md` | `.now.ts` code generation from verified official patterns (tables, ACLs, BRs, REST, ATF, UI pages) |

</details>

Every reference follows the same discipline: **decision matrices instead of essays**, hard rules with falsification tests ("The test: ..."), and anti-pattern tables. Nothing is written from memory — factual claims (CLI commands, API signatures, table names) were verified against official ServiceNow documentation, and the skill itself carries the durable rule: *verify every CLI command against `now-sdk --help` before emitting it.*

## What you get

For a full application requirement, the agent produces an **architecture specification** before any code:

- OOTB evaluation with justification for every custom component
- Data model with relationships, naming, and indexing decisions
- Mermaid architecture diagram and sync/async execution mapping
- Security matrix: scoped roles, explicit CRUD + wildcard ACLs
- Conditional appendices exactly when relevant: SLA matrix, notification matrix, knowledge base map, CMDB ingestion & IRE precedence table, approval matrix, routing map, reporting & analytics map, AI agent tool & entitlement map
- Chronological phased build order and a risk register

For small tasks ("fix this ACL", "add one business rule"), the skill runs in **surgical mode** — it loads only the targeted reference and applies only the relevant check, instead of forcing the full ceremony.

---

## Installation

Install `sn-architect` by cloning or linking the `sn-architect/` directory into your coding agent's skill path.

### Supported Runtime Paths

| Runtime | Skill Installation Path | Tested Status | Reference Documentation |
|---|---|:---:|---|
| **Claude Code** (user global) | `~/.claude/skills/sn-architect` | Tested | [Claude Code Reference](https://docs.anthropic.com) |
| **Claude Code** (project local) | `.claude/skills/sn-architect` | Tested | [Claude Code Reference](https://docs.anthropic.com) |
| **Antigravity IDE** | `.agents/skills/sn-architect` (workspace) or `~/.gemini/config/skills/` | Tested | Antigravity Skills System |
| **Cursor** | `.cursor/skills/sn-architect` | Manual copy | [Cursor Documentation](https://docs.cursor.com) |
| **GitHub Copilot** | `.github/skills/sn-architect` or `~/.copilot/skills/sn-architect` | Per SDK ref | [ServiceNow SDK Skills](https://servicenow.github.io/sdk) |
| **ServiceNow Build Agent (Now IDE)** | *Not directly compatible* | Incompatible | Uses internal Studio package format; requires manual conversion. |

### Manual Installation (Claude Code)

```bash
git clone https://github.com/YuvrajZende/servicenow-skills.git
mkdir -p ~/.claude/skills
cp -r servicenow-skills/sn-architect ~/.claude/skills/sn-architect
```

Verify with `/skills` inside Claude Code, or by asking: *"Design a ServiceNow app for tracking building access passes"* — the agent should start with the OOTB evaluation, not with code.

### Installation via skills.sh

```bash
npx skills add YuvrajZende/servicenow-skills
```

> **Note on relative paths:** `SKILL.md` routes on demand to `references/*.md` located relatively beside it. When copying or moving `sn-architect`, keep the `references/` directory adjacent to `SKILL.md`.

---

## Usage

**Full architecture mode** — any request like:

- "Design a ServiceNow app to manage building access requests with manager approval"
- "We need to ingest asset data from our ERP — architect it"
- "Create a table model for vendor risk assessments with SLAs"

**Surgical mode** — single-component tasks:

- "Add an ACL so only fulfillers can close loan requests"
- "Write a Before BR that defaults the priority"
- "Why does my client script freeze the form?"

**Review mode** — point it at an existing design and ask it to run the consistency checks:

- "Review this architecture against your 6 checks"

## Provenance & accuracy

Status: **Beta (v1.1.0)**. Written against ServiceNow Xanadu product documentation and `@servicenow/sdk` (≥ 4.6.0). Re-check against your instance release family.

- **Spot-checked against official docs / SDK:**
  - `@servicenow/sdk` CLI commands (`init`, `auth`, `build`, `install`, `download`, `transform`, `dependencies`, `cicd`, `explain`) verified against `servicenow.github.io/sdk/cli`.
  - Scoped role naming standard (`<scope>.<role>`) verified against official `ServiceNow/sdk-examples` (`acl-sample`).
  - ACL evaluation hierarchy (most-specific first: table.field ➔ parent_table.field ➔ *.field ➔ table.* ➔ parent_table.* ➔ *.*) and empty-ACL security verified against ServiceNow security documentation.
  - Identification and Reconciliation Engine (IRE) API signatures (`sn_cmdb.IdentificationEngine`) verified against ServiceNow CMDB documentation.
  - GlideElement reference retention behavior in `GlideRecord` loops verified empirically on ServiceNow instance (Scripts - Background).
  - Table, Column, and BusinessRule Fluent builder signatures verified against `ServiceNow/sdk-examples`.
- **Not yet verified on live instance:**
  - Full end-to-end deployment of generated Fluent `.now.ts` projects through an automated enterprise CI/CD pipeline (`now-sdk cicd install`).
  - Automated execution of generated ATF test builders against an active test runner.

Found something wrong? [Open an issue](https://github.com/YuvrajZende/servicenow-skills/issues) with the official documentation link.

## Test evidence

One full RED/GREEN comparison run (same model, same requirement, ~150-line output cap each): *"Build a ServiceNow app for temporary building access passes: employees request a pass, their manager approves, security issues it, and the pass expires."*

| Behavior | Without skill | With skill |
|----------|:---:|:---:|
| OOTB-first evaluation (task vs standalone, Store apps) | ✗ skipped | ✓ explicit, justified |
| Naming/indexing standards (plural labels, indexed refs) | ✗ partial | ✓ full |
| ACL package (CRUD + `table.*`, zero empty ACLs, RCA) | ~ partial | ✓ full |
| Approval timeout/escalation path | ✗ missing | ✓ timeout → escalate → auto-reject |
| Consistency checks run & reported | ✗ none | ✓ all 6, per-rule |
| Verified CLI (`now-sdk install`) | ✗ generic packaging | ✓ verified commands, ATF, Instance Scan |
| Invented APIs in code sample | ✗ 1 (`setWorkNotes`) | ✓ none |

**Honest caveats:** this is **n=1** — one scenario, one model, one run per arm. It is evidence the skill changes behavior in the targeted directions, not a multi-arm benchmark. The test also surfaced real platform nuances (such as before-query business rules for row filtering, and storing primitive values instead of GlideElement object references), which were directly back-ported into `05-security-design.md` and `09-anti-patterns.md`.

### Reproducible Test Cases

To validate the skill across other architectural patterns, run these test cases with and without `sn-architect`:

1. **CMDB / Ingestion:** *"We receive an hourly JSON payload from AWS containing EC2 instances; ingest and update our CMDB server inventory."*
   - *Target behavior:* Stages via `sys_import_set_row`, routes CI writes through `sn_cmdb.IdentificationEngine` (Rule 6.4), verifies `com.snc.cmdb.scoped`, rejects direct `gr.insert()` to `cmdb_ci_server`.
2. **Surgical Review Mode:** *"Review this Before Business Rule: it calls an external payment gateway via RESTMessageV2 and executes current.update()."*
   - *Target behavior:* Operates in surgical mode (does not emit a full 10-section architecture document), isolates Rule 3.1 (`current.update()` in Before BR) and Rule 3.3 (synchronous outbound HTTP) violations, and provides an asynchronous refactoring.
3. **AI Agent Governance:** *"Design a Now Assist GenAI agent in Customer Service to auto-triage inbound cases and assign priorities."*
   - *Target behavior:* Enforces Rule 6.5 entitlement check gate, binds minimal scoped tools instead of raw script execution, and requires a human-in-the-loop review point for consequential state mutations.

More scenarios welcome — [open an issue](https://github.com/YuvrajZende/servicenow-skills/issues) with your prompt and architecture run if the skill routes wrong or misses a check.

## Ecosystem & complementary tools

`sn-architect` focuses specifically on **system architecture and design governance** before code generation. For downstream implementation and task-level execution, it pairs naturally with:

- **[ServiceNow SDK Agent Skill (`@servicenow/sdk`)](https://servicenow.github.io/sdk):** The official ServiceNow vendor agent skill for Claude Code / Cursor, providing live `explain` commands, SDK project scaffolding, and CLI execution.
- **[ServiceNow SDK Samples (`ServiceNow/sdk-examples`)](https://github.com/ServiceNow/sdk-examples):** The official ServiceNow repository for runnable Fluent (`.now.ts`) declarative metadata examples across tables, ACLs, and business rules.
- **[Happy Platform Skills (`Happy-Technologies-LLC/happy-platform-skills`)](https://github.com/Happy-Technologies-LLC/happy-platform-skills):** A library of 180+ task-level procedural skills covering granular admin and operational tasks across ITSM, CSM, and HRSD.

| Layer | Tool | Role |
|---|---|---|
| **System Architecture & Governance** | `sn-architect` (this repo) | OOTB evaluation, data model, sync/async logic, ACL package, IRE & AI gates, build order |
| **Vendor SDK Agent Skill** | [`@servicenow/sdk`](https://servicenow.github.io/sdk) | Official SDK CLI agent commands (`explain`, build, install) |
| **Reference Fluent Code** | [`ServiceNow/sdk-examples`](https://github.com/ServiceNow/sdk-examples) | Official TypeScript Fluent patterns for SDK projects |
| **Task & Operational Execution** | [`happy-platform-skills`](https://github.com/Happy-Technologies-LLC/happy-platform-skills) | Granular procedural recipes for individual platform workflows |

## Contributing

The highest-value contributions are: (1) failing test scenarios — requirements where agents without the skill produce broken designs, (2) corrections with sources (docs links required — this project's standard is *no invented facts*), and (3) new reference files following the existing format: decision matrix → standards → anti-patterns → required output.

## License & disclaimer

MIT. This is an unofficial community project and is **not affiliated with, endorsed by, or sponsored by ServiceNow**. "ServiceNow" is a trademark of ServiceNow, Inc.; used here descriptively to indicate compatibility.

- **Contains no ServiceNow exam or training course content:** `sn-architect` is strictly an architecture and engineering skill based on public documentation and APIs. It contains no certification exam questions, answers, or training courseware.
- Always validate generated architectures against your instance's release, licensing, and security posture — the skill is designed to tell the agent the same thing.