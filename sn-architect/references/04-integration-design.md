# Integration & Inbound/Outbound Staging Architecture (`04-integration-design.md`)

> Guidance for designing REST APIs, IntegrationHub spokes, Import Sets, Robust Import Transformers, MID Server relays, authentication, and error resilience.

---

## 1. Integration Method Selection Matrix

| Pattern | Direction | Primary Use Case | Transport / Mechanism | Staging Table Required? | MID Server? |
|---------|:---------:|------------------|----------------------|:------------------------:|:-----------:|
| **Table API** | Inbound | Direct, single-record CRUD by trusted external systems | REST / JSON | ❌ Direct to target | No |
| **Scripted REST API** | Inbound | Bespoke endpoints, complex payload validation, custom HTTP status codes, multi-table operations | REST / JSON | Optional | No |
| **Import Set + Transform Map** | Inbound | Bulk data feeds, CSV/JSON file ingestion, legacy ETL | Staging (`u_imp_*`) | ✅ Yes | Optional |
| **Robust Import Transformer (RTE)** | Inbound | High-volume structured data, complex ETL pipelines, multi-target loading | Staging / Memory Engine | ✅ Yes | Optional |
| **IntegrationHub Spoke** | Outbound | Standard enterprise SaaS integrations (Jira, Slack, Microsoft 365, Workday) | REST / Flow Action | ❌ Direct | Optional |
| **RESTMessageV2 (in Script Include)** | Outbound | Custom REST endpoints without spokes, high-throughput custom payloads | HTTPS / Script | ❌ Direct | Optional |
| **MID Server Relay** | Inbound / Outbound | Secure access to on-prem databases, internal APIs, LDAP, SSH, PowerShell behind firewalls | ECC Queue polling | Depends on payload | **Mandatory** |

---

## 2. Inbound Data Ingestion: Scheduled Jobs vs Import Sets / RTE

Architects frequently confuse when to use a Scheduled Script Execution vs an Import Set pipeline:

```
+─────────────────────────────────────────────────────────────────────────────+
|                         Inbound Ingestion Decision                          |
+─────────────────────────────────────────────────────────────────────────────+
| External system pushes data via API?                                        |
|   ├── Simple single-record CRUD? ──────────► Scripted REST / Table API      |
|   └── Bulk batch payload? ────────────────► Import Set API (u_imp_* table)  |
|                                                                             |
| ServiceNow pulls data from external system on schedule?                     |
|   ├── Complex ETL, deduplication, transform mapping?                        |
|   │     └── Scheduled Data Import ──► Import Set Table ──► Transform/RTE     |
|   │                                                                         |
|   └── Internal database maintenance, batch record calculations?              |
|         └── Scheduled Script Execution (sys_trigger) with GlideRecord chunking|
+─────────────────────────────────────────────────────────────────────────────+
```

### Import Set & RTE Architectural Best Practices:
1. **Never write directly to target production tables from external feeds:** Always stage data in an Import Set table (`u_imp_*`) that extends `sys_import_set_row`.
2. **Coalesce Fields:** Define clear, indexed coalesce keys to ensure deterministic insert vs update behavior.
3. **Robust Import Transformer (RTE):** Use RTE instead of legacy Transform Maps for complex multi-entity mapping (e.g., CSDM data loading, where one payload creates both a CI and an Asset record).

---

## 3. Outbound Integration: IntegrationHub Spoke vs Custom RESTMessageV2

| Criterion | IntegrationHub Spoke | Custom RESTMessageV2 in Script Include |
|-----------|----------------------|----------------------------------------|
| **Authoring Experience** | Drag-and-drop Flow Designer action | Procedural JavaScript class |
| **Spoke Availability** | Official Store Spoke exists | Custom internal API or legacy protocol |
| **Payload Complexity** | Standard JSON/REST payloads | Dynamic streaming or non-standard protocols |
| **Licensing** | Requires IntegrationHub subscription tier | Core platform capability |
| **Maintenance** | Automatically updated by spoke vendor | Maintained entirely by internal development team |

---

## 4. Authentication & Credential Storage

### Security Directives:
1. **Zero Hardcoded Secrets:** Never store client secrets, API keys, or bearer tokens in script includes or system properties.
2. **Connection & Credential Aliases (`sys_alias`):** Always route outbound connections through aliases. This decouples endpoint URLs and auth tokens from business logic and allows seamless environment switching (Dev ➔ Test ➔ Prod).
3. **Supported Auth Mechanisms:**
   - **OAuth 2.0 (Preferred):** Authorization Code grant for user-delegated access; Client Credentials grant for machine-to-machine integrations.
   - **Mutual TLS (mTLS):** For high-security banking, healthcare, or government integrations requiring bidirectional certificate verification.
   - **API Key:** Permissible for low-sensitivity external calls; store in Credential store.

---

## 5. Resiliency, Retries & Idempotency

### Inbound Idempotency Pattern:
External systems retrying network requests must not duplicate transactions.
- Require an external transaction ID passed in HTTP header `X-Correlation-ID`.
- Check if a record with `correlation_id = header_val` already exists.
- If found, return existing record identifier with HTTP `200 OK` without re-executing logic.

### Outbound Retry & Circuit Breaker:
- **Exponential Backoff:** Configure IntegrationHub Retry Policies with exponential backoff (e.g., 3 retries at 2s, 8s, 30s intervals) for transient network errors (HTTP 429, 502, 503).
- **Circuit Breaker:** If an external endpoint returns consecutive 5xx errors for >5 minutes, trip the circuit breaker in a Script Include to fail fast and prevent thread pool exhaustion.
- **Dead Letter Staging:** Divert permanently failed outbound payloads to a custom error table (`x_<app>_failed_transaction`) for alert generation and manual replay.

---

## 6. Special Integration Patterns

| Pattern | Standard |
|---------|----------|
| **e-Bonding (system-to-system ticket exchange)** | Define a written contract per direction: field mapping table, ownership rules, loop prevention (never re-sync records back to their source), and a correlation ID on both sides. Treat the remote system as untrusted input (validate against its own schema). |
| **Attachments** | Transfers use the Attachment API with size limits in mind; for large/binary payloads prefer link-based exchange (store externally, send a reference) over inlining into records. |
| **Webhooks (inbound push)** | Every webhook endpoint is an unauthenticated-until-proven surface: verify signatures/HMAC, reject replays (timestamp/nonce), and treat payloads as untrusted input. Applies to inbound email too (`11-notifications.md`). |
| **API versioning (Scripted REST)** | Version the namespace path from day one (`/api/<vendor>/<app>/v1/...`); additive changes only within a version; deprecate explicitly with a documented sunset. |
| **CMDB writes** | Any integration loading CIs must route through the IRE — never direct `cmdb` writes. See `14-cmdb.md` before designing any CI feed. |

---

## 7. Identity & Authentication Integration

The identity layer feeds every integration and every user record:

- **SSO (SAML / OIDC):** the standard enterprise path for user authentication — design the IdP integration as an architecture component (name the IdP, the federation protocol, and the provisioning trigger), never as "login will be configured."
- **User provisioning:** LDAP vs SCIM vs automated creation — choose one authoritative provisioning path. **NEVER extend `sys_user`** for provisioning attributes (see `01-data-model.md`).
- **Service accounts for integrations:** every outbound integration uses a dedicated, named service account with least-privilege roles (never a human user's account, never admin) — and its credentials live in Connection & Credential Aliases (§4).
