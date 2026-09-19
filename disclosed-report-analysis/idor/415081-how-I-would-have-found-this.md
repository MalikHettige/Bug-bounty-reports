# PayPal — IDOR to Add Secondary Users in Business Accounts

## Report Information

**Original Report:** https://hackerone.com/reports/415081

**Platform:** HackerOne

**Bounty:** $10,000
**Severity:** Critical

**Vulnerability Type:** IDOR (Insecure Direct Object Reference)

**Tags:** #IDOR #AccessControl #H1 #PayPal #BusinessLogic

---

# Executive Summary

PayPal Business Accounts allow owners to add secondary users (employees) with specific privileges. A researcher discovered that the endpoint responsible for adding secondary users did not verify account ownership — meaning an attacker could add their own account as a secondary user to any PayPal Business account they did not own. The new secondary user would gain login access and be able to perform privileged actions on the victim's business account.

---

# Vulnerability Overview

## Vulnerability Details

PayPal Business Accounts support a multi-user feature where the primary account owner can grant up to 200 secondary users access to the account with specific privileges — view balance, edit profile, contact customer service, etc.

The vulnerability existed in the endpoint that handles adding secondary users. The server accepted a request to add a secondary user to a business account identified by its account ID — but never verified that the requester was the legitimate owner of that account.

Attack flow:
```
1. Attacker has their own PayPal Business account
2. Attacker identifies victim's Business account ID (via OSINT or enumeration)
3. Attacker sends a modified POST request to add themselves as a secondary user
   to the victim's account — replacing their own account ID with victim's ID
4. Server processes the request without ownership verification
5. Attacker receives email invitation to activate secondary user access
6. Attacker now has login access to victim's business account
```

## Root Cause Analysis

Missing authorization check — the server trusted the account ID supplied in the request body without verifying that the authenticated session belonged to the owner of that account.

```
Expected check: does session.user == account.owner?
Actual check:   none
```

This is a textbook IDOR — replacing one identifier with another to access resources you should not control.

---

# Hunter Analysis

## Original Hunter's Approach

The researcher identified the "Manage Users" feature in PayPal Business account settings. By intercepting the request to add a secondary user with Burp Suite, they found the account ID in the POST body. Replacing their own account ID with another business account ID and forwarding the request revealed the server processed it without ownership verification — granting the attacker secondary user access to the target account.

## My Alternative Approach

### Recon

- Map all Business account features that reference account IDs — manage users, permissions, settings
- Identify any endpoint that accepts an account ID as a parameter in the body or URL
- Enumerate account IDs via any public-facing feature (merchant pages, transaction references)

### Testing Strategy

- Create two PayPal Business test accounts (Account A and Account B)
- On Account A, add Account B as secondary user — intercept the request
- Note which parameter contains the account ID
- Repeat the request but replace Account A's ID with a third account ID (Account C) that you do not own
- If server returns success — IDOR confirmed

### Tools

- Burp Suite (intercept and modify request)
- Two test accounts on the same platform

### Payloads / Requests

```http
POST /businessmanage/users/add HTTP/1.1
Host: www.paypal.com
Cookie: session=YOUR_SESSION

account_id=VICTIM_ACCOUNT_ID&user_email=attacker@evil.com&privileges[]=VIEW_BALANCE
```

Replace `account_id` value with any business account ID you don't own.

---

# Impact Analysis

## Technical Impact

- Attacker gains login access to any PayPal Business account
- Can view account balance, transaction history, customer data
- Can perform all actions granted to secondary users
- Up to 200 secondary users can be added per account — persistent access even if one is removed

## Business Impact

- Full financial account access for any business using PayPal
- Exposure of transaction history, customer PII, revenue data
- Attacker could impersonate employees of the target business
- At scale: automated attack against all business accounts = mass data breach
- Severe reputational and regulatory risk for PayPal (GDPR, PCI-DSS)

## Scope and Severity

- Affects all PayPal Business accounts globally
- No victim interaction required
- Low attack complexity — one modified HTTP request
- Severity: Critical — unauthenticated access to financial accounts at scale

---

# Key Lessons and Patterns

## Vulnerability Patterns

- Any endpoint that accepts a resource ID without verifying ownership is an IDOR
- Multi-user / team management features are high-value IDOR targets — they reference account IDs explicitly in requests
- "Add user to account" flows almost always pass the target account ID in the body — always test it

## Future Hunting Rules

- On any platform with team/org management: intercept the "add member" or "invite user" request
- Look for: `account_id`, `org_id`, `business_id`, `owner_id` in POST body
- Replace with ID from a second test account you own — if it works, replace with any ID
- Always test with two accounts you control first — never assume, always verify

---

# Personal Reflection

- What surprised me: The simplicity. One parameter change in one request = access to any PayPal Business account. No chaining required.
- What mistake did the developer make: They implemented the feature assuming only legitimate account owners would use it. They never added server-side verification that the session user owns the target account. Classic trust in the client.
- What did I learn: Multi-user management endpoints are goldmines for IDOR. The account ID almost always has to be passed somewhere in the request — and if the server doesn't verify ownership, you own every account on the platform.
- How will I apply this during hunting: Every time I see a team/org/user management feature, I intercept every request and look for IDs I can swap. This is now a standard step in my IDOR methodology.

---

# References

- Original report: https://hackerone.com/reports/415081
- Related OWASP category: OWASP A01:2021 Broken Access Control
- OWASP IDOR guide: https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/05-Authorization_Testing/04-Testing_for_Insecure_Direct_Object_References
