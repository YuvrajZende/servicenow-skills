# Changelog

All notable changes to `sn-architect` are documented here. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versioning is SemVer.

## [1.1.0] — 2026-10-12

### Added
- `17-fluent-cookbook.md` — bridge from approved architecture to verified ServiceNow SDK (Fluent) code generation, using official patterns from [ServiceNow/sdk-examples](https://github.com/ServiceNow/sdk-examples); includes canonical Fluent patterns for Tables (`task` extension, typed columns), explicit CRUD ACLs, and Business Rules with server-side logic separation (attributed under MIT), plus Fluent anti-patterns (`keys.ts` delete convention, `generated/` output, `localOrExisting` semantics).
- Added `now-sdk explain` to CLI reference table in `06-delivery-design.md` for querying live Fluent schema definitions (SDK ≥ 4.6.0).
- SDK UI Pages with modern JS frameworks (React / Vue / Svelte / SolidJS) added to `03-ui-layer.md` as an explicit, last-resort UI tier with official sample references.
- Rule 4.6 added to Check 4 in `08-consistency-checks.md` enforcing platform standard dot notation for scoped roles (`x_<vendor>_<app>.<role>`).
- Conditional architecture template sections in `07-architecture-doc-template.md`: §6C Knowledge Base Map, §7B Approval Matrix, §7C Routing Map, §7D Reporting & Analytics Map.
- Interactive PASS/FAIL verification table with required evidence slots added to Section 10 of `07-architecture-doc-template.md`.
- Authoritative Sources sections added across all 18 reference files, citing public ServiceNow product documentation and developer guides.
- Repo governance files: `CHANGELOG.md`, `CONTRIBUTING.md` (unverified claim marker, public sources standard, exam IP policy, worked retrieval test), and `NOTICE` (MIT attribution for `ServiceNow/sdk-examples`).
- README enhancements: runtime installation compatibility table, scoped provenance section, test cases, and ecosystem links including the official vendor SDK agent skill.

### Changed
- Renamed `10-requirements.md` to `10-sla.md` to accurately reflect its focus on SLA/OLA/UC design and volume math.
- Outer markdown fence in `07-architecture-doc-template.md` expanded to four backticks to prevent premature termination around inner Mermaid diagram blocks.
- Role naming standardized across `05-security-design.md`, `07-architecture-doc-template.md`, and `09-anti-patterns.md` to use the official dot separator (`x_acme_loan.admin`, `x_acme_loan.fulfiller`).
- Clarified `After` Business Rule execution timing in `02-logic-placement.md`: runs after the database write but synchronously within the same database transaction before commit.
- Corrected `17-fluent-cookbook.md` Anti-Pattern 5: clarified that `localOrExisting` defines ACL security attribute localization, while table extension uses `augments`.
- Corrected CSDM citation in `00-ootb-first.md` to point to official product documentation instead of a retired community URL.
- Softened `sys_user` extensibility rule in `01-data-model.md` and `SKILL.md` to explicitly target custom scoped applications, acknowledging native platform extensions like CSM `customer_contact`.
- Added empirical Scripts - Background reproduction output in `09-anti-patterns.md` demonstrating GlideElement reference retention across loop iterations.
- Added explicit `> ⚠️ Unverified:` marker in `13-ai-automation.md` for the 300+ records classification guideline.
- Removed speculative unverified claims: unverified `deny_unless` evaluation order, `u_imp_*` prefix mandate, and Otto rebrand references.

## [1.0.0] — 2026-10-11

### Added
- `sn-architect` v1: thin-router `SKILL.md` over 17 modular reference files (`00`–`16`); 6 consistency checks including Rule 6.4 (IRE Enforcement) and Rule 6.5 (AI Entitlement Gate); conditional architecture-template appendices (SLA 4A, Notifications 6A, AI 6B, CMDB/IRE 7A).
- Initial factual verification against official docs: SDK CLI (`install` not `deploy`; `download`/`transform` not `fetch`; `cicd test` not `test`), ACL evaluation order (incl. `parent_table.*`), scoped IRE API namespace (`sn_cmdb.IdentificationEngine`).
- Empirical RED/GREEN agent comparison on a temporary building access pass scenario ($n=1$).