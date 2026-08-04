# Pattern 1 - The Relationship Record

*Part of [Relationship-Derived Access](../README.md)*

---

*This pattern defines the relationship. [Pattern 2](02-relationship-derived-visibility.md) derives visibility from it. [Pattern 3](03-relationship-context.md) determines which relationship a user is acting within.*

---

## Problem

An account can participate in more than one commercial relationship at the same time, and those relationships carry different terms, different scope and different lifespans. The standard account hierarchy provides one parent, so there is nowhere to put the second relationship.

## Context and forces

- The legal hierarchy is real and still useful. Roll-up reporting depends on it.
- Operational authority does not follow legal ownership. A dealer buys from two distributors who compete; a branch orders but headquarters negotiates.
- Relationships have terms. They start, they end, they get suspended, and their scope changes.
- Access decisions depend on the relationship, not just on the entity.
- Duplicating the account is the path of least resistance and the beginning of decay.

## Solution

Store the entity once. Store each relationship it participates in as its own record.

The Account record answers *who this company is*. The Relationship Record answers *who it works with, where that applies, and for how long*.

The relationship record becomes the authoritative source for commercial context: the thing every downstream access, pricing and entitlement decision reads.

## Structure

| What the record captures | Design | Example |
|---|---|---|
| Who is connected | Two Account lookups: the party whose access is governed, and the counterparty | Summit Equipment and Distributor A |
| What the relationship is | Relationship Type | Distributor, Dealer, Service Provider, Parent Branch |
| Where it applies | Access Scope: one or more dimensions | Brand X, Colorado |
| When it applies | Start date, end date, status | Active from January 2026 |

**On Relationship Type.** It describes the nature of the relationship, not the kind of account participating in it. The same account can be a distributor in one relationship and a dealer in another.

Keep type and scope distinct. *Distributor* is a relationship type. *Brand X* is scope within that relationship. If brand appears in both, you will end up with a type explosion: one type per brand, which defeats the purpose of having scope at all.

**On Access Scope.** Deliberately not a single field. One or two dimensions are often fields on the relationship record. When scope becomes highly dimensional, or when a single relationship needs several combinations, it is usually better modeled as child records.

There is no hard threshold. The decision depends on how many dimensions you have, whether they combine, and whether they change independently of the relationship itself. Common dimensions: brand, territory, product family, region, market segment.

## Rules

**1 · Scope-aware uniqueness.** Do not allow overlapping active relationships for the same parties, relationship type *and normalized scope*.

The scope clause is the part that is easy to get wrong. Keying on parties and type alone would wrongly reject Summit holding two valid relationships with the same distributor for different brands. If scope lives in child records or spans several dimensions, this check is Apex or Flow rather than a unique field.

**2 · Expire, never delete.** When a relationship ends, set the end date and status. Do not delete the record. History is what lets you answer who could see what last quarter, and access history is an audit question sooner or later.

**3 · Access decisions read the relationship.** Anything deciding visibility, pricing, entitlement or application access reads this record rather than the account tree.

This rule is deliberately narrow. Reporting and hierarchy navigation still use the account hierarchy, and should. The rule applies to access decisions specifically.

## Consequences

**You gain** one entity record instead of duplicates, one login per person regardless of how many relationships they work across, an auditable history of who had access and why, and a single explicit input for every downstream access decision.

**You accept** a custom object to maintain, validation logic that cannot be a simple unique field, a migration path if you are retrofitting an org that already has duplicate accounts, and the discipline of keeping downstream logic off the parent field.

## When not to use it

If partner relationships never overlap, if access is expressible as an account's own records plus its parent's, or if relationships are stable enough that occasional manual administration works, the standard hierarchy is the better answer. See [when not to use it](../README.md#when-not-to-use-it).

## Implementation notes

See [salesforce/implementation-notes.md](../salesforce/implementation-notes.md) for object design, licensing implications and the relationship between this record and the sharing engine.
