# Architecture Decision Record

*The grounding decisions behind [Relationship-Derived Access](../README.md). The session slides surface only three of these. This is the fuller list, kept generic on purpose: object and system names are illustrative, and you map them to your own domain.*

A portal at this scale is not one decision, it is dozens. The eight below are the ones that shaped the model most, the ones that are expensive to reverse once you have built on them. Each is written as context, decision, and consequence, so you can judge whether your situation matches before you copy the choice.

---

## 1. System of record for each core entity

**Context.** A dealer portal touches customers, users, products, and prices, and every one of those already has an authoritative home in some upstream system: CRM, ERP, a product master, a pricing engine. A portal that quietly starts mastering any of them creates a second definition that drifts from the first.

**Decision.** The portal is not the system of record for core business entities. Customer and account data, the person's identity as a business contact, product and parts master, and pricing each stay mastered upstream. The portal reads from them and adds only what is genuinely portal-specific: the relationship records, the sharing, and the portal access itself.

**Consequence.** Integration direction is decided per entity up front (see the integration decision matrix). It also draws a clean line for decision 7: the portal does not own who a person *is*, only what that person can *reach* in the portal. Getting this wrong shows up later as duplicate customers and "which system is right" arguments that no amount of sharing logic can fix.

---

## 2. Experience Cloud license

**Context.** The license is not a billing footnote. It decides which sharing mechanisms even exist for external users. High-volume community licenses have no roles and lean on sharing sets. Partner and Customer Community Plus licenses carry a role hierarchy, sharing rules, and Apex managed sharing. Channel Account prices per partner account rather than per user.

**Decision.** Select the license from the personas and the access model, not from headcount cost alone. For a dealer network that needs relationship-scoped, role-aware, programmatic sharing, that means a license with roles and Apex managed sharing available.

**Consequence.** The license caps what the sharing engine can do, so it is chosen before the model is built, not after. Choosing a high-volume license first and discovering later that the access model needs roles is a rebuild, not a tweak.

---

## 3. Person and account model

**Context.** Before modeling relationships, you decide how external people and companies exist in Salesforce. Person Accounts blur the line between the company and the individual. Business Accounts with related Contacts keep them separate.

**Decision.** Dealers and distributors are business Accounts. People are Contacts enabled as portal users. Person Accounts are not used.

**Consequence.** This is a prerequisite for Pattern 1, not a style choice. The relationship object is a junction between two Accounts, and a membership ties a Contact to one relationship. Both depend on Account and Contact being distinct. Model people as Person Accounts and the junction has nothing clean to join.

---

## 4. Org strategy

**Context.** The portal can live in the same org as the internal CRM, or in a separate org connected by integration. Shared orgs simplify data proximity but widen the blast radius of a sharing mistake. Separate orgs isolate but add integration cost.

**Decision.** [State your choice and the driver. A shared org keeps partner and internal data in one sharing model; a separate org isolates external access at the org boundary. The right answer depends on data volume, security posture, and how much internal data partners must never touch.]

**Consequence.** This decision sets where the competitive-isolation boundary is enforced, at the sharing layer inside one org, or at the org boundary plus sharing. It is close to irreversible once either path has real data on it.

---

## 5. Single site versus per-brand sites

**Context.** A manufacturer with several brands can run one Experience Cloud site with brand as a scope dimension, or one site per brand. One site centralizes maintenance but pushes all brand separation into the access model. Per-brand sites separate cleanly but multiply configuration.

**Decision.** [State your choice. A single site with brand as a relationship scope dimension keeps the model uniform and lets one dealer span brands without a second login. Per-brand sites suit cases where branding, navigation, or governance differ enough that a shared site fights you.]

**Consequence.** With a single site, brand becomes part of the relationship scope and the visibility engine, which is more work in the access layer but one place to maintain. With per-brand sites, isolation is partly structural, at the cost of duplicated setup.

---

## 6. Localization

**Context.** A global dealer network implies multiple currencies, languages, and regional rules. Retrofitting multi-currency or translation after launch is painful.

**Decision.** Decide multi-currency, multi-language, and regional data handling up front, even if the first release is single-region. Enable what cannot be added cheaply later.

**Consequence.** Some of these (multi-currency in particular) are effectively one-way doors in an org. Deciding them late forces a migration rather than a configuration change.

---

## 7. Delegated administration: access, not identity

**Context.** Onboarding every partner user centrally does not scale with the network. Delegating it to dealer admins does, but delegation is where isolation quietly breaks if it is not bounded. This is also the decision most easily read as contradicting decision 1, so it is worth stating precisely.

**Decision.** Dealer admins provision *portal access*, not identity. They enable their own Contacts as portal users, assign from a set of permission sets you approve, and activate or deactivate within their own dealer and its relationships. They do not become the system of record for who a person is. The approved permission sets never carry View All or Modify All, so record access stays governed by relationship-derived sharing rather than by what an admin can assign.

**Consequence.** This is consistent with decision 1: identity is mastered upstream, the portal governs reach. The security of the whole model then depends on the boundary around what a delegated admin can grant, which is why the approved-permission-set constraint is not optional.

---

## 8. Entitlement and the two-tier application grant

**Context.** In a multi-application portal, "which apps can this dealer use" and "which of those can this specific person use" are two different questions. Modeling them as one loses the account boundary. Modeling app access straight onto the person lets someone be granted an app their dealer was never entitled to.

**Decision.** Application access is granted in two tiers. The relationship grants a pool of applications to the dealer (account tier). Each membership grants a subset of that pool to the individual (person tier). A member grant is always a subset of its relationship grant, and that invariant is enforced in code, because the platform will not enforce it.

**Consequence.** A single query resolves a user's cross-application access from their active membership. Two people in the same dealer can differ, and the same person can differ across two dealers, without anyone being able to exceed what the relationship allows. See [data-model.md](data-model.md) for the objects.

---

## How these connect

Decisions 1, 3, and 7 together settle the identity question: entities are mastered upstream, people are Contacts under business Accounts, and dealers provision access rather than identity. Decisions 2, 4, and 5 settle where and how isolation is enforced: the license sets what sharing is possible, the org and site strategy set where the boundary sits. Decisions 6 and 8 are the ones most expensive to add late. None of these is a contribution or a novelty. They are the grounding calls, documented so the pattern is legible and reproducible.

*Names and systems here are illustrative. The shape holds wherever a person participates in more than one scoped, time-bound relationship and receives a subset of what that relationship grants.*
