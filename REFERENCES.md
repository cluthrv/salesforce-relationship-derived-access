# References

Salesforce guidance this pattern builds on. Relationship-Derived Access applies that guidance to relationship models that do not fit cleanly into one account hierarchy. It is not a critique of the platform.

Links were checked in August 2026. Salesforce documentation moves; if a link is stale, the title is the reliable search term.

---

## Record access and sharing

**[Record-Level Access: Under the Hood](https://developer.salesforce.com/docs/atlas.en-us.salesforce_record_access_under_the_hood.meta/salesforce_record_access_under_the_hood/uth_intro.htm)**
How sharing rows, access grants and group maintenance tables actually work. Salesforce states this paper is intended for expert architects working on implementations with complex record access requirements. It is the single most useful document here if you are building a sharing engine, and most people in the ecosystem have never opened it.
[PDF version](https://resources.docs.salesforce.com/latest/latest/en-us/sfdc/pdf/salesforce_record_access_under_the_hood.pdf)

**Designing Record Access for Enterprise Scale**
Recalculation cost, group membership churn, ownership skew and large-scale realignments. Directly relevant to the scale considerations in Pattern 2, particularly the warning that seemingly simple group membership changes can trigger substantial recalculation.
Search: *Salesforce Designing Record Access for Enterprise Scale*

**[Apex Sharing Reasons](https://help.salesforce.com/s/articleView?id=platform.security_apex_sharing_reasons.htm&type=5)**
Custom row causes for programmatic sharing. Confirms the constraint that underpins Pattern 2: sharing reasons are defined per individual custom object, and are not available on standard objects.

**[Secure Apex Classes](https://developer.salesforce.com/docs/platform/lwc/guide/apex-security)**
Sharing declarations, user mode and the API version defaults referenced in the testing sections. Confirms that `WITH SECURITY_ENFORCED` is removed for classes at API 67.0 and later, and that `WITH USER_MODE` should be used instead.

---

## Experience Cloud and partner access

**[Create an Account Relationship](https://help.salesforce.com/s/articleView?id=networks_configure_account_relationship.htm&type=5)**
Account Relationships and Account Relationship Data Sharing Rules: the closest native capability to this pattern. Also see the linked considerations page for the constraints noted in the implementation notes, including the one-rule-per-relationship-type-and-object-type pairing and editability after save.

**[Experience Cloud User Licenses](https://help.salesforce.com/s/articleView?id=customer_portal_users.htm&type=0)**
Licence comparison, including External Apps and Channel Account. The source for the licence decision section in the implementation notes.

**[Delegate External User Administration](https://help.salesforce.com/s/articleView?id=platform.networks_DPUA.htm&type=5)**
Confirms delegated external user administration is available on Partner Community and Customer Community Plus.

**[Considerations for Relating a Contact to Multiple Accounts](https://help.salesforce.com/s/articleView?id=sf.shared_contacts_considerations.htm)**
Account Contact Relationship behaviour and its sharing implications. Useful when explaining why it does not, by itself, solve account-to-account relationship scope.

---

## Architecture and integration

**[Integration Patterns and Practices](https://architect.salesforce.com/docs/architect/fundamentals/guide/integration-patterns.html)**
The six standard integration patterns applied in the integration decision matrix. These are Salesforce's patterns, used here as an applied framework rather than as a contribution of this work.

**[Salesforce Well-Architected](https://architect.salesforce.com/docs/architect/well-architected/guide/overview.html)**
Trusted, easy and adaptable. The framing that the trade-offs in this pattern sit inside, particularly the discipline of stating when a pattern does not apply.

---

## Prior art outside Salesforce

**Relationship-based access control (ReBAC)**
The general authorization model in which permissions derive from stored relationship facts. The term was coined by Carrie E. Gates in 2006. Google's Zanzibar paper (2019) and implementations such as OpenFGA and SpiceDB are the modern lineage.

Relationship-Derived Access sits in this lineage. The distinction is one of mechanism and domain: ReBAC systems typically evaluate a relationship graph at request time, while this pattern materialises access into platform share rows ahead of time, and addresses commercial relationships specifically within Salesforce partner ecosystems.

**Party-role modeling**
Established data modeling practice for representing entities that participate in multiple roles and relationships. Predates this pattern by decades and is the conceptual ancestor of the Relationship Record.
