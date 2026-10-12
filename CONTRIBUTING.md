# Contributing to sn-architect

Thanks for helping make AI agents better ServiceNow enterprise architects. This project maintains a strict standard of verifiability:

---

## 1. No Invented Facts

Every factual claim — CLI commands, API signatures, table names, ACL behavior — must trace to verifiable evidence:
1. **Public Official Documentation:** Prefer public documentation links ([docs.servicenow.com](https://docs.servicenow.com), [developer.servicenow.com](https://developer.servicenow.com), [servicenow.github.io/sdk](https://servicenow.github.io/sdk), [ServiceNow/sdk-examples](https://github.com/ServiceNow/sdk-examples)). Avoid citing Support KB articles that sit behind authentication portals, as community reviewers cannot independently verify them.
2. **Empirical PDI Execution:** For runtime JavaScript or Glide API behaviors (e.g., GlideRecord/GlideElement handling, execution order), empirical test output executed on a Personal Developer Instance (PDI) is accepted. Contributions must document the instance release family (e.g., Xanadu patch 2) and paste the exact script and execution log from Scripts - Background.
3. **Explicit Unverified Markers:** If an architectural rule-of-thumb is widely used in practice but lacks an official documented threshold, mark it explicitly:
   ```markdown
   > ⚠️ Unverified: <description of empirical observation or unverified rule-of-thumb>
   ```

---

## 2. Intellectual Property & Exam Content Policy

**Strict Rule:** Do NOT contribute content derived from ServiceNow certification exams (CSA, CAD, CIS, CMA), exam dumps, official instructor-led training slides, or proprietary courseware. `sn-architect` is an independent, architecture-first design skill built from public APIs and platform engineering best practices.

---

## 3. Highest-Value Contributions

1. **Failing Test Scenarios:** Real requirements where coding agents *without* the skill produce broken designs (e.g., extending `sys_user` in custom apps, omitting table wildcard ACLs, synchronous HTTP calls inside Before rules, direct `cmdb` table writes). Include the prompt and baseline agent failure.
2. **Factual Corrections with Sources:** Corrections to outdated API signatures or platform claims, accompanied by public documentation links.
3. **Changelog Updates:** Every contribution that modifies architectural rules or reference files must update [`CHANGELOG.md`](CHANGELOG.md) under `[Unreleased]` or the current version entry.
4. **New Reference Files:** Follow the established house format:
   - Header with filename and one-line blockquote purpose
   - Decision matrix (concise tables, not long essays)
   - Hard rules with falsification tests (`**The test:** ...`)
   - Anti-patterns section with comparison code blocks
   - "Output in the Architecture Document" section defining section mapping
   - "Authoritative Sources" section with public links

---

## 4. Testing Your Changes (Retrieval Testing)

Edits must be retrieval-tested: with only `SKILL.md` in context, a fresh AI coding agent should route a user request to your reference file and answer without hallucination.

**Worked Example:**
- **User Prompt:** *"We receive an hourly JSON feed of AWS EC2 instances; how should we ingest them into the CMDB without duplicate CIs?"*
- **Expected Routing:** The agent routes via `SKILL.md` to [`references/14-cmdb.md`](sn-architect/references/14-cmdb.md) and [`references/04-integration-design.md`](sn-architect/references/04-integration-design.md).
- **Expected Output:** Enforces Rule 6.4 (IRE routing via `sn_cmdb.IdentificationEngine`), verifies `com.snc.cmdb.scoped`, stages payloads in a table extending `sys_import_set_row`, and rejects direct `gr.insert()` to `cmdb_ci_server`.

---

## 5. Submitting Issues & Pull Requests

Open issues or pull requests on GitHub at [YuvrajZende/SERVICENOW-SKILLS/issues](https://github.com/YuvrajZende/SERVICENOW-SKILLS/issues). Please describe the platform release tested and link relevant documentation.

## License

By contributing, you agree that your contributions are licensed under the MIT License of this repository.