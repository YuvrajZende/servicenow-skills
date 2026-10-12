# Step 0: Out-of-the-Box (OOTB) Evaluation

> **Golden Rule of ServiceNow Architecture:**  
> Before designing any custom table, script, or workflow, verify whether the platform already provides that capability natively. Customizing what already exists creates technical debt, increases upgrade friction, and wastes subscription value.

---

## 1. The OOTB Evaluation Protocol

When a requirement is presented, the architect must evaluate these 4 questions in order:

### Question 1: Does an existing base table represent this business entity?
ServiceNow provides rich data models across product lines. Customizing should never be the default.

| Domain | Standard Tables to Evaluate | When to Use Instead of Custom Table |
|--------|-----------------------------|-------------------------------------|
| **Work Items & Operations** | `task`, `planned_task`, `ticket` | Any record with assigned groups, individual assignees, states, SLAs, or approval lifecycles. |
| **User & Customer Contacts** | `sys_user`, `customer_contact` (CSM B2B, extends `sys_user`), `csm_consumer` (CSM B2C, standalone) | People. **In custom applications, never create a custom user table and never extend `sys_user`.** |
| **Companies & Vendors** | `core_company` | Accounts, vendors, manufacturers, clients. |
| **Physical / Logical Assets** | `alm_asset`, `cmdb_ci` | Physical equipment, software licenses, servers, network appliances. |
| **Omnichannel Intake** | `interaction` | Inbound chat, phone, or email requests before converting to work items. |
| **Customer Service** | `sn_customerservice_case` | B2B or B2C customer service cases (CSM). |
| **HR Service Delivery** | `sn_hr_core_case` | Employee requests, onboarding, benefits inquiries (HRSD). |
| **Requests & Catalog** | `sc_request`, `sc_req_item`, `sc_task` | User-initiated catalog orders with fulfillment stages. |

### Question 2: Does an existing Plugin, Store App, or Spoke solve this?
- Check the ServiceNow Store and standard platform plugins.
- For integrations, verify if an official **IntegrationHub Spoke** (e.g., Jira, Azure DevOps, Microsoft Teams, Slack, DocuSign, SAP) is available before designing a Scripted REST API or custom REST Message.

### Question 3: Can existing workflows, actions, or subflows be reused?
- Check Flow Designer for pre-built subflows (e.g., standard multi-level manager approvals, notifications, task generation).
- Reuse existing approval engines (`sysapproval_approver`) rather than creating custom approval tables or status flags.

### Question 4: Is this Configuration or Customization?

| Decision Criterion | Configuration (Preferred) | Customization (Requires Justification) |
|--------------------|---------------------------|----------------------------------------|
| **Schema** | Adding custom fields to existing tables, configuring forms | Creating new standalone or extended tables |
| **Logic** | UI Policies, standard Flow Designer triggers, Data Policies | Scripted Business Rules, Script Includes, Client Scripts |
| **Intake** | Service Catalog Item, Record Producer | Custom UI Page, bespoke Scripted REST API |
| **Lifecycle** | Standard state model, SLA Definitions | Custom state machines with procedural script controls |
| **Upgrade Impact** | Seamless; platform updates base objects automatically | High; requires skipped record review and regression testing |

---

## 2. When Custom Is Justified

A custom component (e.g., new table, Scripted REST API, custom Script Include) is justified **ONLY** when:
1. **Isolated Data Domain:** The data represents reference, configuration, or staging data with no lifecycle (e.g., an internal tax rate table, API lookup mapping).
2. **License / Entitlement Boundary:** Building within an App Engine / Custom Application entitlement without access to specific licensed product tables.
3. **High-Volume Isolation:** Transactions occur at volumes (millions/month) that would degrade performance if mixed into shared operational tables like `task`.
4. **Distinct Non-Task Lifecycle:** The record represents a pure approval, ledger entry, or token with zero overlap with task assignment, SLAs, or standard fulfillment.

---

## 3. Output in the Architecture Document

Every architecture document produced by `sn-architect` must explicitly contain an **OOTB Evaluation Summary**:
- **OOTB Candidates Evaluated:** List of standard tables and modules evaluated.
- **Decision:** Extension vs Configuration vs Custom Standalone.
- **Justification:** Clear business or technical rationale if custom components are chosen.

---

## Authoritative Sources
- [ServiceNow Documentation: Table extension and the data model](https://docs.servicenow.com/bundle/xanadu-platform-administration/page/administer/managing-data/concept/c_DataDictionaryTables.html)
- [ServiceNow Documentation: Customer Service Management tables](https://docs.servicenow.com/bundle/xanadu-customer-service-management/page/product/customer-service-management/reference/r_CustomerServiceTables.html)
- [ServiceNow Community: Common Service Data Model (CSDM) Overview](https://www.servicenow.com/community/developer-articles/common-service-data-model-csdm-overview/ba-p/2275468)
