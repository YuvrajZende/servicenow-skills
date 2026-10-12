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

## 2. Verified Fluent ACL Pattern (from the official `acl-sample`)

Attribution: [ServiceNow/sdk-examples — acl-sample/src/fluent/index.now.ts](https://github.com/ServiceNow/sdk-examples/tree/main/acl-sample). Reproduced to show the canonical shape:

```typescript
import { Acl, Role } from '@servicenow/sdk/core'

export const sampleAdmin = Role({
    name: 'x_acl_sample.admin',
})

Acl({
    $id: Now.ID['create_acl'],
    localOrExisting: 'Existing',
    type: 'record',
    operation: 'create',
    roles: [sampleAdmin, 'x_other_scope.manager'],
    table: 'x_acl_sample_table',
})

Acl({
    $id: Now.ID['read_acl'],
    localOrExisting: 'Existing',
    type: 'record',
    operation: 'read',
    table: 'x_acl_sample_table',
    condition: 'field=value^active=true',
})

Acl({
    $id: Now.ID['rest_acl'],
    name: 'sample_api',
    type: 'rest_endpoint',
    operation: 'execute',
    roles: [sampleAdmin],
    securityAttribute: 'user_is_authenticated',
})
```

**Design notes connecting this to the architecture rules:**
- One `Acl()` per operation — the explicit CRUD package from `05-security-design.md` maps 1:1 onto four `Acl()` builders (create/read/write/delete) plus a `table.*`-equivalent via field-level rules.
- `condition` takes an encoded query string — the same condition discipline as Check 2.
- `securityAttribute` (e.g., `has_admin_role`, `user_is_authenticated`) is the modern attribute-based control referenced in `05-security-design.md` §2.
- `rest_endpoint` ACLs gate Scripted REST APIs — the `execute` operation pattern from `04-integration-design.md`.

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
5. **Skipping `localOrExisting` decisions:** determine per record whether the definition creates new metadata or binds to existing records — an accidental `'New'` duplicates OOTB objects (Check 6 territory).

---

## Authoritative Sources
- [ServiceNow/sdk-examples (official runnable Fluent samples)](https://github.com/ServiceNow/sdk-examples)
- [Now SDK CLI Reference](https://servicenow.github.io/sdk/cli)
- `npx @servicenow/sdk explain` — versioned Fluent API documentation shipped with the SDK (requires SDK ≥ 4.6.0)