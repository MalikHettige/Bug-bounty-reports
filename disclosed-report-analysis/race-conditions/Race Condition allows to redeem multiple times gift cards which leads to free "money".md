# `Reverb.com` — **Race Condition allows to redeem multiple times gift cards which leads to free "money"**

## Report Information

**Original Report:** https://hackerone.com/reports/759247

**Platform:** HackerOne 

**Bounty:** [Hidden]

**Severity:** High 

**Vulnerability Type:**  **Race Condition**.

**Tags:** #Race Condition #H1

---

# Executive Summary

The disclosed report that submitted by muon4 to **Reverb.com** was about a vulnerability that allows an attacker to redeem the same gift card multiple times by intercepting a `POST /fi/redeem HTTP/1.1` request and via turbo intruder after redeeming a gift card. The attacker there was able to multiply his gift card balance (Reverb Bucks) from $25 (the price of the gift) to $175. This directly impacts to the company’s trust.

---

# Vulnerability Overview

## Vulnerability Details

After getting authenticated and buying a gift the attacker intercepts the request of gift card redeeming endpoint which was suppose be redeemed once instead of how many times the user wishes for. The Race condition vulnerability probably existed there. And via `turbo intruder` (a burp suite extension) the attacker uses this following python code as a payload and sets the external HTTP header to `x-request: %s` before starting the attack. 

The burp Suite extension Turbo Intruder is built specifically for **firing many requests with precise timing control**, faster and more configurable than Burp's built-in Intruder. For race conditions specifically, its value is the `openGate()` mechanism you saw in the payload — it lets the attacker to **queue up many identical requests and release them all at effectively the same instant**, maximizing the chance they all land inside that narrow check-then-act window simultaneously.

---

## Root Cause Analysis

This is a TOCTOU (time-of-check to time-of-use) vulnerability. The server checks whether the gift card is unredeemed, then redeems it, but these two operations aren't atomic. Concurrent requests can all pass the check before any single one completes the update, allowing multiple redemptions from one card.

---

# Hunter Analysis

## Original Hunter's Approach

This adds a major part for my perspective in business-logics. In my future hunting I would try to intercept the Gift redeeming request, send to repeater and concurrently check if the balance changes to confirm a vulnerability. The turbo intruder will be used and the interesting part is that the python payload below can be used to use in turbo. 

---

## My Alternative Approach

I would extend this beyond coupons/gift cards to any check-then-act endpoint tied to value — refund processing, subscription cancellation, referral-bonus claims, inventory/stock decrements, withdrawal requests. Same TOCTOU shape, different business context.

### Payloads / Requests

```nix
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=30,
                           requestsPerConnection=30,
                           pipeline=False
                           )

   for i in range(30):
	engine.queue(target.req, i)
        engine.queue(target.req, target.baseInput, gate='race1')

    engine.start(timeout=5)
   engine.openGate('race1')

    engine.complete(timeout=60)

def handleResponse(req, interesting):
	table.add(req)
```

---

# Impact Analysis

## Technical Impact

The attacker has the ability to redeem unlimited gift cards = unlimited money. Which means be able to buy any product.

---

## Business Impact

Every exploited gift card represents direct, unrecoverable financial loss — Reverb effectively gives away real products for free. At scale (if automated/shared), this could be exploited repeatedly before detection, since redemption abuse doesn't trigger the same fraud-detection signals as, say, stolen credit card use. Beyond the immediate loss, public disclosure of a "free money" bug specifically damages trust in the platform's payment/rewards integrity — a category of trust that's expensive to rebuild once questioned.

---

## Scope & Severity

Reverb.com's gift card redemption system (`POST /fi/redeem`), affecting any authenticated user account. 

**Severity: High**— no privilege escalation or special access required, directly convertible to real financial loss, and trivially repeatable/scalable (30+ concurrent requests via Turbo Intruder in one attack window).

---

# Key Lessons & Patterns

## Vulnerability Patterns

Uses the same code that must be burnt after being used once.

## Future Hunting Rules

For every state-changing endpoint tied to value (redeem, refund, purchase, withdraw, approve, transfer, apply-coupon), test whether the check-then-act sequence is atomic by firing concurrent requests via Turbo Intruder.

---

# Personal Reflection

I was surprised race conditions can be exploited with a $2 tool extension and no custom scripting; the developer's mistake was treating redemption as instant rather than a two-step check-then-write; I'll now test every value-changing endpoint for atomicity

# References

- Original report: https://hackerone.com/reports/759247
- Related OWASP category: **CWE-362: Concurrent Execution using Shared Resource with Improper Synchronization ('Race Condition') —** https://owasp.org/www-project-web-security-testing-guide/v42/4-Web_Application_Security_Testing/10-Business_Logic_Testing/
- Related research/articles: https://medium.com/@mrasg/how-a-race-condition-became-an-account-takeover-vulnerability-756f14990f38

*Done* :September 13, 2026 9:02 PM
