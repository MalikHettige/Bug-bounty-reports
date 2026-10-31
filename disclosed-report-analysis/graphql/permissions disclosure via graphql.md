# HackerOne — Team object in GraphQL discloses team group names and permissions

## Report Information

**Original Report:** https://hackerone.com/reports/343464

**Platform:** HackerOne

**Bounty:** $2,500
**Severity:** Medium (5.0)

**Vulnerability Type:** Information Disclosure / Over-permissive GraphQL field exposure

**Tags:** #GraphQL #InformationDisclosure #BrokenAccessControl #H1

# Executive Summary

A GraphQL `team` query on HackerOne's own platform exposed internal `team_member_groups` data — including group names and internal permission strings — for any team queryable via its `handle`, including private programs. An unauthenticated or low-privileged requester could enumerate program handles and pull back internal role/permission structure never intended to be public, which also indirectly revealed whether a given program in HackerOne's Directory was private.

# Vulnerability Overview

## Vulnerability Details

The `team` query accepted a `handle` parameter (e.g. `"security"`) and returned a `Team` object. That object's schema included a `team_member_groups` field — an array of internal permission groups (`Support`, `Admin`, `Standard`, `Program Management`, etc.) each with an `id`, `name`, and `permissions` array (e.g. `["user_management","program_management"]`). This field was reachable in a single, simple query with no special authorization check gating it:

```graphql
query {
  team(handle:"security") {
    id, name, handle,
    team_member_groups { id, name, permissions }
  }
}
```

Nothing about this request required elevated privileges — it's the same shape of query any authenticated (or possibly unauthenticated, per the report's "without authorization" framing in the related report) user could send. The field simply shouldn't have been resolvable by a general caller at all.

## Root Cause Analysis

- The `Team` type's schema exposed `team_member_groups` as a standard, queryable field with no field-level authorization check distinguishing "I am a member/admin of this team" from "I am any caller who knows the team's handle."
- This is the same class of failure covered in GraphQL training material generally: the schema is shared across the entire API surface, and without explicit per-field authorization, anything in the schema is fetchable by anyone who can reach the endpoint — regardless of whether the application's UI ever requests that field.
- A secondary issue flagged by the hunter in a follow-up comment: this also indirectly let a caller determine whether an arbitrary program in HackerOne's Directory was running privately, since the ability to resolve team data served as an oracle for program visibility status.

# Hunter Analysis

## Original Hunter's Approach

The report itself is sparse on methodology (language barrier noted by the hunter — "sorry i bad speak english") — effectively just the request/response pair with no detailed recon narrative. The researcher appears to have gone directly at GraphQL schema exploration on HackerOne's own platform, likely via introspection or by observing legitimate `team` queries used elsewhere in the app and then adding fields not normally requested by the frontend (a direct parallel to the field-exposure technique practiced against `getUser`/`password` in training labs — ask for more than the UI ever asks for).

## My Alternative Approach

### Recon
- Run standard GraphQL introspection against the target's endpoint (`__schema`, `__type`) to enumerate every type and field on `Team`, rather than guessing field names from observed traffic alone.
- Cross-reference every field on sensitive object types (`Team`, `User`, `Organization`-equivalent types) against what the frontend UI actually requests — any field the UI never touches is a prime candidate for a missing authorization check, since it was likely never exercised in manual QA either.

### Testing Strategy
- For every object type reachable via a public-ish identifier (here, `handle`), test whether privileged sub-fields (anything resembling `*_groups`, `*_permissions`, `*_roles`, `internal_*`) are resolvable without membership/admin context.
- Test the same query both authenticated (as a low-privilege user) and, if feasible, fully unauthenticated — authorization gaps often apply inconsistently across these two states.
- Enumerate known/guessable handles (company names, common program handles) to establish blast radius — one disclosed team is a finding; dozens is a stronger severity argument.

### Tools
- Burp Suite (Repeater for manual query construction, InQL or manual introspection for schema mapping)
- Browser DevTools (to observe which `team` query shapes the real frontend sends, as a baseline to diff against)

### Payloads / Requests
```graphql
query { __type(name:"Team") { fields { name type { name } } } }
```
Then selectively request any flagged sensitive field against multiple team handles to confirm scope.

# Impact Analysis

## Technical Impact
- Disclosure of internal permission-group structure (group names, associated permission strings) for any team reachable by handle, including private programs.
- Indirect oracle for private-vs-public program status across HackerOne's Directory.

## Business Impact
- Exposes HackerOne's internal RBAC (role-based access control) model to external parties — while the group *names* themselves aren't catastrophic, mapping an organization's internal permission structure is reconnaissance value for a more targeted attack (e.g., knowing exactly which permission string controls `user_management` narrows follow-on privilege-escalation attempts).
- Confirmed in the report: usable to infer which programs in the Directory are running privately — a meaningful confidentiality leak for programs that specifically chose not to disclose their existence.

## Scope & Severity
- Affected virtually all programs except fully private ones explicitly excluded from Directory visibility — the hunter noted the issue reproduced across public, limited-visibility, and some private programs.
- No special privileges required to exploit — the severity (Medium, 5.0) reflects that the data itself wasn't highly sensitive (group names/permission labels, not credentials or PII), but the breadth of exposure and the private-program oracle effect justified a non-trivial rating.

# Key Lessons & Patterns

## Vulnerability Patterns
- Pattern 1: GraphQL object types frequently expose more fields than the application's own frontend ever queries — the schema is the real attack surface, not the UI.
- Pattern 2: A field that reveals internal organizational structure (roles, permission labels, group names) doesn't need to be "secret data" like a password to still be a real information-disclosure finding — it's reconnaissance value for an attacker building a bigger picture.
- Pattern 3: A disclosure bug can have secondary effects beyond its literal data — here, the ability to resolve a query at all became an oracle for an unrelated fact (private program existence).

## Future Hunting Rules
- On any GraphQL target, explicitly enumerate every field on identity/organization-adjacent types (`Team`, `User`, `Organization`, `Account`) and test each one independently for field-level authorization, rather than assuming object-level access control covers every sub-field.
- When a query can be parameterized by a guessable/enumerable identifier (like `handle`), always test the blast radius across multiple values before reporting — scope size materially affects severity.
- Treat "does this reveal whether X exists" as its own impact category, separate from "does this reveal X's actual data" — oracle-style leaks are often underreported because they feel indirect.

# Personal Reflection

This is the exact bug class I practiced today — field over-exposure in GraphQL schemas, reachable by asking for fields the frontend never requests. Seeing it land a real $2,500 bounty on HackerOne's own platform (ironic, given they run the program) confirms the lab technique isn't academic — it's a live, recurring pattern across real GraphQL APIs. What stands out: the hunter didn't need a sophisticated exploit chain, just the discipline to actually ask for every field on a type instead of settling for what the obvious query returns. That's a mechanical habit, not a hard technical skill — which means it's realistic to execute consistently once internalized.
# References

- Original report: https://hackerone.com/reports/343464
- Related OWASP category: API3:2023 — Broken Object Property Level Authorization (OWASP API Security Top 10)
- Related research/articles: GraphQL field-level authorization design patterns
