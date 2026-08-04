# Relationship-Derived Access

**A Salesforce architecture pattern for partner and dealer ecosystems, where record visibility, entitlement and application access are derived from explicit, scoped, time-bound business relationships rather than from identity or account hierarchy alone.**

---

## The problem

Standard account hierarchies work until the network stops being a tree.

A dealer signs with a second distributor, and the two distributors compete. A contractor buys the same part through three dealers at three prices depending on the job site. Two dealers merge, and one legal entity now holds two competing distributor relationships that both predate the acquisition. A salesperson moves between dealers, and their pipeline must not follow them.

None of these are edge cases. They arrive eventually in most multi-tier partner networks, and none of them fit in a parent field.

What teams usually do instead is duplicate the account. Reporting splits, the same person ends up with two logins, orders route to the wrong entity, and a spreadsheet quietly becomes the system of record.

This is a recurring failure mode in enterprise CRM implementations for manufacturers and distributors. It is not specific to one company or one industry.

---

## The pattern in one sentence

Store the business once. Store each commercial relationship as its own record, carrying type, scope and effective dates. Derive sharing, entitlement and application access from that relationship rather than from the account tree.

Or more bluntly: **your account hierarchy is not your relationship model.**

---

## Three patterns

### 1 · The Relationship Record
Model the commercial relationship as a first-class record with type, scope, effective dates and status. One dealer, many relationships, no duplication.
→ [patterns/01-relationship-record.md](patterns/01-relationship-record.md)

### 2 · Relationship-Derived Visibility
Compute record access from relationship state, and recalculate when the relationship starts, changes, expires or is suspended.
→ [patterns/02-relationship-derived-visibility.md](patterns/02-relationship-derived-visibility.md)

### 3 · Relationship Context
Separate four questions that most portals collapse into one: who are you, who are you acting for, what are you allowed to do, and what can you see.
→ [patterns/03-relationship-context.md](patterns/03-relationship-context.md)

---

## When to use it

- Partners work with more than one of your partners, and those partners compete with each other.
- Access has to differ depending on which relationship the user is acting within.
- Relationships start, end and change scope often enough that manual maintenance fails.
- Different partner levels control different things: pricing, ordering, customers.

## When not to use it

- A single-tier network, or partners whose relationships never overlap.
- Access is expressible as a partner's own records plus their parent's records.
- Relationships are few and stable enough for standard sharing rules and occasional administration.
- Access does not vary by brand, territory, product family or effective date.

**Standard Salesforce sharing plus a clean account hierarchy will carry you further than most architects expect.** Skipping this pattern early is a reasonable decision. The mistake is not noticing when you have crossed the line.

---

## What this pattern adds

This is not a new access-control theory, and it is important to be precise about what is and is not novel here.

**Established prior art this builds on:**

- **Relationship-based access control (ReBAC)** as a general authorization model, where permissions derive from stored relationship facts. The term dates to 2006; Google's Zanzibar paper and implementations such as OpenFGA and SpiceDB are the modern lineage.
- **Party-role modeling** as a data modeling practice, long predating Salesforce.
- **Salesforce Apex managed sharing** and Apex sharing reasons, which are documented platform capability.
- **Salesforce's integration patterns**, used here as an applied framework rather than as a contribution.

**What this pattern contributes, specifically:**

1. **Scope-aware uniqueness.** The uniqueness constraint is not on parties and relationship type alone, but on parties, type *and normalized scope*, because the same dealer legitimately holds two relationships with the same distributor for different brands or territories. Keying without scope wrongly rejects valid relationships.

2. **Relationship context named as a distinct access layer.** Identity, authorization and visibility are well understood individually. Naming *relationship context* as a separate layer that most portal architectures omit entirely, and identifying the conflation of authorization with visibility as the resulting failure mode, is the conceptual core of Pattern 3.

3. **The date-expiry gap.** A relationship can expire by date with no DML event. No trigger fires, and access silently persists unless a scheduled sweep retracts it. This is a practical implementation gap that is easy to miss when teams design only around record-change events.

4. **The standard-object row cause constraint and its workaround.** Custom Apex sharing reasons exist only on custom objects. On standard objects the engine cannot identify its own share rows by row cause, so it requires separate ownership tracking before any deletion. This is the practical boundary most descriptions of Apex managed sharing skip.

5. **The synthesis itself, with stated boundaries.** Naming a recurring problem class and specifying a coherent pattern that includes when *not* to apply it.

Developed and documented by Vikas Luthra, based on architecting enterprise Salesforce partner ecosystems. This is the pattern arrived at after repeatedly encountering the same failure modes, not a claim to have invented relationship-based access.

---

## Repository contents

| Path | What it is |
|---|---|
| `patterns/` | The three patterns, in a consistent format |
| `salesforce/implementation-notes.md` | Platform-specific depth: licensing, sharing mechanics, security context |
| `salesforce/integration-decision-matrix.md` | Scenario-to-pattern mapping for partner portal integrations |
| `templates/visibility-matrix.csv` | Persona by record by expected result. The regression suite for isolation |
| `REFERENCES.md` | Salesforce guidance this pattern builds on |

---

## Limitations

- This is documentation of an architecture pattern, not an installable package.
- Code samples are illustrative pseudocode showing the shape of the approach, not production Apex.
- The reference implementation is Salesforce. The patterns themselves are platform-neutral, but no other implementation is documented here.
- Exact implementation depends on object volume, licensing, scope complexity and your data model. Nothing here removes the need for your own design work.
- Examples use dealer and distributor language to keep the pattern concrete. Real implementations may use different entities, object names and relationship types. The pattern is not specific to equipment distribution.
- This is not a replacement for standard Salesforce sharing. It is what you reach for when standard sharing runs out.

---

## License

Documentation is licensed CC BY 4.0. Free to use and adapt with attribution.

---

## Contact

Vikas Luthra · [linkedin.com/in/vikas-luthra-crm](https://www.linkedin.com/in/vikas-luthra-crm/)

If you apply this in your own org, I would genuinely like to know what broke. Open an issue or get in touch.
