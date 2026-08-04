# Salesforce Implementation Notes

*Platform-specific depth for [Relationship-Derived Access](../README.md). The patterns themselves are platform-neutral; this document is the Salesforce reference implementation.*

**Read [Pattern 1](../patterns/01-relationship-record.md) first.** This document assumes you already understand the pattern and want to know what building it involves.

### What this document does not cover

- Exact object volumes or governor-limit thresholds for your org
- Specific licensing terms or contract pricing
- Public group sizing rules
- Production Apex. Code here is illustrative pseudocode showing shape
- Anything that removes the need for your own performance testing

---

## Before you design sharing: the licence decision

Licence choice caps the sharing model before architecture begins. Decide it after you understand the network shape, not before.

Three questions, in this order:

**Who logs in?** High-volume customers, dealer employees, partner sellers, or occasional users spread across many partner accounts. These are different problems.

**What must they do?** Self-service reads, role-based sharing and reports, working leads and opportunities, or using a custom external experience.

**How do you pay, and what does it cost at three times the users?** Per named user, per login, per member, or per partner account. Growth is the question that most often changes the answer.

### Rough mapping

| If the portal needs | Evaluate |
|---|---|
| High-volume self-service | Customer Community, or External Apps |
| Role-based sharing and reports | Customer Community Plus |
| Sales-channel objects: leads, opportunities, campaigns | Partner Community |
| Many occasional users under partner accounts | Channel Account, priced per partner account rather than per user |
| Custom external experience with heavy custom objects | External Apps SKU, but confirm the underlying licence |

**Caveats that matter.** Customer Community has no external roles, so role-hierarchy-based sharing is not available and a distributor roll-up is not buildable on it. SKU names and licence names do not always match, and an External Apps SKU can include Partner Community licences, so confirm what a contract actually grants. Channel Account carries Partner Community-equivalent permissions but is bought per partner account. Delegated external user administration is supported on both Customer Community Plus and Partner Community. Say Experience Cloud, not Communities.

---

## What the platform already gives you

Evaluate these before building anything custom. Every share you get from configuration is one you never own in code.

| Native capability | Works well when | Becomes difficult when |
|---|---|---|
| **Parent Account** | Access follows one legal or reporting hierarchy | An account participates in more than one relationship |
| **Account Contact Relationship** | One person is connected to several companies | You need account-to-account scope, or relationship-specific sharing |
| **Account Relationships + Data Sharing Rules** | One external account needs selected records from another account | Access also varies by brand, territory, dates, status, or which relationship the user is acting through |
| **Role hierarchy** | Access follows one stable parent-child branch | The same dealer works through several competing relationships |
| **Sharing rules** | Access can be granted by ownership or record criteria | Access depends on a specific active relationship and its date range |
| **Sharing sets** | External users need records connected to their own Account or Contact | Access must come through one particular relationship |
| **Manual sharing** | An occasional exception | It must change automatically as relationships change |

**On Account Relationships specifically.** This is the closest native fit and it deserves honest credit. It connects two accounts and shares selected records between them, driven by relationship type. Its boundaries: sharing rules pair a relationship type with an object type and only one rule can hold that pair, so every relationship of a given type shares identically. It is available to Partner and Customer Community Plus users. And after a relationship is saved, only its name is editable without deleting and recreating it, which is a problem when scope changes.

Note also that `AccountAccountRelation`, used in industry cloud features such as Actionable Relationship Center, is a different capability with different availability. This document means the Experience Cloud Account Relationships feature.

---

## The relationship object

See [Pattern 1](../patterns/01-relationship-record.md) for the design. Salesforce-specific notes:

**Both party references are Account lookups.** Not a hierarchy field, not a record type.

**Scope-aware uniqueness is not a unique field.** If scope spans several dimensions or lives in child records, enforcement is Apex or a Flow-based validation. A simple unique index on parties plus type will reject valid relationships.

**Status and dates are separate.** Status handles suspension; dates handle the contractual window. A relationship can be Active but not yet in effect.

**Consider an audience group reference** if you are sharing to public groups. This is a sharing optimization rather than part of the relationship model, and the group is typically resolved through a controlled mapping rather than a lookup, since custom lookups to `Group` are not available.

---

## The sharing engine

### Mechanism

Apex managed sharing. On custom objects, define an Apex sharing reason so the engine can identify the rows it created:

```apex
// pseudocode: shape, not production Apex
// prefer a group when the same access applies to several users
List<Dealer_Order__Share> want = new List<Dealer_Order__Share>();
for (Dealer_Order__c o : scopedOrders) {
    want.add(new Dealer_Order__Share(
        ParentId      = o.Id,
        UserOrGroupId = groupIdFor(rel),   // resolved, not a lookup
        AccessLevel   = 'Read',
        RowCause      = Schema.Dealer_Order__Share
                          .RowCause.Relationship_Derived__c
    ));
}
// diff against what exists, then apply only the delta
Database.insert(toAdd(want, existing), false);
Database.delete(toRemove(want, existing), false);
```

**The sharing reason protects nothing by itself.** Your filter does. Recalculation must query on that row cause so it only ever removes rows it created.

**Idempotency comes from the diff**, not from the row cause. Computing desired state, comparing, and applying only the delta is what makes a re-run safe. Fear of re-running is what kills these designs in production.

### Standard objects

Custom Apex sharing reasons are available on custom objects only. On `AccountShare`, `OpportunityShare` and similar, programmatic shares use one of the supported standard row causes, commonly Manual, and are therefore indistinguishable from human-created manual shares.

If the engine writes shares on standard objects, it needs its own ownership tracking: a lightweight custom object recording what it created, diffed against before any deletion. This is more work, and it is a good reason to keep relationship-driven records on custom objects where the data model allows.

### Execution

Queueable is often appropriate for ordinary recalculation, and Batch when volume requires chunking. Avoid synchronous trigger paths for anything but the smallest recalculations. Platform Events are a reasonable alternative where you want producers and consumers decoupled.

Many share fields are immutable, so an access level change is a delete plus an insert rather than an update. This makes the desired-state diff the safer approach, and it is why naive delete-all-and-reinsert causes lock contention at scale.

### The scheduled sweep

Non-optional. A relationship with an end date of yesterday expires without anyone editing a record, so no trigger fires and access persists until something looks for it. Schedule a sweep that finds relationships expired by date and retracts the access they justified.

### Scale

**Share to groups** where the same access applies to several users. One share row per shared record per target group rather than one per user.

**Scope groups to the business boundary.** Distributor and territory, dealer and branch. One giant group per distributor network reduces share rows and creates lock contention through membership churn.

**Watch for skew.** Very large parent records and frequent group membership changes both drive recalculation cost.

---

## Relationship context

This layer is an application pattern, not a platform feature.

The active relationship is held in **trusted server-side state**. Every Apex entry point returning relationship-sensitive data resolves it through a single context service, which re-validates that the user actually holds the claimed relationship before anything else runs.

The reference implementation uses session state. Platform Cache, a dedicated session service, an external session store or a signed token are all reasonable alternatives. What matters is that the store is server-controlled and that the claim is re-validated rather than trusted.

If context is enforced only in the LWC, the picker is the vulnerability: a user can switch to an organization they have no relationship with.

---

## Testing isolation

**Assert the negative.** A test proving a user sees their own records passes just as happily when isolation is broken.

**Make the security context explicit.** Declare `with sharing` and use user-mode data access rather than relying on API-version defaults. This advice holds regardless of which release you are on.

### API version considerations

*Current as of Summer 2026. Verify against current documentation before relying on it.*

For classes compiled at API 67.0 and later, database operations default to user mode, classes without a sharing declaration default to `with sharing`, and `WITH SECURITY_ENFORCED` is removed in favour of `WITH USER_MODE`. Earlier API versions retain the previous behaviour. See [REFERENCES.md](../REFERENCES.md).

```apex
public with sharing class OrderService {
  public static List<Dealer_Order__c> forCurrentUser() {
    return [SELECT Id FROM Dealer_Order__c
            WITH USER_MODE];
  }
}

System.runAs(dealerB) {
  Assert.isTrue(
    OrderService.forCurrentUser().isEmpty(),
    'Dealer B saw Dealer A records');
}
```

**Isolate portal-user setup** from business-data DML in test setup, or you run into mixed DML restrictions.

**Keep a visibility matrix.** Persona by record by expected result, versioned with the code, run every release. Template at [templates/visibility-matrix.csv](../templates/visibility-matrix.csv).

---

## Delegated administration

You cannot onboard a network of this size centrally, so partners manage their own users inside boundaries you define. This is where isolation is most often lost, because you are handing out access at the edge.

**Approval-routed onboarding.** Self-registration plus an approval workflow, or partner-admin-led provisioning. Enabling self-registration does not give you approval routing; that is a workflow you design.

**Scoped delegation.** A partner administrator manages users only inside their own organization and its relationships.

**A bounded catalog.** Administrators assign from permission sets you control, not anything in the org.

**The leak to design against:** a partner administrator who can assign a permission set granting visibility beyond their own relationships has walked around the entire sharing model.

---

## Integration

See [integration-decision-matrix.md](integration-decision-matrix.md).

Three points where integration and the access model interact:

**Relationship changes are access events**, not ordinary master-data updates. Treat them as higher priority and trigger recalculation on arrival.

**Embedded applications inherit the context problem.** A configurator receiving only user identity cannot price correctly for a dealer working with two distributors. It needs the active relationship, and it should validate rather than trust it.

**Virtualized data still needs a visibility decision.** Leaving invoice history in the ERP does not exempt it from competitive isolation.
