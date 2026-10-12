# Delivery, Packaging & ServiceNow SDK Architecture (`06-delivery-design.md`)

> Guidance for scoped application packaging, local ServiceNow SDK development, CLI commands, Git source control, and Automated Test Framework (ATF) strategy.

---

## 1. Suggested ServiceNow SDK Local Project Layout

The ServiceNow SDK (`@servicenow/sdk`) enables developers to manage applications as code locally using TypeScript and Fluent metadata definitions.

```
my-app/
├── src/
│   ├── fluent/                 # .now.ts Fluent metadata definitions
│   │   ├── tables/             # Table & dictionary definitions
│   │   ├── business_rules/     # Business Rule metadata definitions
│   │   ├── acls/               # Access Control & role definitions
│   │   └── rest_apis/          # Scripted REST API metadata definitions
│   │
│   └── server/                 # Implementation logic (TypeScript / JavaScript)
│       ├── script_includes/    # Modular business logic & GlideAjax classes
│       └── utils/              # Helper functions & data transformations
│
├── now.config.json             # App scope, sys_id, and active instance auth profile
├── package.json                # Project dependencies, scripts (@servicenow/sdk)
└── dist/                       # Output build directory containing compiled metadata XML
```

> **Note:** This tree represents the local filesystem project structure managed via the ServiceNow SDK and Git. Inside the ServiceNow instance database, all application components are stored as records in `sys_metadata` and table-specific metadata tables.
> 
> **Canonical Implementation Reference:** For official, runnable Fluent code samples across tables, ACLs, business rules, and script includes, refer to [ServiceNow/sdk-examples](https://github.com/ServiceNow/sdk-examples).

---

## 2. Verified ServiceNow SDK CLI Commands

**Durable rule: the CLI evolves between releases. Before emitting any command, verify it against `now-sdk --help`, `now-sdk <command> --help`, or the official CLI reference (servicenow.github.io/sdk/cli). Never write a command from memory.**

Verified `@servicenow/sdk` commands (exactly one of `auth --add|--delete|--use|--list|--print` is required; credentials are stored in the OS keychain):

| Command | Action |
|---------|--------|
| `now-sdk init` | Creates a new app project, applies a template, or converts a legacy app (`--from <sys_id-or-path>`). Run `npm install` afterward. |
| `now-sdk auth --add <instance_url> --type basic\|oauth --alias <name>` | Stores credentials for an instance under an alias. |
| `now-sdk auth --list` / `--use <alias>` | Lists stored credentials / sets the default alias. |
| `now-sdk build` | Compiles `.now.ts` Fluent definitions and TypeScript files into an installable package. Always build before installing. |
| `now-sdk install` | Installs or updates the application on the connected instance. There is no `deploy` command. |
| `now-sdk download <directory>` | Pulls application metadata XML from an instance (JS modules are NOT included). |
| `now-sdk transform` | Converts XML records (local files or instance) into Fluent `.now.ts` source. |
| `now-sdk dependencies` | Fetches TypeScript type definitions for platform APIs (needed for Fluent authoring). |
| `now-sdk clean` / `now-sdk pack` | Removes build output / zips the built app into an installable artifact. |
| `now-sdk cicd install\|publish\|rollback` | Application Repository operations via `sn_cicd`. |
| `now-sdk cicd test run\|watch\|result` | Runs ATF tests against the instance (there is no top-level `test` command). |
| `now-sdk query <table> -q '<encoded query>' -o json` | Queries instance records via the Table API. |
| `now-sdk explain <topic>` | (SDK ≥ 4.6.0) Outputs versioned Fluent schema definitions, properties, and usage rules directly into CLI/agent context. |

---

## 3. Component Authoring & Skill Mapping

To avoid contradictions between Fluent metadata and code authoring, the architecture cleanly delineates responsibilities:

| Artifact / Component | Authoring Environment | Primary Tool |
|----------------------|-----------------------|--------------|
| **Tables, Fields & Choices** | Declarative TypeScript (`.now.ts`) | Fluent (ServiceNow SDK) |
| **ACL Rules & Scoped Roles** | Declarative TypeScript (`.now.ts`) | Fluent (ServiceNow SDK) |
| **Business Rule Triggers** | Declarative TypeScript (`.now.ts`) | Fluent (ServiceNow SDK) |
| **Script Includes (Business Logic)** | TypeScript / JavaScript class modules | Code (with `now-sdk dependencies` type defs) |
| **Client Scripts & UI Policies** | Declarative metadata + JS modules | Fluent (ServiceNow SDK) |
| **Scripted REST API Endpoints** | Declarative routes (`.now.ts`) + script handler | Fluent (ServiceNow SDK) |
| **Flows & Subflows** | Flow Designer visual builder | Platform Flow Designer |
| **Configurable Workspaces & Pages** | UI Builder visual builder | Platform UI Builder |

*Note: safety-critical Fluent convention — never delete a `Table()`, `BusinessRule()`, or `Record()` definition from a `.now.ts` file without confirming with the user first; deletion may need to propagate as an upgrade-time delete via `keys.ts`, which the code alone cannot reveal.*

---

## 4. Deployment Pipeline & Packaging Strategies

```
+─────────────────────────────────────────────────────────────────────────────+
|                         Enterprise CI/CD Promotion                          |
+─────────────────────────────────────────────────────────────────────────────+
|                                                                             |
|  [ Dev Instance / Local IDE ]  ──►  Git Source Control                      |
|                                     (Feature branches, PR reviews)          |
|                                           │                                 |
|                                           ▼                                 |
|  [ Test Instance (Sub-Prod) ]  ──►  App Repository                          |
|                                     (Run ATF Suites + Instance Scan)        |
|                                           │                                 |
|                                           ▼                                 |
|  [ Prod Instance ]             ──►  Install / Upgrade via App Repo          |
|                                                                             |
+─────────────────────────────────────────────────────────────────────────────+
```

### Packaging Comparison

| Packaging Mechanism | Recommended Usage | Version Control | Conflict Resolution |
|---------------------|-------------------|:---------------:|---------------------|
| **ServiceNow SDK + Git** | Modern scoped app development | ✅ Full Git history | Branching, PR reviews |
| **Application Repository** | Promoting scoped app versions across instances (Test ➔ Prod) | ✅ Semantic versioning | Platform checksum validation |
| **Update Sets** | Global scope configurations, emergency hotfixes | ❌ XML snapshots | Manual preview & collision resolution |

---

## 5. Automated Test Framework (ATF) Architecture

Every scoped application architecture must incorporate an automated test plan:

### Test Hierarchy & Execution Priority
1. **Critical Path Server Tests (High Priority, Fast):**
   - Server-side script steps testing Script Includes directly.
   - Record validation steps verifying that Before Business Rules calculate fields correctly and enforce `setAbortAction`.
2. **Security & Role Impersonation Tests:**
   - Impersonate Persona User ➔ Attempt to read, create, update, and delete records.
   - Verify that unauthorized roles receive access denied.
3. **Integration Point Validation:**
   - Scripted REST API endpoint tests submitting sample payloads and checking response codes and JSON bodies.
4. **UI & End-to-End Tests (Low Priority, Fragile):**
   - Workspace and Service Portal form tests (keep minimal; prioritize server-side validation).

### ATF Architecture Rules
- All ATF tests must reside **inside the application scope**.
- Tests must be **self-contained**: create dummy test records with unique identifiers and delete test records in teardown steps.
- Bundle tests into an **Application Deployment Test Suite** to run automatically in CI/CD before instance promotion.

---

## 6. Upgrade Management & Environment Strategy

Delivery is not done at first deploy — the app must survive platform upgrades:

1. **Skipped record review:** any customization that collides with a platform upgrade lands as a skipped record. Design to minimize skips (prefer configuration over code — see `00-ootb-first.md`) and include a skipped-record review step in every post-upgrade runbook.
2. **Customization registry:** every deviation from OOTB behavior is documented in the architecture document with its upgrade-impact note — an undocumented customization is an upgrade time bomb.
3. **Environment strategy:** document the instance landscape (Dev / Test / Prod, cloning cadence, and refresh policy). Clones wipe test data and local-only settings; the delivery plan must state what each environment expects.
4. **Platform release pinning:** the SDK/CLI evolves between releases (see the verify-CLI rule in §2) and metadata compatibility is release-bound — record the target platform release in the architecture document and re-verify tooling on release upgrades.
5. **Post-upgrade verification:** ATF suites (§5) double as upgrade regression tests — run the full suite after every platform upgrade, not just before deploys.

---

## Authoritative Sources
- [ServiceNow SDK CLI Reference](https://servicenow.github.io/sdk/cli)
- [Official ServiceNow SDK Samples & Fluent Reference Implementations](https://github.com/ServiceNow/sdk-examples)
- [ServiceNow Documentation: Automated Test Framework (ATF)](https://docs.servicenow.com/bundle/xanadu-application-development/page/administer/auto-test-framework/concept/c_ATF.html)
- [ServiceNow Documentation: Instance Scan](https://docs.servicenow.com/bundle/xanadu-application-development/page/administer/instance-scan/concept/instance-scan.html)
