# Native Capabilities Evaluated

*Which native Salesforce Experience Cloud capabilities were weighed before building [Relationship-Derived Access](README.md), and where each one stops for a multi-tier partner network with competing, scoped, time-bound relationships.*

This file exists for one reason: to show the work. Building a custom pattern only makes sense if the native platform does not already do the job, and the platform in this space has moved a lot. Salesforce has progressively shifted from "a portal user belongs to one account" toward identity, plus a relationship graph, plus derived access, plus an active account context. That is the same shape as the pattern in this repo. The platform is converging on this model. It has not yet arrived at the specific combination a competing multi-tier dealer network needs, and that gap is what the pattern fills.

Everything below is stated fairly. Each capability is real and useful for its intended job. The "where it stops" lines are about fit for this specific problem, not criticism of the feature.

The naming history alone signals the pace of change: the product launched as Communities in 2013, became Community Cloud, and was renamed Experience Cloud in February 2021. What follows is the current platform, not the old one.

---

## Account Contact Relationship (ACR)

**What it is.** A standard junction object (AccountContactRelation) that relates one Contact to several Accounts without duplicate records. Introduced around 2016 as "Contacts to Multiple Accounts."

**Where it helps.** A person affiliated with more than one company: an executive on several boards, a contractor working across firms.

**Where it stops for this problem.** It models the person across companies. It does not model one dealer company's relationship with two distributor companies, which is account-to-account, not person-to-account. It also carries no business scope or effective dates, and it does not by itself drive record visibility. It is an affiliation record, not an access model.

**Reference.** https://help.salesforce.com/s/articleView?id=000381518&language=en_US&type=1

---

## Account Relationships and Account Relationship Data Sharing Rules

**What it is.** Native partner-to-partner sharing. An account relationship connects a sharing account, an accessing account, and a relationship type; a data sharing rule defines what is shared and the access granted. Matured around 2019 to 2020.

**Where it helps.** This is the closest native fit. It genuinely connects two accounts and shares records between them by relationship type. If the requirements had been simpler, this is where the evaluation would have stopped. In fact, for a separate portal with simpler sharing needs, this is exactly what was used, and it fit well. The custom pattern in this repo was reserved for the harder case, where brand, territory, and effective-date scope exceeded what the native rules could carry.

**Where it stops for this problem.** The rules are defined per relationship-type and object-type pairing, with limits on how many you can create, and they do not carry brand, territory, and effective dates as first-class scope that varies per relationship. Relationships that start, end, and overlap competitively need more dimensions than the rule model exposes.

**Reference.** https://help.salesforce.com/s/articleView?id=platform.networks_partner_account_relationships_and_sharing.htm&language=en_US

---

## Sharing Sets and Share Groups

**What it is.** Declarative sharing for Experience Cloud site users. A sharing set grants access to records associated with a user's account or contact by matching a lookup, and it can span related accounts through ACR. Sharing sets were extended to Customer Community Plus and Partner Community licenses in Winter '19.

**Where it helps.** Low maintenance, no groups to keep in sync, and a strong fit for the common case: "let a user see the records tied to their own account."

**Where it stops for this problem.** The match is anchored to the user's account affiliation, not to a chosen active relationship. It fits "records tied to my account," not one person acting across two competing relationships with a different scoped view in each. Share groups, which extend this outward, are not available to Customer Community Plus and Partner Community licenses.

**Reference.** https://help.salesforce.com/s/articleView?id=platform.networks_setting_light_users.htm&language=en_US&type=5

---

## External Managed Accounts and the Account Switcher

**What it is.** Introduced in Winter '21. An external managed account designates a managing user, a target account they can manage, and the access they are authorized to on it. The Account Switcher component then lets that user switch to their target accounts from the profile menu. This is the native feature closest to the "dealer picker" idea.

**Where it helps.** Its intended jobs, and it does them well: a dealer-group owner switching among their own dealerships, buy-on-behalf-of, and delegated admins who manage users across accounts.

**Where it stops for this problem.** It is built for accounts a user owns or manages, where having access to all of them is the entire point. The hard case here is the opposite: one person acting for two competing dealers who must stay isolated from each other. The switch also changes the active account without scoping access to the specific relationship's brand, territory, and effective dates, and it does not derive a per-relationship subset. In practice you can even use the Account Switcher as the picker interface. What it does not provide is the relationship-derived scoping and the competitive isolation underneath, which is the part the pattern adds.

**Reference.** https://help.salesforce.com/s/articleView?id=platform.networks_external_managed_accounts.htm&language=en_US&type=5

---

## Experience Cloud licenses

**What it is.** The external license type determines which sharing mechanisms even exist. High-volume Customer Community leans on sharing sets and has no roles. Customer Community Plus and Partner Community carry a role hierarchy, sharing rules, and Apex managed sharing.

**Why it is in scope.** It is not a native feature to adopt or reject; it is the constraint that caps everything above. It is decided first, because a license that cannot express the needed sharing turns a later gap into a rebuild rather than a change.

**Reference.** https://help.salesforce.com/s/articleView?id=experience.exp_cloud_plan_licenses.htm&language=en_US&type=5

---

## The platform is still moving

This evaluation reflects the current platform, and the platform keeps changing under all of us. In 2026 alone, API version 67 changed Apex sharing and DML defaults, and new integrations now register as External Client Apps rather than Connected Apps. Those are tracked in [REFERENCES.md](REFERENCES.md) and in the session itself. The point is that the direction of travel, identity to relationship graph to derived access to active context, keeps moving toward the model in this repo.

---

## What is still not native: the seam

Every capability above is real, and several are close. What no single one gives you today, and what they do not compose into cleanly, is the full combination this problem needs:

- Overlapping, competing relationships between the same two accounts.
- Business scope on each relationship: brand, territory, product family.
- Effective dates, so a relationship starts and ends and access follows.
- Visibility derived from the relationship, recalculated when it changes.
- An active-relationship context that keeps two competing relationships isolated from each other, not merged.

The platform is converging on this shape. When it fully arrives, the right move is to retire the custom pieces and use the native version. Until then, this repo documents the pattern that composes those five requirements, built on the current platform, with each native capability evaluated and credited above.

*Names and tiers used here are illustrative. The evaluation holds wherever a person participates in more than one scoped, time-bound, competing relationship.*
