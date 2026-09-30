# Shopify — Email Confirmation Bypass Leading to Full Privilege Escalation via SSO

## Report Information

**Original Report:** https://hackerone.com/reports/791775

**Platform:** HackerOne

**Bounty:** $15,000 + $1,000 bonus + swag

**Severity:** Critical (9–10)

**Date:** 01-10-2026 

**Vulnerability Type:** Business Logic — email verification sent to the wrong recipient, chained into SSO-based account takeover

**Tags:** #AccountTakeover #BusinessLogic #EmailVerificationBypass #SSO #H1

---

# Executive Summary

Shopify's email-change flow generated a confirmation link for a *new* email address but delivered that link to the *old* email address still on file. Because the attacker controlled the old email (they'd just signed up with it), they could confirm ownership of an email they never actually owned — the victim's. Shopify's SSO then let any confirmed email be used to merge and set a shared master password across every store tied to that address, turning a single misdelivered email into full takeover of any shop owner's account and partner account.

---

# Vulnerability Overview

## Vulnerability Details

Attack flow:

1. Attacker signs up for a free Shopify trial using an email they control (`attacker@gmail.com`).
2. Inside the new store's profile settings, before confirming their own signup email, the attacker changes the account email to the victim's address (`victim@target.com`) and saves.
3. Shopify's mailer sends a confirmation link for the *new* email (`victim@target.com`) — but sends it to the *old* email still on file (`attacker@gmail.com`), which the attacker controls.
4. The attacker clicks the link. The system applies it as confirmation that `victim@target.com` is now verified and owned by this account, even though the attacker never received anything at that address.
5. With an arbitrary confirmed email now attached to an attacker-controlled account, the attacker opens their profile. Shopify's SSO detects other stores/partner accounts sharing that same email and offers to integrate them.
6. Accepting the integration lets the attacker set a single master password covering every store and the partner account tied to that email — full takeover, no interaction from the actual victim required at any point.

Required conditions: victim's Shopify account had not yet adopted Shopify's single-login (SSO) system at the time of the attack. Accounts already migrated to single login were not affected.

## Root Cause Analysis

Two related but distinct root causes were involved, confirmed by Shopify's own comments, which is why they asked the reporter to split this into two separately-bountied reports:

- **Primary cause:** the mailer used the account's *pre-change* email as the delivery address for a confirmation token that verified a *post-change* email value. The system tracked "who to email" and "what to confirm" from two different points in the state transition instead of keeping them in sync.
- **Secondary cause (found on retest):** even after Shopify's first patch, the flaw was still reachable if the email was changed *before* the initial verification message had gone out at all — a race/ordering issue in when the "email on file" was read versus when the confirmation was generated and addressed.

Both share the same underlying pattern from the Shopify Business Logic family: a value used for one purpose (delivery address) and a value used for another purpose (what's being confirmed) drifted out of sync, and nothing re-validated that they still matched before granting trust.

---

# Hunter Analysis

## Original Hunter's Approach

The reporter (ngalog) worked from a straightforward but sharp hypothesis: "what happens to in-flight verification email if I change the target value mid-flow?" Their process:

1. Signed up for a free trial to get a fresh, low-friction test account.
2. Immediately changed the account email before completing their own signup confirmation — deliberately interrupting the normal sequence.
3. Waited for and inspected the resulting email, rather than assuming its behavior.
4. Noticed the confirmation link had been sent to the wrong address and clicked it to confirm the actual bug, then documented the exact mismatch.
5. Recognized this wasn't the full impact by itself, connected it to SSO/store integration, and demonstrated the arbitrary-email-confirmation primitive could escalate into a full cross-store takeover.
6. After Shopify's first fix, retested rather than assuming it was closed, found the flow still worked under a slightly different ordering (change email *before* the first verification send), and reported that too, transparently, as a distinct root cause.

The persistence after the "fix" is the standout trait here: most hunters would stop at the first patch confirmation. Retesting under a variant sequence is what surfaced the second bounty.

## My Alternative Approach

### Recon

- On any flow involving a pending state change (email, phone, password) with an async confirmation step, identify exactly which value the confirmation token targets versus which value determines where the token is delivered.
- Specifically test what happens when you interrupt the normal sequence — change a value again before the first confirmation for it has resolved.

### Testing Strategy

- Trigger an email change, then immediately trigger a second email change before confirming the first, and check which address receives which token.
- After any patch to this class of bug, deliberately vary timing and ordering (change before vs. after the first message sends) rather than only retesting the original exact sequence.
- Check whether "verified" status is tied to a specific email string, or to an account object generally — the latter is where this bug lived.

### Tools

- Two email inboxes you control (to clearly distinguish "old" vs "new" address behavior).
- Burp Suite to control exact request timing/ordering if the race condition variant needs to be triggered precisely.

### Payloads / Requests

```haskell
# Conceptual sequence, not literal endpoints
1. Sign up with attacker@own.tld (do not confirm yet)
2. PATCH /profile { email: victim@target.tld }
3. Inspect the confirmation email delivered to attacker@own.tld — check: does the link's token target victim@target.tld?
4. Click link — check: does the system mark victim@target.tld as verified on the attacker's account?
5. Visit account/profile page — check: does an SSO/integration prompt appear offering to merge other accounts under that email?
```

# Impact Analysis

## Technical Impact

- Ability to confirm ownership of an arbitrary email address never actually verified by that address's real owner.
- Once confirmed, ability to leverage Shopify's SSO to merge and set a single master password across every store and the partner account associated with that email — a horizontal escalation across an arbitrary number of accounts, not just one.
- The reporter also flagged that impact could extend beyond individual stores into the shared partner account, since partner and store accounts are tied together under the same identity.

## Business Impact

- **Total loss of control over a merchant's business** — store settings, payment configuration, order/customer data, and now the partner account layer above it.
- **Blast radius beyond a single store** — because the SSO merge pulls in every account sharing the email, one successful confirmation could compromise multiple businesses simultaneously if a merchant ran several stores under one email.
- **Silent to the victim** — the entire chain happens without any interaction from the real account owner. They receive no verification prompt, no suspicious-login alert during the attack itself, since the "confirmation" email never even touches their inbox.
- **Mitigating factor Shopify itself named:** only accounts that hadn't yet adopted Shopify's single-login system were affected, and most merchants already had. This is why the bounty landed mid-range for the "privilege escalation to shop owner" category rather than at the top.

## Scope & Severity

- **Affected users:** shop owners who had not yet migrated to Shopify's unified single-login system.
- **Attack complexity:** low — no special tooling, just correctly-timed use of an existing profile feature.
- **Privileges required:** none beyond the ability to sign up for a free trial, which is open to anyone.
- **Severity reasoning:** Critical (9–10) is well justified — arbitrary email confirmation with zero victim interaction, escalating to cross-account, cross-store, partner-account-level takeover via a legitimate platform feature (SSO), is about as close to maximum impact as a logic bug gets. The bounty being "only" mid-range for its category reflects the narrowing population (pre-SSO-migration accounts), not a discount on technical severity.

# Key Lessons & Patterns

## Vulnerability Patterns

- A confirmation token generated for value B, delivered to the address associated with value A (the pre-change state) — a desync between "what's being confirmed" and "where the confirmation goes."
- Interrupting a normal state-change sequence (changing a value again before its own confirmation resolves) as a technique to expose ordering bugs.
- A single confirmed value (email) treated as sufficient authorization to trigger a powerful secondary system (SSO merge) without additional friction.
- Two root causes hiding behind what looked like one bug — the value of retesting after a fix rather than trusting it.

## Future Hunting Rules

- On any async verification flow, test triggering the underlying value-change twice in quick succession and see which recipient gets which token.
- After a vendor patches a reported bug, retest with varied timing/ordering, not just the original PoC sequence.
- Treat any feature that offers to "merge" or "integrate" accounts based on a shared verified attribute (email, phone) as high-value — it's a force multiplier for any upstream verification bug.
- A single confirmed email should not be sufficient, by itself, to authorize merging control over other accounts; look for whether a second factor or explicit victim action is required anywhere in that step.

# Personal Reflection

- SSO ATO stands for Single Sign-On Account Takeover in this context.
- What surprised me is despite the fact that another finding with a separate root cause was found from retest.
- In my future hunting I will always remind myself to test this exact mechanism once I confirm the email of the admin is disclosed which it is in most organizations but worth a confirmation. In this report the horizontal privilege escalation directly turns into a vertical after discovering the fact that the victim/ordinary user happens to be the owner.

# References

- Original report: https://hackerone.com/reports/791775
- OWASP Business Logic Testing category: https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/10-Business_Logic_Testing/README
- OWASP Testing for Account Enumeration and Guessable User Accounts: https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/03-Identity_Management_Testing/04-Testing_for_Account_Enumeration_and_Guessable_User_Account
- PortSwigger Web Security Academy — Authentication vulnerabilities: https://portswigger.net/web-security/authentication
- PortSwigger Web Security Academy — Business logic vulnerabilities (parent topic): https://portswigger.net/web-security/logic-flaws
- PortSwigger Web Security Academy — OAuth authentication (SSO/identity-merge adjacent class): https://portswigger.net/web-security/oauth
- CWE-640, Weak Password Recovery Mechanism for Forgotten Password (adjacent classification for token-misdelivery flaws): https://cwe.mitre.org/data/definitions/640.html
- HTB Academy — Attacking Authentication Mechanisms module (covers JWT, OAuth, SAML attack classes): https://academy.hackthebox.com/catalogue
