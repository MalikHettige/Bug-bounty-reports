# **AWS VDP** — **Unlimited Reuse of Coupon Code Allows Free Shipping on All Orders on [REDACTED]**

## Report Information

**Original Report:** https://hackerone.com/reports/3426839

**Date:** Sep 22, 2026

**Platform:** HackerOne

**Bounty:** Hidden

**Severity:** Low (0.1 ~ 3.9)

**Vulnerability Type:** business logic / missing idempotency **check** (CWE-840, which is cited in References).

**Tags:** #IDOR #AccessControl #H1 #businessLogic

---

# Executive Summary

A researcher reading a previously disclosed AWS VDP report noticed the free-shipping promo code inside it had not been properly redacted before publication. They confirmed the code still worked and that it carried no single-use restriction, allowing it to be applied to an unlimited number of separate orders on the AWS merch store. The finding is notable less for technical sophistication and more for showing how a program's own disclosure process can leak an active secret.

# Vulnerability Overview

- Server validates a coupon on two axes only: does it exist, is it active. No per-use tracking, no redemption count, no expiry enforcement.
- Classic **missing state/idempotency check** — the flaw isn't "coupon exists," it's "nothing records that this coupon has already been spent.”

# Hunter Analysis

## Original Hunter's Approach

- He didn't find this via normal recon. He found it while **reading a different disclosed report** on the same program, and noticed the coupon code in that report's PoC screenshots wasn't redacted. He then tested that live code himself.
- That's a reusable recon technique worth naming as its own pattern: disclosed reports are sometimes lazily redacted, and a leaked secret/token/coupon in someone else's PoC is itself an attack surface.

## My Alternative Approach

I would start by creating a test account, not going to rush into another victim test account until I am certain I’m done with testing single account hunting. 

**Recon**

- Locate the coupon-apply request (e.g. `coupon-code`) and inspect the request line, body, and response for state-tracking fields such as `remaining_uses`, `discount_id`, or `redeemed`
- Note whether any of those fields actually change after the coupon is applied once — presence of a field means nothing if it isn't enforced
- Watch the rest of the flow for a separate request that might mark the coupon as spent (names like `redeem`, `consume`, `finalize`); its absence is itself a signal that no consumption step exists

**Testing Strategy**

- Apply the coupon once, complete the checkout, then reapply the same code on a second order to confirm reuse
- Observe that item price doesn't matter once reuse is unlimited

**Tools** 

- Burp suite (community edition)

**Payloads / Requests** 

Fields to inspect: `remaining_uses`, `discount_id`, `redeemed`. 

Suspect endpoint naming: `redeem`, `consume`, `finalize`.

---

# Impact Analysis

#### Scope & Severity

- H1 triager scored it Medium (5.6→5.0) initially.
- ~3 months later, AWS closed it as Resolved but **reclassified**: the merch store (was mentioned in the disclosed report) is a **third-party-owned asset**, not AWS infrastructure, so it doesn't meet AWS's own severity bar. Severity dropped to Low, scope changed to `None` retroactively.
- Hunter pushed back once ("I believe severity should stay Medium per H1 triager") — didn't get it reversed.

> AWS's position: 3rd-party-owned = 3rd-party's severity criteria, not theirs.
> 

---

#### Key Lessons & Patterns

- Missing redemption/usage tracking on any discount/coupon/promo mechanism is a recurring business-logic pattern — test reuse before assuming single-use is enforced.
- Reading other disclosed reports isn't just for methodology — check if PoC images/strings are actually redacted. Leaked secrets in someone else's report are fair game.
- Program-owned vs. third-party-owned assets get scored differently even for the same program's VDP — verify asset ownership before assuming your severity estimate will hold; **don't bank a submission on a subdomain/store you haven't confirmed is first-party**.
- Patience required — this took ~3 months and two hunter follow-ups to close.

---

# Personal Reflection

- What surprised me is the fact that the hunter had found the bug on someone else’s disclosed report which led him to hunt and verify the bug.
- The mistake the developer made is failing to enforce that the coupon could only be redeemed once.
- What I learned is that always to read at least 2 disclosed reports of the program I am about to hunt and to always verify if the subdomain of the bug I found is in-scope.
- I will add these to the methodology and remind myself of this report’s severity reduction cause when hunting

# References

- CWE-840 — Business Logic Errors (the general class this falls under; MITRE's own CVE example, CVE-2022-0689 in microweber, is functionally identical: a single-use coupon reusable multiple times) — https://cwe.mitre.org/data/definitions/840.html
- OWASP API Security Top 10 2023, API6 — Unrestricted Access to Sensitive Business Flows (covers exactly this class: a business flow, in this case coupon redemption, with no execution-frequency limit) — https://owasp.org/API-Security/editions/2023/en/0xa6-unrestricted-access-to-sensitive-business-flows/
- PortSwigger Web Security Academy — Business logic vulnerabilities (background theory + labs to replicate the "no server-side enforcement of a stated limit" pattern) — https://portswigger.net/web-security/logic-flaws
