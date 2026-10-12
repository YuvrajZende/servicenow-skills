# UI & Experience Architecture (`03-ui-layer.md`)

> Guidance for selecting the correct interface tier across Next Experience Workspaces, UI Builder, Service Portal / Employee Center, and Platform UI.

---

## 1. UI Layer Comparison & Selection Matrix

ServiceNow provides four primary presentation layers. Choosing the wrong UI tier creates poor user adoption and excessive maintenance:

| UI Layer | Target Personas | Key Strengths | When to Select | Technology Foundation |
|----------|-----------------|---------------|----------------|-----------------------|
| **Configurable Workspace** | Fulfillers, Tier-1/2 Agents, Dispatchers | Multi-tab browsing, Agent Assist side panels, Playbooks, unified Activity Stream | Work-intensive agent fulfillment where users handle multiple records simultaneously. | Next Experience Framework |
| **UI Builder (UIB)** | Custom role-specific app users | Drag-and-drop WYSIWYG page designer, custom layouts, component composition | Custom dashboards, specialized record views, or novel application experiences. | Now Component Framework |
| **Employee Center / Service Portal** | End users, Employees, Requesters | Consumer-grade search, catalog browsing, knowledge articles, ticket status | Self-service intake and status tracking for non-fulfiller users. | Angular / Service Portal widgets |
| **Platform UI (Classic Forms & Lists)** | System Administrators, Developers | Direct schema access, related lists, standard list filtering, zero layout overhead | Administrative setup, configuration tables, system backend. | Classic Form & List Engine (`navpage.do`) |
| **SDK UI Page (modern JS frameworks)** | Developers building bespoke embedded experiences | Full frontend framework power (React, Vue, Svelte, SolidJS) with a ServiceNow deployment path | Only when UI Builder and standard components genuinely cannot deliver the requirement — the highest-maintenance tier. | ServiceNow SDK + framework (see `17-fluent-cookbook.md`) |

---

## 2. Persona-Driven UI Architecture Pattern

Every comprehensive application architecture must map personas to their corresponding interface:

```
+─────────────────────────────────────────────────────────────────────────────+
|                         Application Persona Mapping                         |
+─────────────────────────────────────────────────────────────────────────────+
|                                                                             |
|  [ End User / Requester ]       ──►   Employee Center / Service Portal      |
|  (Submit request, view status)        (Record Producer / Catalog Item)      |
|                                                                             |
|  [ Fulfiller / Agent ]          ──►   Configurable Workspace                |
|  (Work items, triage, assist)         (Record Page, Multi-tab, Agent Assist)|
|                                                                             |
|  [ Executive / Manager ]        ──►   Next Experience Dashboard / Analytics |
|  (Review KPIs, approve items)         (UI Builder Page, Performance Analytics)|
|                                                                             |
|  [ System Administrator ]       ──►   Platform UI (Core Forms & Lists)      |
|  (Configure rules, ACLs, data)        (Studio, Table Dictionary, Scripts)   |
|                                                                             |
+─────────────────────────────────────────────────────────────────────────────+
```

---

## 3. UI Decision Criteria

### When to choose Configurable Workspace over Classic Platform UI:
- Fulfillers work across multiple records at once (benefits from browser tab consolidation).
- Agents need contextual guidance (Agent Assist recommendations, Knowledge search, Playbook process flows).
- Unified interaction handling (chat, telephony, email, cases in a single interface).

### When to choose UI Builder:
- Requirements call for custom responsive grids, dynamic page routing, or bespoke data visualization.
- Standard form layouts cannot accommodate required layout density or custom web components.

### When to choose an SDK UI Page (modern framework):
- A genuinely bespoke interactive experience is required that UI Builder composition cannot express — this is the **last resort** tier, justified only after UI Builder and standard components are ruled out (apply the OOTB-first test from `00-ootb-first.md` at the UI layer).
- The org accepts the maintenance load: framework upgrades, build tooling, and upgrade compatibility are now the app's responsibility — record this in the architecture document's risk register.
- Official runnable samples for React, Vue, Svelte, and SolidJS UI Pages exist in [ServiceNow/sdk-examples](https://github.com/ServiceNow/sdk-examples) — start from those, never from scratch.

### When to choose Employee Center / Service Portal:
- Target audience does not hold an `itil` or fulfiller license.
- Clean, guided submission forms (Record Producers, Catalog Items) are required.
- Integration with knowledge base and virtual agent self-service.

---

## 4. UI Architecture Best Practices & Pitfalls

1. **Avoid Custom HTML/Angular in Forms:** Do not build custom UI Pages or embedded macros where standard Form Layouts or Workspaces suffice.
2. **Design Forms for Workspace Compatibility:** Use standard Form Sections and Form Views rather than custom DOM-manipulating Client Scripts.
3. **Prefer UI Policies for Form Behavior:** UI Policies work automatically across both Classic UI and Next Experience Workspaces. Client Scripts with DOM manipulation (`document.getElementById`) fail in Workspaces and Scoped applications.
4. **Mobile Responsiveness:** Record Producers in Service Portal and standard UI Builder pages adapt cleanly to mobile; classic platform UI forms do not.

---

## 5. Cross-Cutting UI Concerns

| Concern | Standard |
|---------|----------|
| **Mobile (Now Mobile / Agent app)** | If any persona is field/mobile, design for the mobile apps explicitly — record views, offline expectations, and push notifications (`11-notifications.md`) — not as a "portal shrinks down" afterthought. |
| **Playbooks** | Process-driven fulfillment journeys in Configurable Workspace. Choose Playbooks when the fulfiller follows a defined multi-stage process; a Playbook is process UX, not a dashboard. |
| **Accessibility (WCAG / Section 508)** | Portal and Workspace surfaces must meet the org's accessibility standard. Custom UI (UI Builder pages, widgets, custom components) is where accessibility gets broken — standard components carry platform accessibility work. |
| **Theming** | Next Experience theming tokens / brand customization apply across Workspace, portals, and mobile. Configure theming centrally; never restyle surfaces with custom CSS hacks per page. |
| **Localization (i18n)** | Custom apps need translated field labels, choices, and UI messages (`sys_ui_message`) if any persona is non-English. Decide the language scope at design time (see `01-data-model.md` translated fields). |
| **UI Action surfaces** | Every UI Action designed in `02-logic-placement.md` must be verified for its Workspace/Portal surface — classic buttons do not auto-appear in Workspace. |

---

## Authoritative Sources
- [ServiceNow Documentation: Next Experience UI Builder](https://docs.servicenow.com/bundle/xanadu-application-development/page/administer/ui-builder/concept/ui-builder.html)
- [ServiceNow Documentation: Configurable Workspaces](https://docs.servicenow.com/bundle/xanadu-platform-user-interface/page/administer/workspace/concept/configurable-workspace.html)
- [ServiceNow Documentation: Employee Center](https://docs.servicenow.com/bundle/xanadu-employee-service-management/page/product/employee-center/concept/employee-center-overview.html)
- [Official SDK samples: UI Pages with React / Vue / Svelte / SolidJS](https://github.com/ServiceNow/sdk-examples)
