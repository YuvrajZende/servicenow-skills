# AI, Agents & Automation Architecture (`13-ai-automation.md`)

> Guidance for designing with ServiceNow's AI surface: AI Agents, Now Assist (GenAI), Virtual Agent, Predictive Intelligence, and AI Search.

> **⚠️ Hard Rule — Verify Entitlements Before Designing:** AI capabilities are separately licensed and release-gated. Before proposing any AI component, instruct the user to verify the instance's entitlements, activated plugins, and platform release. Never design against an AI capability "from memory" — this domain changes faster than any other in the platform.

---

## 1. AI Capability Selection Matrix

| Capability | Use For | Key Constructs | Licensing Reality |
|------------|---------|----------------|-------------------|
| **AI Agents / Agentic AI** | Autonomous multi-step tasks: triage, enrichment, resolution proposals with tool access | Agent definitions, tools, triggers, use cases (tables `sn_aia_agent`, `sn_aia_tool`, `sn_aia_agent_tool_m2m`, `sn_aia_trigger_configuration`) | Entitled add-on; verify release and entitlement. |
| **Now Assist / GenAI skills** | Summarization, generation, code assist on standard workflows | Platform-provided skills per product (ITSM, CSM, HR, Creator) | Entitled add-on; per-product. |
| **Virtual Agent (VA)** | Guided conversational intake and deflection | Topics (`sys_cb_topic` design-time / `sys_cs_topic` runtime), NLU models (`sys_nlu_model`) | Part of broader entitlements; NLU model training consumes capacity. |
| **Predictive Intelligence (PI)** | Classification, similarity, regression, clustering on record data | ML solution definitions (`ml_solution`, `ml_capability_definition`); training via solution records | Entitled add-on; requires training data volume. |
| **AI Search** | Relevance-tuned enterprise search across knowledge, catalog, community | Search profiles, federated sources | Entitled add-on replacing legacy text search. |
| **Document Intelligence** | Invoice/ID document extraction into structured fields | Document extraction models | Entitled add-on. |

---

## 2. AI Agents (Agentic AI) Design Standards

AI Agents are defined as: an agent (persona + instructions) with bound tools, invoked by a use case, triggered manually or by automated triggers.

1. **Tools are the unit of security.** Tool types include script, CRUD, RAG, subflow, Flow Designer action, catalog, and topic. Every tool an agent can call is a capability the agent can exercise — review tools with the same rigor as ACLs: **least-privilege tool sets**, never "give it everything."
2. **Wrap, don't expose:** bind agents to Flow Designer subflows/actions and CRUD tools scoped to specific tables — not to raw scripts with broad GlideRecord access. The agent inherits the tool's scope and permissions.
3. **Triggers:** automated trigger configurations (`sn_aia_trigger_configuration`) map events/conditions to use cases. Apply Check 3 discipline — heavy agent runs must not execute in the user's synchronous transaction.
4. **Human-in-the-loop for consequential actions:** any agent step that writes records, sends external communications, or spends money must be review-gated in the design. Autonomous execution is a deliberate, documented decision — not a default.
5. **Observability:** design for execution trace review before go-live; agent runs must be auditable the same way flow executions are.

---

## 3. Virtual Agent & NLU

- **Topics are design-time (`sys_cb_topic`) → published to runtime (`sys_cs_topic`).** Migrating topics between instances requires both design and runtime artifacts plus NLU model attachments — plan deployment like any metadata (see `06-delivery-design.md`).
- **NLU models** (`sys_nlu_model`) bind intents to topics per language (`sys_cs_topic_language`). Utterance-entity mappings (`m2m_sys_nlu_utterance_entity`) are the training surface.
- **Design standards:** every VA intake path must have a live-agent handoff design (`sys_cs_live_agent_setup`); every topic must have a fallback path for unmatched intents; log analysis (`open_nlu_predict_log`) feeds the utterance improvement loop.
- **Model artifacts must travel with deployments** (e.g., `ml_model_artifact`) — a topic migration without its published NLU model breaks intent matching in the target instance.

---

## 4. Predictive Intelligence

1. **Capabilities:** classification (predict a field value), similarity (find related records), regression (predict a number), clustering (group records). Choose per business question, not per fashion.
2. **Training data is the design constraint:** solutions train on historical record data (`ml_solution`); dirty or sparse training data produces confidently wrong predictions. Design the data-quality gate **before** the model.
3. **Environment & training data dependencies:** Predictive Intelligence models train against historical record volume.
   > ⚠️ Unverified: A figure of 300+ historical records per category is widely cited across ServiceNow practitioner forums and implementation guides for acceptable classification accuracy, but official minimum thresholds vary by solution type and class balance.
   In multi-instance pipelines (Dev ➔ Test ➔ Prod), sub-production environments frequently lack the data volume or variety needed for quality training; the architecture must explicitly document where models are trained and how accuracy is validated before production activation.
4. **PI outputs augment, humans decide:** predicted category/similarity are suggestions into the workflow, never silent overrides of fulfiller judgment without an explicit automation decision.

---

## 5. AI Anti-Patterns

1. **Designing AI without an entitlement check** — the single most common architecture failure in this domain. Gate every AI component behind a verified entitlement step.
2. **Tool sprawl / god-agents:** one agent with every tool. Decompose into focused agents per use case with minimal tool sets.
3. **Trusting model output for irreversible actions:** no autonomous deletes, external sends, or approvals without human-in-the-loop or strict guardrails.
4. **Training on production PII without a data-privacy review:** GenAI/PI usage of record data requires the same privacy review as any integration exposing data.
5. **Skipping evaluation:** AI features ship with acceptance criteria like any other component — define precision/recall or deflection targets in the architecture document, and measure.
6. **Hardcoding model/topic sys_ids in scripts** (see `09-anti-patterns.md` §5).

---

## 6. Output in the Architecture Document

When the design specifies AI, Now Assist, or automation capabilities, the architecture document must include an **AI Agent Tool & Entitlement Map** (template §6B, rendered only for components that survive the entitlement check):

| Capability | Business Use | Entitlement Verified? | Inputs / Training Data | Human-in-the-Loop Point | Success Metric |
|------------|--------------|----------------------|------------------------|-------------------------|----------------|
| AI Agent: triage | Auto-categorize new cases | ☐ Instance: ___ | Case text history | Approval before state change | ≥X% correct triage |

---

## Authoritative Sources
- [ServiceNow Documentation: Now Assist and Generative AI](https://docs.servicenow.com/bundle/xanadu-intelligent-experiences/page/administer/now-assist/concept/now-assist.html)
- [ServiceNow Documentation: Virtual Agent Architecture](https://docs.servicenow.com/bundle/xanadu-build-workflows/page/administer/virtual-agent/concept/virtual-agent-overview.html)
- [ServiceNow Documentation: Predictive Intelligence](https://docs.servicenow.com/bundle/xanadu-intelligent-experiences/page/administer/predictive-intelligence/concept/predictive-intelligence.html)