# Changelog

All notable changes to `sn-architect` are documented here. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versioning is SemVer.

## [1.1.0] — 2026-10-12

### Added
- `17-fluent-cookbook.md` — bridge from approved architecture to verified ServiceNow SDK (Fluent) code generation, using official patterns from [ServiceNow/sdk-examples](https://github.com/ServiceNow/sdk-examples); includes canonical Fluent patterns for Tables, explicit CRUD ACLs, and Business Rules with server-side logic separation (attributed), plus Fluent anti-patterns (`keys.ts` delete convention, `generated/` output, `localOrExisting` discipline).
- SDK UI Pages with modern JS frameworks (React / Vue / Svelte / SolidJS) added to `03-ui-layer.md` as a distinct, last-resort UI tier with official sample references.
- `SKILL.md`: implementation step wired into the reasoning loop (architecture → approval → verified Fluent code generation); routing row for the Fluent cookbook.
- Repo maturity: `CHANGELOG.md`, `CONTRIBUTING.md`, `NOTICE`.
- README: reproducible test cases, ecosystem positioning (sdk-examples, happy-platform-skills), skills.sh install path.

### Changed
- Conditional template appendices extended (§6C Knowledge Base Map, §7D Reporting & Analytics Map — contributed by author).

### Verified against official sources
- Fluent `Table`/`Column`, `Acl`/`Role`, and `BusinessRule` builder shapes from `ServiceNow/sdk-examples`.
- SDK UI Page samples (React/Vue/Svelte/SolidJS) existence and layout confirmed from `ServiceNow/sdk-examples`.

## [1.0.0] — 2026-10-11

### Added
- `sn-architect` v1: thin-router SKILL.md over 17 reference files (00–16); 6 consistency checks including Rule 6.4 (IRE Enforcement) and Rule 6.5 (AI Entitlement Gate); conditional architecture-template appendices (SLA 4A, Notifications 6A, AI 6B, CMDB/IRE 7A).
- Behavior-tested: RED/GREEN subagent comparison on a building-access-pass scenario; GlideElement loophole found, plugged, and re-verified by micro-test; before-query BR pattern back-ported from baseline findings.
- Factual verification pass against official docs: SDK CLI (`install` not `deploy`; `download`/`transform` not `fetch`; `cicd test` not `test`), ACL evaluation order (incl. `parent_table.*`), AI/VA/NLU tables, IRE API.