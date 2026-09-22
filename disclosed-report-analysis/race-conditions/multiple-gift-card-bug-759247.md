# Reverb — **Race Condition allows to redeem multiple times gift cards which leads to free "money"**

## Report Information

**Original Report:** https://hackerone.com/reports/759247

**Date:** 22-09-2026

**Platform:** HackerOne 

**Bounty:** Hidden

**Severity:** High (7 ~ 8.9)

**Vulnerability Type:** Race conditions

**Tags:** #IDOR #AccessControl #H1

# Executive Summary

A race condition in Reverb.com's gift-card redemption endpoint allowed an authenticated attacker to redeem a single gift card multiple times by firing concurrent requests via a single-packet attack (Turbo Intruder). Because the "is this card already redeemed?" check and the "mark it as redeemed" write were not atomic, all 30 concurrent requests passed validation before any of them committed the used-state, letting a single $25 card be redeemed 7 times for $175 in account credit. This is significant because the financial loss scales directly with concurrency — an attacker isn't limited to 7x, only to how many parallel requests they choose to fire.

# Vulnerability Overview

## Vulnerability Details

The race condition vulnerability is exposed `https://sandbox.reverb.com/<lang>/redeem`endpoint while an authenticated attacker would perform single-packet-attack while redeeming a gift card. Resulting the attacker to be able to redeem it 7 times and potentially buy an item for free. 

## Root Cause Analysis

The root cause is a TOCTOU (time-of-check-to-time-of-use) race condition — the server checks whether the gift card has already been redeemed, then separately marks it as redeemed, and these two steps are not atomic. Concurrent requests sent via a single-packet attack can all pass the 'is this valid?' check before any of them commits the 'now mark it used' write, so all of them succeed.

**Hunter Analysis — Original Approach**

- Bought a real gift card ($25), intercepted the `/redeem` POST, sent to Turbo Intruder
- Used the classic single-packet race template: `engine.queue()` with a `gate='race1'`, all 30 requests held then released together via `engine.openGate('race1')`
- Verified via account balance screenshot: 7 successful redemptions of one $25 card → $175 credited
- One detail worth flagging as a possible reporting artifact, not confirmed technique: the report mentions setting a custom header `x-request: %s`, described as "needed by Turbo Intruder" — that's not a standard Turbo Intruder requirement, likely specific to the Burp/TI version at the time (2019) or a report-writing shorthand. Don't take it as something you need to replicate; note it as unclear rather than present it as fact.

**My Alternative Approach**

- You already have Turbo Intruder reps from PortSwigger labs, so this section should be fast — Recon: identify the redeem/claim/apply endpoint for anything representing stored value (gift card, credit, points); Testing: fire the standard race template against it before assuming single-use is enforced, same as you'd now do by default after today's pattern (AWS coupon → Stripe discount → this)

**Impact Analysis**

- Technical: unlimited store credit generation from one paid gift card, scales with concurrency count
- Business: direct financial loss per exploited card; **important nuance to include** — Reverb's own resolution comment said they had *behind-the-scenes fraud logic* that would likely have caught and reversed these transactions on the real (non-sandbox) site. That's a real mitigating control worth naming — it doesn't invalidate the bug, but it's why severity often gets judged not just on "what's theoretically possible" but "what actually survives downstream defenses." Worth remembering when you estimate your own severity on future finds.
- Scope & Severity: requires authentication + upfront purchase (raises attack complexity slightly vs. AWS coupon which needed no purchase); rated High (7–8.9); resolved same day disclosed, bounty hidden

**Key Lessons & Patterns**

- Third report today in the same lineage: reuse (AWS, no concurrency needed) → race (Stripe $20K, race needed) → race (this one). The unifying pattern: any endpoint that grants value from a "consumable" token needs testing for both simple reuse *and* concurrent-request abuse — presence of a "used" flag doesn't mean it's enforced atomically
- Add "fire redeem/claim/apply endpoints through Turbo Intruder's race template" as a default test, not a follow-up hypothesis, for any value-bearing resource

**References**

- CWE-362 — Concurrent Execution using Shared Resource ('Race Condition') — https://cwe.mitre.org/data/definitions/362.html
- CWE-367 — Time-of-check Time-of-use (TOCTOU) Race Condition — https://cwe.mitre.org/data/definitions/367.html
- PortSwigger — Race conditions (the labs you've likely already run against this exact pattern) — https://portswigger.net/web-security/race-conditions
