# Shopify — Takeover an Account That Doesn't Have a Shopify ID

## Report Information

**Original Report:**  https://hackerone.com/reports/867513

**Platform:** HackerOne

**Bounty:** $22,500 + $1,000 bonus + swag

**Severity:** Critical (9–10)

**Date:** 30-09-2026

**Vulnerability Type:** Business Logic — trust-state bypass leading to account takeover (secondary: broken access control on staff object updates)

**Tags:** #AccountTakeover #BusinessLogic #EmailVerificationBypass #H1

---

# Executive Summary

Shopify's account-merge flow trusted a cached "email verified" flag instead of re-verifying it at the point of use. By creating a store under an attacker-owned email, legitimately verifying that email, then using a separate POS Staff endpoint to silently swap the email to a victim's address, an attacker could pass the merge flow's verification check without the victim ever confirming ownership. This allowed full account takeover of any Shopify user who hadn't yet linked a Shopify ID or enabled 2FA.

---

# Vulnerability Overview

## Vulnerability Details

The chain used three separate surfaces, none individually broken, combined in an order the application didn't anticipate:

1. **Store creation with attacker-controlled email.** In the Partner Dashboard's store-creation form, the shop email field is read-only in the UI. Intercepting the request (or editing `window.RailsData` in the browser console) let the attacker set `business_email` and `user.email` to an address they own.
2. **Legitimate verification of that email.** Because the attacker genuinely owns the email at this point, the confirmation link sent to it is real and clickable. The store's email is now marked "verified" server-side.
3. **Silent email swap via a weaker endpoint.** The POS Staff feature allows updating a staff member's email. Using the attacker's own staff page as a template, they captured the update request via browser inspector and replaced the email parameter with the victim's email, then sent it. This endpoint did not re-trigger email confirmation.
4. **Merge trust.** Refreshing the shop profile page prompted account merging into a Shopify ID. Because the "verified" flag was still set (from step 2, now attached to the victim's email via step 3), no second verification was required. The attacker's dev store was now merged under the victim's identity.
5. **Recovery lockout.** From here, the attacker could change the merged Shopify ID's email to their own, and the victim's only remaining path was a standard forgot-password flow, by then already too late.

## Root Cause Analysis

The account-merge logic checked a cached `verified: true` state rather than re-deriving verification status against the email currently attached to the account at merge time. The POS Staff endpoint was the specific gap: it allowed an authenticated user to change their own email without re-confirming it, an inconsistency with the store-creation flow, which did require confirmation. The merge step never questioned why a "verified" flag existed for an email it had never itself sent a confirmation to.

In short: verification was treated as a persistent property of the account object rather than a property of the specific email/account pairing at the moment it mattered.

---

# Hunter Analysis

## Original Hunter's Approach

The reporter (imgnotfound) worked from a clear hypothesis: any flow that treats "verified" as a durable flag rather than something re-checked per use is worth probing for state confusion. Their process:

1. Used the Partner Dashboard to create a development store, a context deliberately chosen because dev stores commonly lack a linked Shopify ID.
2. Found the store's email field was read-only client-side and bypassed this via browser console / Burp interception.
3. Verified the attacker-owned email through the legitimate flow, establishing a genuine "verified" state.
4. Explored adjacent functionality (POS Staff) specifically looking for another path that could alter the same underlying identity without re-triggering verification.
5. Chained the two together and confirmed the merge flow didn't catch the substitution.
6. Kept exploring after the initial takeover was proven, and found the staff-object manipulation impact (swapping first/last name and email between two staff members) as a related but distinct abuse of the same endpoint.

The reporter also flagged uncertainty appropriately, submitting the report while noting a full existing-Shopify-ID takeover "will require digging deeper," rather than overclaiming impact they hadn't proven.

## My Alternative Approach

### Recon

- Map every flow that sets a persistent trust flag (verified, confirmed, approved, linked) versus every flow that later reads that flag to make a decision.
- Specifically look for secondary/lesser-used endpoints (like POS Staff here) that touch the same underlying object as a primary, well-guarded flow (like account creation).
- Identify any object with multiple update paths, since inconsistent validation between paths is exactly the gap this bug lived in.

### Testing Strategy

- For every read-only or client-disabled field, test whether the server independently re-validates it or just accepts whatever the client submits.
- For every multi-step flow (verify → merge, invite → accept), test triggering the final step using state established via a different, weaker path than the one the flow was designed around.
- Ask specifically: "if I change the value this flag was set for, does anything re-check the flag, or does it just get inherited?"

### Tools

- Burp Suite Proxy/Repeater to intercept and modify read-only or hidden fields.
- Browser DevTools console for direct object/state manipulation before a request is even built.
- Two separate test accounts/browser profiles to cleanly separate "attacker" and "victim" state during testing.

### Payloads / Requests

```idris
# Conceptual chain, not literal endpoints — reconstructed from the report
1. Intercept store-creation POST, override read-only email field:
     business_email=attacker@owned.tld
     user.email=attacker@owned.tld
2. Complete the legitimate verification link sent to attacker@owned.tld
3. Capture the "update own staff email" request via browser inspector,
   replace the email param with victim@target.tld, resend
4. Reload account/profile page — check whether a merge prompt appears
   WITHOUT a fresh verification step for victim@target.tld
```

---

# Impact Analysis

## Technical Impact

- Full account takeover of any Shopify user whose shop had no linked Shopify ID and no 2FA enabled — the attacker ends up in control of the merged identity.
- Ability to create a Shopify ID marked "verified" against an email that was never actually confirmed on that identity.
- Ability to rewrite staff object fields (name, email) between two staff members on a shared shop, letting an attacker intercept an intended ownership/access transfer by redirecting it to themselves.

## Business Impact

- **Direct account and store takeover** — an attacker gains control of a merchant's Shopify identity, with everything that implies: store settings, payment configuration, customer data, order history.
- **Trust and integrity of ownership transfers** — the staff-object manipulation impact means even a shop owner's intentional handoff process (transferring a store to a new staff member) could be silently redirected, which is a serious integrity failure independent of the main takeover path.
- **Reputational and platform-trust risk** — this is exactly the class of bug (identity/account merge logic) that, if exploited before discovery, would be difficult for Shopify to detect after the fact, since each individual step looks like legitimate account activity.

## Scope & Severity

- **Affected users:** any account without a prior Shopify ID merge and without 2FA — Shopify itself confirmed these as the two limiting factors in their bounty comment.
- **Attack complexity:** moderate — requires understanding the relationship between three separate features (store creation, email verification, POS Staff), but no advanced tooling once the chain is understood.
- **Privileges required:** none beyond the ability to create a dev store via the Partner Dashboard, which is broadly available.
- **Severity reasoning:** Critical (9–10) is justified — full account takeover with no privileges required, and a real possibility of chaining into SSO-linked accounts, was explicitly named by Shopify as the scenario this could have escalated into. The two limiting factors (must not have merged already, must not have 2FA) narrow the population but don't reduce the severity of what's possible against that population.

---

# Key Lessons & Patterns

## Vulnerability Patterns

- A "verified" or "confirmed" flag treated as a durable account property instead of being re-derived at the point it's consumed.
- Multiple update paths to the same underlying field (email) with inconsistent validation between them — one enforces confirmation, another doesn't.
- Client-side read-only fields carrying zero actual server-side enforcement.
- Legitimate multi-step flows chained across features never designed to interact (store creation + POS Staff + account merge).

## Future Hunting Rules

- Whenever an app has a "verified" concept, find every place that flag gets set and every place it gets read — inconsistency between the two is the bug.
- Any read-only/disabled client-side field is a hypothesis to test with Burp, not a real control.
- When a feature allows updating a field also used elsewhere as a trust signal (like email), check if that update independently re-triggers whatever validation the primary flow enforces.
- Don't stop at proving the first impact — the reporter's staff-object manipulation finding came from continuing to explore after the main takeover was already confirmed.

---

# Personal Reflection

- **What surprised me?** — This bug used no "exploit" in the traditional sense: no injection, no payload, just legitimate features used in an unintended order.
- **What mistake did the developer make?** — The core issue is treating verification as global state instead of re-checking per use.
- **What did I learn?** — This report paid $22,500 for a chain a single-endpoint scanner would never find.
- **How will I apply this?** — 5 blocks from this Shopify batch (trust-flag lifecycle, multi-path field abuse, read-only field trust, cross-feature chaining, post-impact continuation)

---

# References

- Original report: https://hackerone.com/reports/867513
- OWASP Business Logic Testing category: https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/10-Business_Logic_Testing/README
- OWASP Testing for Account Enumeration and Guessable User Accounts: https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/03-Identity_Management_Testing/04-Testing_for_Account_Enumeration_and_Guessable_User_Account
- PortSwigger Web Security Academy — Authentication vulnerabilities: https://portswigger.net/web-security/authentication
- PortSwigger Web Security Academy — Access control vulnerabilities: https://portswigger.net/web-security/access-control
- PortSwigger Web Security Academy — Business logic vulnerabilities (parent topic): https://portswigger.net/web-security/logic-flaws
- PortSwigger Lab — Excessive trust in client-side controls: https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-excessive-trust-in-client-side-controls
- HTB Academy — API Attacks module (BOLA/BFLA, adjacent authz classes): https://academy.hackthebox.com/course/preview/api-attacks
- HTB Academy — Broken Authentication module: https://academy.hackthebox.com/catalogue
