# Security Design & Access Control Architecture (`05-security-design.md`)

> Guidance for designing Contextual Security (ACLs), roles, Restricted Caller Access (RCA), and hardening against data leakage.

---

## 1. The Default Deny Model & Mandatory Explicit ACLs

### The System Property Reality
In ServiceNow, Contextual Security behavior is governed by the system property:
`glide.sm.default_mode`
- **When set to `deny`** (standard in High Security instances): Requests for tables/operations without matching ACL rules are blocked.
- **When set to `allow`** (legacy or misconfigured instances): Requests without matching ACLs are granted access to any authenticated user.

### The Architect's Hard Rule
> **Never rely on implicit deny.**  
> 1. Always instruct the implementation team to verify that `glide.sm.default_mode` is set to `deny`.  
> 2. **Always create explicit CRUD ACLs (`create`, `read`, `write`, `delete`) for every custom table.**  
> If an instance is ever cloned, misconfigured, or accessed via unmonitored endpoints, explicit ACLs ensure the table remains strictly protected.

---

## 2. Exact ACL Evaluation Order

For a user to view or edit a specific field on a record, **both the Table-level ACL and the Field-level ACL must evaluate to `true`**.

```
                        User requests access to [table.field]
                                         │
                                         ▼
                     ┌───────────────────────────────────────┐
                     │       Step 1: Table-Level ACL         │
                     │ (Does the user have access to row?)   │
                     └──────────────────┬────────────────────┘
                                        │ Passes?
                         ┌──────────────┴──────────────┐
                         │ YES                         │ NO
                         ▼                             ▼
     ┌───────────────────────────────────────┐   [ACCESS DENIED]
     │       Step 2: Field-Level ACL         │
     │(Does the user have access to column?) │
     └──────────────────┬────────────────────┘
                        │ Passes?
         ┌──────────────┴──────────────┐
         │ YES                         │ NO
         ▼                             ▼
   [ACCESS GRANTED]              [ACCESS DENIED]
```

### Specificity Hierarchy
When searching for matching ACL rules, the engine evaluates from **most specific to least specific**:

1. **Table-level ACL Resolution:**
   `table` ➔ `parent_table` ➔ `*` (global table wildcard)
2. **Field-level ACL Resolution:**
   `table.field` ➔ `parent_table.field` ➔ `*.field` ➔ `table.*` (table field wildcard) ➔ `parent_table.*` (parent field wildcard) ➔ `*.*` (global wildcard)

### Evaluation-Stop Behavior
- The user must pass the **table-level ACL first**; failing the table ACL denies access to **all fields**, even ones the user would pass a field rule for.
- For field access, the **first matching field ACL in the hierarchy that the user passes grants access and stops evaluation**. A matching field-level ACL that fails does **not** fall back to a more permissive wildcard (`table.*` cannot rescue a failed `table.field` rule).

### Inside Any Individual ACL Rule
Within a single matching ACL rule, **ALL criteria must evaluate to true**:
1. **Roles:** User must possess at least one of the roles listed (OR logic across roles in the list).
2. **Security Attributes (if configured):** If Attribute-Based Access Control (ABAC) or Data Filtration attributes are defined, the user session must satisfy the policy requirements.
3. **Condition Builder:** The condition expression must evaluate to `true` against the target record.
4. **Script Block:** If a script is configured, it must set `answer = true`.

*(Note: What is commonly referred to as "row-level security" is not a separate third step; it is simply the Condition and Script evaluated during the Table-level ACL on a specific record).*

---

## 3. Minimum ACL Checklist for Custom Tables

Every custom table (`x_<app>_<table>` or `u_<table>`) requires this baseline ACL package before moving to production:

| Operation | Target Object | Required Configuration | Failure Risk if Omitted |
|-----------|---------------|------------------------|-------------------------|
| **Read** | `table` | Scoped reader or fulfiller role | Users cannot view records |
| **Read** | `table.*` (Wildcard) | Scoped reader or fulfiller role | Fields remain blank even if row passes |
| **Create** | `table` | Scoped creator role | Users cannot insert records |
| **Write** | `table` | Scoped fulfiller role + State condition | Users cannot update records |
| **Write** | `table.*` (Wildcard) | Scoped fulfiller role | Fields remain read-only |
| **Delete** | `table` | Scoped admin role only | Unauthorized record deletion |

### Row-Level Filtering: ACL Conditions + Before-Query Business Rules

For "users see only their own records" designs, pair the ACL condition with a **before-query Business Rule** on the table (`current.addQuery('opened_by', gs.getUserID())` or role-conditional variants). The query rule filters rows *before* list rendering and aggregation, which keeps list views fast and closes aggregate/report leaks that condition-only ACLs cannot. The ACL remains the enforcement layer; the query rule is the performance and leakage backstop — designs should specify both.

---

## 4. Empty ACL Vulnerabilities & Public Surfaces

### Empty ACL Threat
An **Empty ACL** is an ACL rule created with **no roles, no conditions, and no script**.
- In ServiceNow, an empty ACL evaluates to `true` for **any authenticated user**.
- If created on sensitive tables, it unintentionally opens unrestricted read or write access to every user in the company.

### Public Surface & Guest Session Leakage
Unauthenticated users operate under the platform **Guest user session** (look up user ID `guest` in `sys_user` to confirm the `sys_id` on your instance — it is a default system record but the `sys_id` can vary across installations).
- **Research Finding (AppOmni / CVE disclosures):** Instances leaked corporate data when tables had empty ACLs or lacked explicit ACLs while being queried by public UI Pages, public Service Portal widgets, or APIs marked with `Requires authentication = false`.
- **Architectural Safeguards:**
  1. Never mark custom Scripted REST APIs as public unless explicitly intended for public anonymous traffic.
  2. Audit public UI Pages and widgets (`public: true`).
  3. Ensure all client-callable Script Includes validate user authentication and roles inside `initialize` or method entry points.

---

## 5. Role Design & Application Scope Security

### Role Architecture Standards
- **Naming Standard:** `x_<vendor>_<app>.<role_name>` (e.g., `x_acme_loan.admin`, `x_acme_loan.fulfiller`, `x_acme_loan.user` — dot separator after scope prefix per official ServiceNow SDK patterns).
- **Principle of Least Privilege:** Create distinct roles for Read-Only vs Fulfill vs Admin.
- **Group Assignment:** Assign roles exclusively to Groups (`sys_user_group`), never directly to individual user records.

### Restricted Caller Access (RCA) & Cross-Scope Security
When an application resides in a custom scope:
- **Application Access Settings:** Configure Table Dictionary settings (`Can read`, `Can create`, `Can update`, `Can delete` from other application scopes).
- **Restricted Caller Access (RCA):**
  - Use `Caller Tracking` during initial development to log cross-scope calls.
  - Transition to `Caller Restriction` prior to testing and production promotion to require explicit approval for every cross-scope invocation.
- **Cross-Scope Privileges:** Ensure all cross-scope database queries, Script Include calls, and event triggers are tracked and packaged into deployment artifacts.

---

## 6. Instance-Level Hardening & Data Protection

ACLs govern records; the instance layer governs everything around them:

| Layer | Standard |
|-------|----------|
| **Instance access** | IP access controls / allowed ranges, admin two-factor authentication, and the High-Security plugin settings posture are instance design inputs — verify them when the design involves sensitive data. |
| **Public surface audit** | Enumerate every `public: true` surface (UI pages, processors, Scripted REST) and justify each in the architecture document (extends §4). |
| **Sensitive data in logs** | `gs.info`/`gs.warn` must never log tokens, PII, or secrets — logs are broadly readable. Treat log statements as part of the security review. |
| **PII handling** | Identify which fields are PII in the data model; their read/write ACLs and retention (archiving, see `01-data-model.md`) follow the org's privacy policy. |
| **Domain separation** | If the platform hosts multiple tenants/partitions, domain separation is an architectural decision with deep ACL and process implications — surface it explicitly rather than discovering it in testing. |

---

## 7. Security Checklist Additions for Custom Tables

Beyond the §3 ACL package, verify before production:

- [ ] Client-callable Script Includes validate roles at every public method (Rule 2.4).
- [ ] No security-relevant decision made client-side (client scripts are advisory; server ACLs are the enforcement).
- [ ] Inbound email actions and webhooks validate the sender (unauthenticated write surface — see `11-notifications.md`, `04-integration-design.md`).
- [ ] Service accounts are least-privilege and dedicated (see `04-integration-design.md` §7).
- [ ] Security Attributes reviewed for applicability on the target instance release (see §2).

---

## Authoritative Sources
- [ServiceNow Documentation: Access Control Rules (ACL)](https://docs.servicenow.com/bundle/xanadu-platform-security/page/administer/contextual-security/concept/access-control-rules.html)
- [ServiceNow Documentation: Restricted Caller Access (RCA)](https://docs.servicenow.com/bundle/xanadu-application-development/page/build/applications/concept/c_RestrictedCallerAccess.html)
- [ServiceNow Documentation: High Security Settings and Default Deny](https://docs.servicenow.com/bundle/xanadu-platform-security/page/administer/security-advisory/concept/high-security-settings.html)
