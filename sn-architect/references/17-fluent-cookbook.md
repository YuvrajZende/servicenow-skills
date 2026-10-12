# Fluent Code Cookbook (`17-fluent-cookbook.md`)

> Bridge from architecture to implementation: when the design is approved, generate ServiceNow SDK (Fluent) metadata from **verified official patterns** — never from memory.

---

## 1. The Source-of-Truth Rule

Fluent syntax evolves with the SDK. Before generating any `.now.ts` definition:

1. **Prefer the official examples:** [ServiceNow/sdk-examples](https://github.com/ServiceNow/sdk-examples) — a pnpm monorepo of runnable samples, one directory per artifact (`<feature>-sample`, e.g., `acl-sample`, `restapi-sample`, `test-atf-sample`; also tables, business rules, client scripts, script includes, UI actions, UI pages, catalog, portal, flows, state models, records, lists, cross-scope modules, table dependencies).
2. **Verify the API surface:** with the ServiceNow SDK plugin available, run `npx @servicenow/sdk explain <topic> --peek --format=raw` (e.g., `explain Acl`, `explain Table`, `explain BusinessRule`) for the versioned, authoritative definition of every builder's fields. Never guess property names.
3. **Type definitions:** run `now-sdk dependencies` in the project to get TypeScript definitions for platform APIs before writing implementation code.

**Hard rule:** a Fluent definition with an invented property name or wrong builder signature is a defect — same severity as a fabricated CLI command (Hard Rule 3 of `SKILL.md`).

---

## 2. Verified Canonical Fluent Patterns (from `ServiceNow/sdk-examples`)

### 2A. Verified Table & Schema Pattern (from `tables` sample)
Attribution: [ServiceNow/sdk-examples — tables](https://github.com/ServiceNow/sdk-examples). Defines custom tables with clean inheritance, singular naming, plural display labels, and indexed foreign key references (`01-data-model.md`):

```typescript
import { Table, StringColumn, ReferenceColumn, ChoiceColumn } from '@servicenow/sdk/core'

export const x_acme_loan_request = Table({
    name: 'x_acme_loan_request',
    label: 'Loan Requests',
    extends: 'task', // Extends task per 01-data-model.md (inherits assignment, SLAs, work notes)
    schema: {
        loan_amount: StringColumn({
            label: 'Loan Amount',
            maxLength: 40,
            mandatory: true,
        }),
        requested_by: ReferenceColumn({
            label: 'Requested For',
            reference: 'sys_user', // Avoid extending sys_user; reference it cleanly
            mandatory: true,
        }),
        status: ChoiceColumn({
            label: 'Loan Status',
            choices: {
                draft: { label: 'Draft', sequence: 0 },
                submitted: { label: 'Submitted', sequence: 1 },
                approved: { label: 'Approved', sequence: 2 },
                rejected: { label: 'Rejected', sequence: 3 },
            },
            default: 'draft',
        }),
    },
})
```

### 2B. Verified Fluent ACL Pattern (from `acl-sample`)
Attribution: [ServiceNow/sdk-examples — acl-sample](https://github.com/ServiceNow/sdk-examples/tree/main/acl-sample). Demonstrates explicit CRUD and REST endpoint access controls (`05-security-design.md`):

```typescript
import { Acl, Role } from '@servicenow/sdk/core'

export const sampleAdmin = Role({
    name: 'x_acme_loan.admin',
})

export const sampleFulfiller = Role({
    name: 'x_acme_loan.fulfiller',
})

Acl({
    $id: Now.ID['create_acl'],
    localOrExisting: 'Existing',
    type: 'record',
    operation: 'create',
    roles: [sampleAdmin, sampleFulfiller],
    table: 'x_acme_loan_request',
})

Acl({
    $id: Now.ID['read_acl'],
    localOrExisting: 'Existing',
    type: 'record',
    operation: 'read',
    table: 'x_acme_loan_request',
    condition: 'active=true',
})

Acl({
    $id: Now.ID['rest_acl'],
    name: 'loan_api',
    type: 'rest_endpoint',
    operation: 'execute',
    roles: [sampleAdmin],
    securityAttribute: 'user_is_authenticated',
})
```

### 2C. Verified Business Rule & Logic Separation Pattern (from `business-rule` sample)
Attribution: [ServiceNow/sdk-examples — business-rule](https://github.com/ServiceNow/sdk-examples). Fluent metadata (`.now.ts`) defines the trigger; server-side JavaScript (`/server`) implements the logic (`02-logic-placement.md`):

```typescript
// src/fluent/business_rules.now.ts
import { BusinessRule } from '@servicenow/sdk/core'
import { validateLoanSubmission } from '../server/validateLoanSubmission.js'

export const validateLoanSubmissionRule = BusinessRule({
    $id: Now.ID['validate_loan_submission'],
    name: 'Validate Loan Submission',
    table: 'x_acme_loan_request',
    when: 'before',
    action: ['insert', 'update'],
    order: 100,
    active: true,
    script: validateLoanSubmission,
})
```

```javascript
// src/server/validateLoanSubmission.js
import { gs } from '@servicenow/glide'

export function validateLoanSubmission(current, previous) {
    // Synchronous validation rule per 02-logic-placement.md & 09-anti-patterns.md
    if (current.loan_amount.nil()) {
        gs.addErrorMessage('Loan amount is mandatory before submission.')
        current.setAbortAction(true) // ABORT is valid only in Before BR; never call current.update()
    }
}
```

**Design notes connecting these patterns to the architecture rules:**
- `Table()` enforces singular technical table name with plural label, extending `task` when workflow features are needed.
- `Acl()` enforces explicit CRUD operation definitions; zero empty ACLs (`05-security-design.md`).
- `BusinessRule()` separates declarative metadata from runnable logic; synchronous validation uses `current.setAbortAction(true)` in Before BR and never calls `current.update()` (`02-logic-placement.md`).
- `condition` takes encoded query strings; `securityAttribute` uses modern ABAC policies.

---

## 3. Artifact → Authoring Path → Official Sample

| Artifact (from the architecture spec) | Fluent builder | Official sample to consult |
|----------------------------------------|----------------|----------------------------|
| Tables & fields | `Table()` | `tables` sample directory |
| ACLs & roles | `Acl()`, `Role()` | `acl-sample` |
| Business rules | `BusinessRule()` | business rule sample directory |
| Script Includes | TS/JS module + `Record()` metadata | script include sample directory |
| Client scripts | `ClientScript()` (metadata) + JS module | client script sample directory |
| UI actions | `UiAction()` | UI action sample directory |
| Scripted REST API | REST API builders | `restapi-sample` |
| ATF tests | ATF test builders | `test-atf-sample` |
| State models | State model builders | state model sample directory |
| Catalog items | Catalog builders | service catalog sample directory |
| UI Pages (modern JS) | UI Page builder + frontend framework | React / Vue / Svelte / SolidJS sample directories |

*Exact directory names evolve with the repo — browse [github.com/ServiceNow/sdk-examples](https://github.com/ServiceNow/sdk-examples) root rather than trusting a memorized slug.*

---

## 4. Fluent Anti-Patterns

1. **Deleting definitions carelessly:** never remove a `Table()`, `BusinessRule()`, or `Record()` from a `.now.ts` file without confirming with the user — deletion may need to propagate as an upgrade-time delete via `keys.ts`, which the code alone cannot reveal.
2. **Hand-writing what the compiler verifies:** run `now-sdk build` immediately after authoring; build errors are the cheapest syntax checker.
3. **Mixing concerns:** Fluent files declare *metadata*; business logic lives in `/server` modules — a `.now.ts` file containing multi-hundred-line scripts is a design smell (see `02-logic-placement.md` Script Include rule).
4. **Editing `generated/` output:** files under `src/fluent/generated/` are compiler-managed; hand edits are lost on the next build.
5. **Misunderstanding `localOrExisting` in ACLs:** in `Acl()`, `localOrExisting: 'Local'` specifies an inline security attribute condition, while `'Existing'` references a shared platform security attribute. To extend existing platform tables without duplicating them, use `augments: '<table_name>'` on `Table()`, never declare a duplicate table with `name: '<existing_table>'` (Check 6 territory).

---

## Authoritative Sources
- [ServiceNow/sdk-examples (official runnable Fluent samples)](https://github.com/ServiceNow/sdk-examples)
- [Now SDK CLI Reference](https://servicenow.github.io/sdk/cli)
- `npx @servicenow/sdk explain` — versioned Fluent API documentation shipped with the SDK (requires SDK ≥ 4.6.0)