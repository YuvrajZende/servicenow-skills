# Knowledge Management Architecture (`12-knowledge.md`)

> Guidance for designing knowledge strategy — knowledge bases, article lifecycle, KCS adoption, and search integration — for any application that captures or serves institutional know-how.

---

## 1. Knowledge Architecture Decision Points

| Decision | Options | Standard |
|----------|---------|----------|
| **How many KBs?** | One per audience (internal / fulfiller / end-user) | Separate KBs by audience permissions, not by product. End-user content in a public-facing KB; fulfiller content internal. |
| **Who owns articles?** | Ownership groups + article authors | Every KB has ownership groups; articles without an accountable owner rot. Design ownership as part of the org model, not an afterthought. |
| **Lifecycle** | Draft → Review → Published → Retire (versioned) | Use the platform article workflow and versioning rather than custom state fields on custom tables. |
| **Who can write?** | `kb_*` role design per KB via can-read/can-write/user criteria | Least privilege per KB; user criteria for audience gating (employees vs customers vs agents). |
| **Where surfaced?** | Employee Center / Portal search, Workspace Agent Assist, Virtual Agent topics | Knowledge must appear in the tool the persona already uses — map KB → surface per persona (see `03-ui-layer.md`). |

---

## 2. Knowledge in the Application Architecture

```
[Capture] ──► [Author (draft)] ──► [Review workflow] ──► [Publish] ──► [Search surfaces]
     ▲                                                                      │
     └────────────── Feedback / analytics flag gaps ◄─── Usage telemetry ───┘
```

1. **Capture at the point of resolution:** fulfiller work notes and resolutions are the raw material; design article creation prompts into the Workspace flow (KCS "solve loop").
2. **Search is the delivery mechanism:** knowledge that cannot be found does not exist. Verify which search stack the instance uses (AI Search vs legacy text search) and configure knowledge sources accordingly — this is an entitlement check, not an assumption.
3. **Feedback loop:** article feedback ratings and search-without-click telemetry identify content gaps; route them into the review workflow rather than building custom feedback tables.
4. **Virtual Agent integration:** published knowledge feeds VA topic responses and contextual search — design the KB with VA consumption in mind (see `13-ai-automation.md`).

---

## 3. Knowledge Anti-Patterns

1. **Custom knowledge tables:** never build a "tips and tricks" custom table when `kb_knowledge` provides lifecycle, versioning, permissions, and search. Extending knowledge tables for specialized content types is the only justified custom path — and requires the same OOTB-first justification as any other extension.
2. **One giant KB with security via article flags:** audience control belongs at KB/user-criteria level, not per-article hacks.
3. **No retirement process:** publishing without retiring produces contradictory articles that erode trust.
4. **Knowledge blocks everywhere:** knowledge blocks (shared reusable content) create cross-article coupling; use sparingly, only for genuinely shared content.
5. **Skipping the review workflow:** self-publishing without review is acceptable only where the org explicitly accepts the quality risk — document the decision.

---

## 4. Output in the Architecture Document

Every architecture document with a knowledge component must include a **Knowledge Base Map** (template §6C):

- **Knowledge Base map:** KB name → audience (user criteria) → surfaces (Portal search, Workspace Agent Assist, VA) → ownership groups.
- **Lifecycle decision:** standard review workflow vs self-publish (with sign-off).
- **KCS posture:** whether the design includes solve-loop article capture, and which personas are incentivized to author.

---

## Authoritative Sources
- [ServiceNow Documentation: Knowledge Management](https://docs.servicenow.com/bundle/xanadu-service-management-framework/page/product/knowledge-management/concept/knowledge-management.html)
- [ServiceNow Documentation: User Criteria for Knowledge Bases](https://docs.servicenow.com/bundle/xanadu-service-management-framework/page/product/knowledge-management/concept/user-criteria.html)