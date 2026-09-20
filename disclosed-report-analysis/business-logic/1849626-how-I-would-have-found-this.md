# Stripe — Fee discounts can be redeemed many times, resulting in unlimited fee-free transactions

## Report Information

**Original Report:** [hackerone.com/reports/1849626](https://hackerone.com/reports/1849626)

**Platform:** HackerOne

**Bounty:** $5,000
**Severity:** Medium

**Vulnerability Type:** Business Logic Error (Race Condition)

**Tags:** #BusinessLogic #RaceCondition #TurboIntruder #Stripe

---

# Executive Summary

Stripe support offered the reporter (@ian), a real Stripe customer, a one-time $20,000 fee discount on transaction processing fees. When accepting the offer through the dashboard, the acceptance request hit a backend endpoint (`/ajax/accept_fee_discount_offer`) that had no protection against concurrent duplicate calls. By firing 30 simultaneous requests at the endpoint with Burp Suite's Turbo Intruder, the reporter got the same $20,000 discount applied 30 separate times to his account — turning a single $20,000 offer into $600,000 worth of fee-free transaction capacity. This is a textbook race-condition-driven business logic flaw: the offer was single-use by intent, but nothing enforced single-use at the point of redemption.

---

# Vulnerability Overview

## Vulnerability Details

- **Where the issue existed:** the `/ajax/accept_fee_discount_offer` endpoint in Stripe's dashboard backend, responsible for applying a support-granted fee discount to a customer's account.
- **The vulnerable functionality:** accepting a one-time promotional fee discount offer shown in the dashboard after Stripe Support applies it to an account.
- **The attack flow:**
  1. Stripe Support manually applies a $20,000 fee discount offer to the reporter's account.
  2. The dashboard shows a prompt/button to "accept" the offer, which calls the accept endpoint once under normal use.
  3. The reporter captured this request in Burp Suite and replayed it 30 times in parallel using the Turbo Intruder extension (built for exploiting race conditions by minimizing network jitter between requests).
  4. Each of the 30 concurrent requests independently passed whatever validation existed and successfully applied the discount, stacking $20,000 × 30 = $600,000 in fee-free transaction credit onto one account.
- **Required conditions:** an account with a pending, unaccepted fee discount offer, and the ability to send many parallel requests to the accept endpoint before the backend's state update from the first request was committed/checked by the others.

## Root Cause Analysis

This is a classic **Time-of-Check to Time-of-Use (TOCTOU)** race condition combined with a **missing idempotency/single-use guarantee**:

- The endpoint likely checked "has this offer been accepted?" and then performed "apply discount" as two separate, non-atomic steps.
- Under concurrent requests, many threads could pass the "not yet accepted" check before any of them completed the "mark as accepted" write — a classic check-then-act race.
- There was no database-level uniqueness constraint, row locking, or atomic compare-and-swap on the offer's redemption state to prevent it from being applied more than once.
- Stripe's first fix attempt added "a check," but it evidently didn't close the race window itself (e.g., it may have checked state without locking the row or using an atomic transaction), so the reporter demonstrated the race was still exploitable afterward. Only a second iteration — presumably using proper locking/idempotency — fully closed it.

---

# Hunter Analysis

## Original Hunter's Approach

- The reporter was a genuine Stripe customer testing on their own live account rather than a synthetic/sandbox target — this is explicitly unusual for a bug bounty submission and only possible because a real discount offer had genuinely been extended to them.
- Recon here was passive/incidental: they simply used the product as offered (accept a discount) and treated the "accept" action as a candidate for a race condition test, likely from experience recognizing that one-time-action buttons on financial platforms are a classic red flag for missing idempotency.
- Testing methodology: captured the single legitimate "accept" request in Burp, then used **Turbo Intruder** (a Burp extension purpose-built for high-speed, single-packet-style race condition attacks) to fire ~30 parallel duplicate requests.
- Tools used: Burp Suite Pro, Turbo Intruder.
- Interesting observation: Stripe's first remediation attempt was insufficient — the hunter didn't just accept the first "fixed" state, they re-tested the same primitive against the patch and found the race window still existed, forcing a second fix iteration. That persistence is the most valuable part of this report.

## My Alternative Approach

### Recon
- On any platform with discounts, credits, promo codes, or one-time claim/redeem/accept actions, I'd specifically enumerate every "accept," "claim," "redeem," "apply," or "activate" button/endpoint — these are prime race condition candidates because they represent a state transition from "unclaimed" to "claimed" that must be atomic.
- I'd check whether the action is reachable multiple times from the UI (e.g., is the button disabled after one click, or can it be spammed / does refreshing let you click it again?).

### Testing Strategy
- Assume every single-use action is a race condition until proven otherwise. Test each with 20-50+ parallel requests via Turbo Intruder's `race` engine (single-packet attack) to minimize jitter and maximize overlap probability.
- Also test with slight variations: duplicate requests with identical vs. slightly varied payloads/timestamps, and test across different sessions/tabs of the same account to rule out session-level locking as a false sense of protection.
- Check whether the fix (if publicly known/patched) added proper locking by re-testing the exact same technique post-patch, exactly as the original hunter did.

### Tools
- Burp Suite Pro + Turbo Intruder (race condition attack template)
- Browser DevTools to confirm client-side there's no hidden debounce/disable that only masks (not prevents) the server-side issue

### Payloads / Requests
- Core technique: capture the single "accept offer" POST request, send it to Turbo Intruder, use the `race-single-packet-attack` template, set concurrent connections to 20-30+, and fire them near-simultaneously so the backend receives them within the same processing window.

---

# Impact Analysis

## Technical Impact
- Arbitrary multiplication of a one-time discount offer — an attacker-controlled number of applications limited only by how many parallel requests they could realistically fire before the race window closed.
- No unauthorized data access or account takeover was involved; the impact is entirely in the domain of business logic/financial abuse.

## Business Impact
- **Financial Loss:** direct cost to Stripe — the reporter calculated ~3% of each $20,000 discount (~$600) as Stripe's real cost per redemption, meaning 30x redemption represented roughly $18,000 in actual processing-fee revenue Stripe would have foregone, from a single offer.
- At scale (multiple customers offered discounts, or larger discount amounts), this same flaw could have represented a much larger, repeatable revenue leak — a systemic version of coupon/promo abuse but against negotiated enterprise-style discounts.
- **Reputational/Trust risk:** since this touches billing and fee logic directly for a payments company, any public exploitation (versus responsible disclosure) could have raised real questions about Stripe's billing integrity.

## Scope & Severity
- **Affected users/scope:** any Stripe customer account with a pending, Support-granted fee discount offer — likely a relatively small population at any given time (this isn't a self-serve public promo code), which is probably why HackerOne/Stripe rated it **Medium** rather than Critical/High despite the six-figure demonstrated impact.
- **Required privileges:** none beyond being a legitimate, authenticated Stripe customer account that had already been granted a discount offer — no privilege escalation needed.
- **Attack complexity:** low-to-moderate — requires knowledge of race condition exploitation techniques and tooling (Turbo Intruder), but no advanced reverse engineering or chained exploits.
- **Severity reasoning:** capped by the narrow precondition (must already have a discount offer extended by Support), but the demonstrated financial multiplier (30x) and the fact that Stripe needed two remediation attempts both argue for it landing solidly at Medium rather than Low.

---

# Key Lessons & Patterns

## Vulnerability Patterns
- Pattern 1: **Any "accept/claim/redeem" action tied to a single-use state is a race condition candidate by default.**
- Pattern 2: **A single added "check" is not a fix for a race condition** — without atomicity (locking, DB constraints, compare-and-swap), a check-then-act sequence is still exploitable under concurrency; this needs re-testing after every "fix."
- Pattern 3: **Business logic flaws don't need to be flashy (no XSS/SQLi/RCE) to be high-value** — a pure logic/timing flaw here was worth $5,000 and demonstrated $600,000 of impact.

## Future Hunting Rules
- Whenever I find any one-time claim/accept/redeem/apply button on a target, immediately test it with Turbo Intruder's race template before moving on — this should be a standard checklist item, not an afterthought.
- When a program marks something as "fixed," always re-run the exact original technique against the patch rather than trusting the fix — partial fixes (adding a check without atomicity) are common.
- Financial/fintech targets are worth prioritizing for business logic testing specifically because the "reward" for a successful business logic bug (monetary abuse) is directly quantifiable and tends to be taken seriously even at "Medium" severity.

---

# Personal Reflection

- What surprised me: that Stripe's first fix attempt still had the race condition — shows that even a well-resourced payments company can ship a check that looks like a fix but isn't atomic.
- What mistake did the developer make: treating "add a check" as equivalent to "add a lock" — the classic TOCTOU trap. The fix needed a database-level atomic operation (row lock / unique constraint / compare-and-swap), not just an extra conditional in application code.
- What did I learn: single-use UI actions on any platform (not just fintech) are a near-default candidate for race condition testing, and Turbo Intruder's single-packet attack is the right tool to reach for immediately.
- How I'll apply this during hunting: build a personal checklist of "claim/accept/apply/redeem"-style endpoints per target and race-test each one early in recon, rather than treating race conditions as a niche technique to try only after other avenues are exhausted.

---

# References

- Original report: [hackerone.com/reports/1849626](https://hackerone.com/reports/1849626)
- HackerOne writeup: [How a Business Logic Vulnerability Led to Unlimited Discount Redemption](https://www.hackerone.com/blog/how-business-logic-vulnerability-led-unlimited-discount-redemption)
- Related OWASP category: OWASP API Security Top 10 — API6/API4: Unrestricted Access to Business Flows / Broken Function Level Authorization (business logic abuse); also relates to CWE-362 (Race Condition) and CWE-841 (Improper Enforcement of Behavioral Workflow)
- Related research/articles: PortSwigger's Turbo Intruder documentation and race condition methodology
