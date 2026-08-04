# Integration Decision Matrix
**A companion resource to Relationship-Derived Access**

For partner and dealer portals built on Salesforce. Use it to choose an integration pattern deliberately rather than by habit.

**Attribution.** The six data-movement patterns are Salesforce's published integration patterns. The application-access grouping and the scenario mapping below are an applied framework, not a Salesforce standard.

---

## Choose on four questions

Before you pick anything, answer these. If you can't, you're not ready to choose.

1. **Where should the data live?** Which system is the source of truth, and is the portal allowed to become one?
2. **How current must it be?** Sub-second, minutes, overnight, or "whenever someone asks"?
3. **Who starts the interaction?** Your platform, the other system, or a user waiting on a screen?
4. **What happens when the other side is down?** Degrade a feature, queue and retry, or block the user?

Question four is the one teams skip, and it's the one that decides whether your portal is available when your ERP isn't.

---

## Part 1 · How data moves

### The six patterns

| Pattern | Use it when | Watch out for |
|---|---|---|
| **Request and Reply** | A user is waiting and needs the answer now | The user experience now depends on the remote system's response time and availability |
| **Fire and Forget** | You must notify another system but need no answer back | Delivery guarantees, retries and reconciliation all have to be designed |
| **Batch Data Synchronization** | High volume, and staleness between runs is acceptable | Reconciliation, and what happens inside the failure window |
| **Remote Call-In** | The external system initiates | Security, throughput and API limits |
| **UI Update Based on Data Changes** | The screen must reflect a change without a refresh | Only worth it for genuine real-time needs |
| **Data Virtualization** | Data must stay in its source system but be visible here | Query performance, and the source being unavailable |

### Scenario mapping

These are common fits, not prescriptions. Every one of them has legitimate exceptions driven by volume, analytics needs, offline access, or what the other system actually supports. Use them as a starting point for the four questions above, not as an answer key.

| Business scenario | Common pattern | Why | Watch out for |
|---|---|---|---|
| Live price for a configured product | Typically Request and Reply | The dealer is on the screen and cannot proceed without it | Set a timeout and a fallback, or the quote flow dies with the pricing engine. Cached pricing with server-side refresh is a legitimate alternative at high volume |
| Inventory or availability check | Request and Reply | Answer is only useful if it is current | Cache briefly for repeat lookups on the same session |
| Credit limit or account status check | Request and Reply | Gates the transaction, so staleness is a commercial risk | Decide explicitly whether an unavailable service blocks or warns |
| Order submission to ERP | Fire and Forget | The dealer should not wait on ERP processing | Guaranteed delivery, retry, and a visible submission status the dealer can trust |
| Order status update back to the portal | Remote Call-In | The ERP knows when status changed; polling wastes calls | API limits, and authentication for a system-to-system caller |
| Nightly price file | Batch Data Synchronization | High volume, and overnight currency is acceptable | Reconciliation, and what a partial failure leaves behind |
| Product master and catalog sync | Batch Data Synchronization | Large, slow-changing, and needed locally for search and selection | Deletions are harder than inserts; decide how retirement propagates |
| Invoice and payment history | Often a good candidate for Data Virtualization | Finance owns it, it is queried infrequently, and copying it means owning the sync forever | Source availability and query performance on wide date ranges. Replication is a reasonable choice where analytics, search or offline access matter |
| Equipment and serial registration | Fire and Forget, or Batch | Manufacturer needs the record but not synchronously | Duplicate submission handling |
| Warranty claim submission | Fire and Forget | Claim processing is asynchronous by nature | The dealer needs a reliable claim reference immediately |
| Warranty claim status | Remote Call-In, or Virtualization | Depends on whether the warranty system can push | If push is unavailable, evaluate virtualization before introducing polling. Virtualization is not always possible: some systems lack a usable query API |
| Dealer hierarchy changes from ERP | Batch, plus targeted Remote Call-In | Bulk reload for accuracy, immediate call-in for a new or terminated relationship | **This one is different.** Relationship changes drive access. Treat them as higher priority than ordinary master data, and trigger recalculation on arrival |
| Document and manual libraries | Virtualization or content links | Large binaries, and no reason to hold them in the platform | Governing visibility without holding the file |
| Live order status on an open screen | UI Update Based on Data Changes | The dealer is watching and expects it to move | Only justify this where someone is genuinely waiting |

---

## A note on event-driven integration

The six patterns above describe the *shape* of an interaction: who initiates it, whether anyone waits, and how much moves at once. Event-driven integration is an *implementation style* that can sit beneath several of them rather than a seventh pattern.

In practice it most often underpins Fire and Forget, Remote Call-In, and UI Update Based on Data Changes. On the Salesforce side the mechanisms are Platform Events, Change Data Capture and the Pub/Sub API.

The architectural difference is topology. Point-to-point looks like this:

```
ERP  →  direct callback  →  consumer
```

Event-driven looks like this:

```
ERP  →  event bus  →  multiple consumers
```

The second is worth the extra infrastructure when more than one system cares about the same fact, when you want producers and consumers to evolve independently, or when replay after an outage matters.

**For this pattern specifically:** relationship changes are a natural event. A relationship starting, ending, changing scope or being suspended is a fact that sharing recalculation, pricing, entitlement and downstream reporting may all need to react to. Publishing it once and letting consumers subscribe scales better than a chain of point-to-point callouts, and it makes the recalculation trigger explicit rather than implicit.

---

## Part 2 · How users reach another application

Different problem from data movement. This is about a person getting into a system, not a record getting between systems.

| Pattern | Use it when | Watch out for |
|---|---|---|
| **SAML or OIDC SSO** | The application owns its own interface and you only need to assert verified identity | Attribute mapping, and what happens when the dealer's IdP is the assertion source |
| **Canvas (signed request or OAuth)** | The application must feel native inside the portal and receive user context | Context passed at launch is a security boundary; validate it on the receiving side |
| **OAuth and token-based** | System-to-system or delegated API access on behalf of a user | Token lifetime, refresh, and revocation when a relationship ends |
| **Content and public links** | There is no session to carry, only visibility to govern | A link that is not access-controlled is public, regardless of where you put it |

### Scenario mapping

| Business scenario | Pattern | Why |
|---|---|---|
| Product configurator | Canvas is a common fit | Must feel native, and needs dealer, brand and territory context to configure correctly. An LWC wrapper over a headless API is a legitimate alternative where you control the tool |
| Selection or sizing tool | Canvas | Same reason: the answer depends on who is asking |
| Warranty administration system | SAML or OIDC SSO | The system owns its own interface; you only need identity |
| Training or LMS | SAML or OIDC SSO | Vendor-hosted, full application, no context needed beyond identity |
| Parts catalog | Depends entirely on ownership | Native if you own it, Canvas if embedded, SSO if the vendor owns the experience. The deciding question is who owns the interface |
| Analytics or reporting portal | SAML SSO, or embedded analytics | Row-level security in the target system must respect the same relationship boundaries |
| Third-party financing or credit application | SAML SSO with attribute assertion | The provider needs to know which dealer is applying, not just which person |
| Manuals, bulletins, approved documents | Content and public links | No session to carry; the only requirement is governed visibility |

---

## Part 3 · Where this connects to Relationship-Derived Access

Three integration decisions are not neutral in a partner portal. They interact directly with the access model.

**Relationship changes are access events, not master-data updates.** When the ERP tells you a dealer-distributor relationship started, ended or changed scope, that is not a nightly-batch problem. Access depends on it. Treat it as a higher-priority feed and trigger recalculation on arrival.

**Embedded applications inherit your context problem.** If a configurator receives only the user identity and not the active relationship, it cannot price or configure correctly for a dealer who works with two distributors. Whatever context the portal resolves, the embedded application needs the same answer, and it needs to validate it rather than trust it.

**Virtualized data still needs a visibility decision.** Leaving invoice history in the ERP does not exempt it from competitive isolation. The external object still has to be filtered by the same relationship boundary, and that filtering has to happen somewhere you control.

---

## Anti-patterns

**Defaulting to one pattern everywhere.** Usually the one the team used last. The cost shows up as either unnecessary real-time coupling or unacceptable staleness.

**Copying data because copying is easier than integrating.** Every copy is a sync you own forever. Virtualization is underused for exactly this reason.

**Treating the portal as a new system of record.** If the portal starts owning customer, pricing or product definitions, integration becomes a permanent reconciliation project.

**Real-time by reflex.** "The business said real time" usually means "the business said they did not want stale data." Those are different requirements with very different costs.

**Polling instead of push.** If the source system can call in, let it. Polling burns API capacity to discover that nothing changed.

---

## How to use this in your own architecture decision record

For each integration, record four lines:

```
Scenario:
Pattern chosen:
Answers to the four questions:
What happens when the other side is down:
```

The fourth line is the one nobody writes and everybody needs eighteen months later.
