# SingleStore — IDOR - Scheduled data leak to other accounts by `projectID`

## Report Information

**Original Report:** HackerOne disclosure page
**Platform:** HackerOne
**Bounty:** Hidden
**Severity:** Medium
**Vulnerability Type:** IDOR / Business Logic
**Tags:** #IDOR #AccessControl #BusinessLogic #H1

---

# Executive Summary

An authenticated user was able to access scheduled job data belonging to other users by changing the `projectID` parameter in the `GetNotebookScheduledPaginatedJobs` API endpoint. The application did not verify ownership or authorization for the referenced project, which exposed sensitive information across accounts.

The issue was medium severity because the attack required a valid project identifier, but it still represented a clear authorization bypass with real data exposure.

---

# Vulnerability Overview

## Vulnerability Details

The vulnerable functionality was an authenticated API endpoint that returned scheduled notebook/job information for a given project. By replacing the `projectID` value with another valid project ID, the attacker could retrieve data that did not belong to their account.

The exposed information included:

* database names
* notebook paths
* scheduling details
* infrastructure-related data

This indicates that the server trusted the client-supplied project reference without verifying whether the requester had access to that project.

---

## Root Cause Analysis

The root cause was a missing server-side authorization check.

Instead of confirming:

* “Does this user own this project?”
* “Is this user allowed to view this project’s scheduled jobs?”

the backend appears to have only checked:

* “Is this request authenticated?”
* “Is the `projectID` format valid?”

That is the classic IDOR pattern: the object is identified, but not protected.

---

# Hunter Analysis

## Original Hunter's Approach

The report suggests the hunter found an authenticated endpoint that returned project-based data and then tested whether the project reference could be swapped. The key idea was likely:

* identify a request that uses an object ID
* change that object ID
* observe whether the response changes across accounts

This is the exact kind of test that often reveals access control flaws in APIs.

---

## My Alternative Approach

### Recon

I would first look for:

* project-based endpoints
* notebook or job listing APIs
* parameters like `projectID`, `workspaceID`, `teamID`, `orgID`, or `accountID`
* requests returning nested or account-scoped data

### Testing Strategy

I would test:

* my own project ID versus another valid project ID
* list endpoints and detail endpoints separately
* whether hidden or predictable IDs can be reused across accounts
* whether the same object can be accessed through multiple routes

### Tools

* Burp Suite for request replay and parameter swapping
* Browser DevTools to map network calls
* a small script to compare responses across IDs

### Payloads / Requests

Important things to inspect:

* request body fields containing object references
* query parameters like `projectID`
* path parameters if the ID is in the URL
* response differences in job lists, metadata, or scheduling details

---

# Impact Analysis

## Technical Impact

An attacker could access scheduled job information from other projects without permission. In some systems, that kind of metadata can help an attacker understand internal infrastructure, data structures, and workflows.

## Business Impact

Possible consequences include:

* unauthorized data exposure
* loss of tenant isolation
* trust damage
* increased risk of follow-up attacks using leaked internal details

## Scope & Severity

The issue affected authenticated users who could guess or obtain valid project IDs.
Attack complexity was low once a valid ID was known.
The presence of sensitive metadata justified a real security impact even though the issue was not a full account takeover.

---

# Key Lessons & Patterns

## Vulnerability Patterns

* Object reference present does not mean access is allowed.
* List endpoints often hide IDORs just as much as detail endpoints.
* Metadata leaks can be important even when the data is not obviously “secret.”
* Authorization must be enforced on the server for every object lookup.

## Future Hunting Rules

* Never trust any account-scoped identifier.
* Test the same request across two different accounts.
* Check list, detail, update, and delete actions separately.
* Treat “harmless” metadata as potentially useful attacker intelligence.

---

# Personal Reflection

This report is useful because it shows that a simple `projectID` swap can still matter a lot when the leaked data gives visibility into internal systems.

What stood out to me is that the bug was not about fancy payloads. It was about a missing ownership check. That is a strong reminder that many high-value IDORs are found by careful request comparison, not by complicated exploitation.

The main lesson is to always ask:
“Who owns this object, and where does the server prove it?”

That question is the heart of good IDOR hunting.

---

# References

* Original report: HackerOne disclosure page
* Related OWASP category: Broken Access Control / IDOR
* Related research/articles: PortSwigger Access Control research, PortSwigger IDOR materials, HackTricks access control notes
