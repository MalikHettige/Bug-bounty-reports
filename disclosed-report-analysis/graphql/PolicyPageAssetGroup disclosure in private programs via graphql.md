# HackerOne — Disclosing PolicyPageAssetGroup in Private Programs via /graphql

## Report Information

**Original Report:** https://hackerone.com/reports/1618347

**Platform:** HackerOne

**Bounty:** $25,000
**Severity:** Critical (9.3)

**Vulnerability Type:** IDOR / Broken Access Control via GraphQL global object identifier enumeration

**Tags:** #IDOR #GraphQL #AccessControl #H1 #GlobalID

# Executive Summary

HackerOne's GraphQL endpoint exposed a `node` query capable of resolving arbitrary objects by their global ID (`gid://hackerone/...`), including `PolicyPageAssetGroupDocument` objects belonging to **private** bug bounty programs. By enumerating or guessing ID values embedded in the global ID string, an unauthorized, unauthenticated requester could retrieve scope/asset information and program names for programs that were never supposed to be visible to them — a severe confidentiality breach given that private program existence and scope are themselves sensitive, non-public facts on a platform whose core value proposition is controlling exactly that kind of disclosure.

# Vulnerability Overview

## Vulnerability Details

GraphQL's `node(id: ID!)` pattern is a common convention (part of the Relay specification) for fetching any object in a schema by a single global identifier, typically base64-encoded and structured like `gid://<app>/<TypeName>/<internal-id>`. This endpoint allowed querying arbitrary `PolicyPageAssetGroupDocument` nodes via that global ID — **without authorization** (explicitly noted by the hunter) — by constructing or guessing valid ID strings:

```graphql
{
  node(id: "gid://hackerone/PolicyPageAssetGroupsIndex::PolicyPageAssetGroup/3981-41287") {
    ... on PolicyPageAssetGroupDocument {
      id
      name
    }
  }
}
```

The response returned the `name` field (redacted in the disclosed report, but per the summary this extended to program names, scope details, and private report titles in the broader exploitation path) for an asset group belonging to a **private** program — one the requester had no relationship to and no authorization to view.

The internal ID portion of the global ID (`3981-41287`) is very likely either sequential or within a guessable/brute-forceable range, making this a textbook IDOR: object-level authorization was missing on the `node` resolver (or on the underlying type's resolution logic), so possessing or guessing a valid-looking ID was sufficient to retrieve the object — membership/ownership of the parent program was never checked.

## Root Cause Analysis

- Missing object-level authorization on the GraphQL `node` interface resolver — a generic "fetch by global ID" pattern is convenient for developers but dangerous by default, because it's easy to wire up without remembering to layer per-type, per-object authorization checks on top of it.
- The global ID's internal numeric component appears enumerable, meaning the barrier to exploitation wasn't "guess a secret token," it was closer to "guess or iterate a predictable-ish number" — a much lower bar.
- Trusting that an object being technically reachable via the schema implies the caller is authorized to see it — the same core failure pattern as the first report (Team disclosure), but here escalated to Critical severity because the underlying data (private program scope, private report titles) is categorically more sensitive than internal permission-group labels.

# Hunter Analysis

## Original Hunter's Approach

Same hunter as the first report (`haxta4ok00`), again with minimal documented methodology in the report body itself ("Just a recent update gives the results of private programs"). The approach again centers on directly querying HackerOne's own GraphQL endpoint and testing the `node` resolver with a `PolicyPageAssetGroupDocument` inline fragment — strongly suggesting prior familiarity with HackerOne's GraphQL schema (likely built from the earlier 2018 finding) and a habit of periodically re-probing the same endpoint family as the platform evolves, since this report is dated 2022, four years after the first.

## My Alternative Approach

### Recon
- On any platform exposing a `node(id: ID!)` or equivalent global-object-lookup pattern, treat it as a first-class target — it's a single resolver that can potentially touch every object type in the schema, making it disproportionately high-value to test versus auditing every individual query one by one.
- Decode any observed global ID (`gid://...` is typically base64-wrapped) to understand its internal structure — type name, numeric/string ID — before attempting to manipulate it.

### Testing Strategy
- Once a valid `gid://` structure is known for one object (even one you're authorized to see), test whether swapping the embedded ID, holding the type constant, returns objects outside your authorization — the type name and action act as the "verb," the ID as the "target," and only the ID needs to be guessed/iterated.
- Test ID enumeration specifically: if the internal ID appears to be two hyphen-separated numbers (as seen here, `3981-41287`), test nearby values and check whether responses differ meaningfully between "object exists but I can't see it" (ideal, secure behavior) vs. "object exists and here's its data regardless of my authorization" (the actual vulnerable behavior).
- Chain this against every type reachable via the `node` interface, not just the one type the original finding focused on — if one type lacks object-level authorization, siblings sharing the same resolver pattern are worth checking too.

### Tools
- Burp Suite (Repeater + Intruder for ID enumeration once a vulnerable pattern is confirmed)
- A GraphQL schema explorer (InQL or manual introspection) to enumerate every type implementing the `Node` interface, to scope the full blast radius

### Payloads / Requests
```graphql
{
  node(id: "gid://hackerone/PolicyPageAssetGroupsIndex::PolicyPageAssetGroup/{id}") {
    ... on PolicyPageAssetGroupDocument { id name }
  }
}
```
Iterate `{id}` across a plausible numeric range once the base pattern is confirmed to leak data for at least one unauthorized object.

# Impact Analysis

## Technical Impact
- Full read access to private program scope/asset data and private report titles via a single, unauthenticated GraphQL query, by iterating a predictable ID pattern.
- Potential to enumerate the entire population of private programs on the platform, not just one.

## Business Impact
- Directly undermines HackerOne's core product guarantee — that a "private" program's existence, scope, and report titles stay confidential to authorized parties only. This is a platform-level trust failure, not a peripheral feature bug.
- Could expose which companies run private bug bounty programs at all (itself sensitive competitive/security-posture information for many organizations), plus specifics of what's in scope and what vulnerability classes have been reported internally (via leaked report titles).
- The $25,000 bounty and 9.3 Critical severity reflect this being close to a worst-case confidentiality failure for a platform whose entire business model depends on access control being airtight.

## Scope & Severity
- Unauthenticated, trivially reproducible once the pattern is known — lowest possible attack complexity.
- Affected potentially every private program on the platform, limited only by how far the ID range could be enumerated.
- Severity justified at Critical: no authentication required, broad scope, and the data category (private program existence + scope + report titles) is maximally sensitive for this specific platform's threat model.

# Key Lessons & Patterns

## Vulnerability Patterns
- Pattern 1: Generic "fetch any object by global ID" resolvers (Relay-style `node` interface) are a single point of failure for authorization across an entire schema — compromising discipline here cascades to every type that opts into the interface.
- Pattern 2: An identifier that looks "random enough" to a developer (base64-wrapped, structured string) can still have a predictable/enumerable internal ID — encoding is not the same as unpredictability.
- Pattern 3: The same hunter returned to the same target's GraphQL surface four years after their first finding and found a new, more severe variant of the same root category (authorization missing on object resolution) — platforms evolve, and previously-patched areas are worth periodic re-testing as new object types and resolvers get added.

## Future Hunting Rules
- On any GraphQL API, specifically search the schema for a generic node/object-lookup resolver (`node`, `entity`, or similar) early in recon — it's a disproportionately high-leverage target compared to auditing every named query individually.
- Always decode and inspect the structure of opaque-looking global IDs before assuming they're unguessable — test for sequential or low-entropy internal components.
- When one object type under a shared resolver pattern shows missing authorization, systematically test every other type reachable through the same resolver rather than stopping at the first confirmed instance — severity scales with how many types are affected, not just the first one found.
- Revisit previously-tested targets periodically, especially ones where you've found a bug before — new schema additions often repeat old architectural mistakes.

# Personal Reflection

This is the same hunter, same target, same root failure category as the first report — just a more severe variant found four years later. That's a useful signal about how bug bounty hunting actually compounds: deep familiarity with one target's architecture (in this case, HackerOne's own GraphQL schema and its `node`-style resolvers) pays off repeatedly, not just once. The $25,000 payout versus the earlier $2,500 also makes the severity math concrete — the technique (ask for data outside your authorization via a GraphQL object resolver) is nearly identical between both reports, but the *sensitivity of the underlying data* is what moved this from Medium to Critical. That's a direct, real-world confirmation of something worth internalizing: the hard part often isn't finding the authorization gap, it's correctly assessing what's actually reachable through it.

# References

- Original report: https://hackerone.com/reports/1618347
- Related OWASP category: API1:2023 — Broken Object Level Authorization (OWASP API Security Top 10)
- Related research/articles: Relay GraphQL Global Object Identification specification (the `node` interface pattern)
