# Pattern 2 - Relationship-Derived Visibility

*Part of [Relationship-Derived Access](../README.md)*

---

## Problem

Competitive separation cannot be expressed as a static rule, because a rule does not know which relationship a user is acting within. Distributor North and Distributor West both have legitimate access to Summit's data, but to different slices of it, and the boundary between those slices is the relationship itself.

Standard sharing can grant access. It does not record the business relationship that explains why the access was granted. That matters most when the relationship ends and you need to remove exactly the access it justified.

## Context and forces

- Three things must hold simultaneously. A distributor rolls up across the dealers it serves. Sibling dealers must not see each other. Authority differs inside a single legal entity.
- Any two of those are straightforward. All three together is where a configuration-only design runs out.
- Relationships change: they start, expire, get suspended, and their scope shifts.
- Some of those changes happen without anyone touching a record.

## Solution

Derive record access from relationship state. When a relationship changes, recalculate the access that relationship justified, and apply only the difference.

## Structure

### Recalculate when

- A relationship starts
- A relationship ends or is suspended
- Brand, territory or product scope changes
- A user moves between dealers
- A scheduled sweep finds relationships that expired by date

### What the engine does

- Recomputes who should see what, from current relationship state
- Adds only the access that is missing
- Removes only the access this engine granted
- Runs asynchronously: Queueable for ordinary work, Batch when volume requires chunking
- Retries failures rather than dropping them

### What you monitor

- Failed recalculations and retry backlog
- Orphan shares: a share row exists but the relationship that justified it does not
- Time between relationship expiry and access removal

## The case people miss

**A relationship can expire by date with no DML event.** Nobody edits a record, so no trigger fires, and the access simply persists.

A scheduled sweep is therefore a required component, not an optimization. This is usually discovered by an auditor rather than by a developer.

## Consequences

**You gain** access that follows commercial reality, a defensible answer to "why does this user see this record," and access that retracts when the relationship does.

**You accept** that you now own recalculation. Budget for bulk recalculation, failure handling, retry and monitoring from the start, not as a hardening phase afterwards.

## Scale considerations

**Share to groups rather than to individual users** where the same access applies to several people. For each shared record that is one share row per target group rather than one per user. A forty-person dealer does not multiply your share table by forty.

**But group membership is its own scaling problem.** Membership churn triggers recalculation, and a single global group per distributor network trades share row count for lock contention. Scope groups to the business boundary: distributor and territory, dealer and branch. Cap their size.

Share row count and group membership churn are two separate problems. Solving one does not solve the other.

**Row count is a design decision, not an accident.** Decide it before go-live, not after the first slow query.

## Platform constraint worth knowing

Custom Apex sharing reasons are available on **custom objects only**. On standard objects, programmatic shares use one of the supported standard row causes, commonly Manual, which means the engine cannot distinguish its own rows from human-created manual shares by row cause alone.

If your engine writes shares on standard objects, it needs separate ownership tracking, typically a lightweight custom object recording what the engine created, diffed against before any deletion.

Many share fields are also immutable, so a desired-state diff with delete and insert is safer than attempting in-place updates.

## Proving it works

A test that confirms a user can see their own records passes just as happily when isolation is completely broken. **Assert the negative.**

Keep a visibility matrix: persona by record by expected result, versioned alongside the code and run every release. It is both your regression suite and your answer when someone from compliance asks how you know.

Make the security context explicit rather than inheriting it from an API version. Declare `with sharing` and use user-mode data access. For classes at API 67.0 and later the defaults changed, and `WITH SECURITY_ENFORCED` is replaced by `WITH USER_MODE`; earlier versions retain the previous behaviour.

Template: [templates/visibility-matrix.csv](../templates/visibility-matrix.csv)

## Implementation notes

See [salesforce/implementation-notes.md](../salesforce/implementation-notes.md).
