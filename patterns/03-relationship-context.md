# Pattern 3 - Relationship Context

*Part of [Relationship-Derived Access](../README.md)*

---

## Problem

Most portal architectures assume identity answers every question. It does not.

Jane logs in once. She works for both Summit Denver and Summit Boulder. Before the portal can show her an order, a price, an application or a customer record, it has to know which of those two she is acting for right now. Her identity does not tell you that.

## Context and forces

- One person can hold relationships with several partner organizations.
- Creating a second login per relationship is the usual workaround, and it is a security and usability failure.
- Authorization and visibility are routinely conflated, which is why portals accumulate hardcoded exceptions.
- If the mechanism that establishes context is not enforced server-side, it becomes the vulnerability.

## Solution

Separate four questions that most portals collapse into one, and answer each with a different mechanism.

| Question | Answered by | Mechanism |
|---|---|---|
| Who are you? | Identity | SSO, identity provider, JIT provisioning |
| Who are you acting for right now? | Relationship context | The selected relationship, held in server-side session state |
| What are you allowed to do? | Authorization | Permissions, roles, application access |
| Which records can you see? | Visibility | Derived from the active relationship |

**Conflating the last two is where portals rot.** Authorization decides what a user can do; visibility decides what they can see. Building one out of the other produces a system where changing a relationship requires changing profiles.

## Structure

**Identity.** SAML or OIDC with just-in-time provisioning, so larger partners keep their own identity provider and their own leaver process. One login, regardless of how many relationships the person holds.

**Relationship context.** The user selects which organization they are acting for. That selection is held in server-side application state, not client state.

This is an application pattern, not a standard platform feature.

**Authorization.** Application and permission access resolved at runtime from tier, role, brand and the active relationship, rather than baked into profiles. Change the relationship and the application list follows.

**Visibility.** Record access derived from the active relationship, per [Pattern 2](02-relationship-derived-visibility.md).

## The security requirement

**The context picker is not navigation. It is part of the security model.**

Every entry point that returns relationship-sensitive data must resolve the active context through a single service, and that service must re-validate that the user actually holds the relationship they claim to be acting under.

If enforcement lives only in the client, a user can switch context to an organization they have no relationship with. The picker becomes the attack surface.

## Consequences

**You gain** one login per person, context that changes without profile surgery, and a clean separation that makes each layer independently reasonable about.

**You accept** a custom context layer to build and secure, a central service that every relationship-sensitive entry point must go through, and the discipline of never trusting client-supplied context.

## Anti-patterns

**A second login per relationship.** The most common workaround, and it produces credential sprawl, broken audit trails and users who cannot see their own work in one place.

**Context in client state only.** Fast to build, and it means the security boundary is enforced by the browser.

**Authorization derived from visibility.** If application access is inferred from what records a user can see, every access change becomes a sharing change.

**Context established once at login and never rechecked.** Relationships end mid-session.
