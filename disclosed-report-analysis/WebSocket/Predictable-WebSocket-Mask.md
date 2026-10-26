# curl — CVE-2025-10148: Predictable WebSocket Mask

## Report Information

**Original Report:** https://hackerone.com/reports/3330839

**Platform:** HackerOne (Internet Bug Bounty)

**Date:** 26:10:2026

**Bounty:** $0 (publication-eligible via IBB; curl project itself set monetary reward to $0, deferred to IBB)
**Severity:** Low (0.1–3.9)

**Vulnerability Type:** Use of Predictable Values / Reused Nonce — WebSocket masking key reuse

**Tags:** #WebSockets #RFC6455 #CWE-323 #PredictableValues #Curl

# Executive Summary

curl's WebSocket client implementation generates a single masking key when a WebSocket connection is accepted and reuses that same key for every subsequent frame on the connection, rather than generating a fresh key per frame as RFC 6455 §5.3 mandates. This affects every version of curl since WebSocket support was introduced. The impact is narrow in curl's specific deployment context but represents a genuine, reusable security property violation that generalizes to cache-poisoning and proxy-confusion attacks described in the RFC itself.

# Vulnerability Overview

## Vulnerability Details

The masking key is generated once, in `Curl_ws_accept`, and stored (`enc.mask`). Every frame subsequently written — both header (`ws_enc_write_head`) and payload (`ws_enc_write_payload`) — reuses this same stored value instead of calling the random-generation routine again per frame.

RFC 6455 §5.3 is explicit that the masking key must be freshly and unpredictably chosen **per frame**, specifically to prevent an attacker who can influence frame content from being able to predict the bytes that will appear on the wire. Reusing one key means that once an attacker observes or derives the key for the first frame (e.g. because they control or can infer part of the plaintext), every subsequent frame's masking becomes trivially predictable — the XOR-based masking is only as strong as the key's unpredictability across frames, and a static key defeats that property entirely.

Required conditions: an attacker needs the ability to observe (or cause observation of) curl-originated WebSocket traffic over the lifetime of a connection, and some way to leverage predictable masking — most realistically the cache-poisoning/proxy-confusion scenario described in RFC 6455 §10.3, where a malicious endpoint can craft frame content that, once unmasked incorrectly by a naive intermediary, resembles a plausible HTTP request/response and poisons a caching proxy.

## Root Cause Analysis

Logical implementation failure, not a weak algorithm. The code path responsible for generating a new mask per frame was simply never invoked after the initial connection-accept step — the RNG itself (`Curl_rand`) was not the issue; it was the control flow failing to call it again. This is a good example of a spec-compliance bug that exists purely because a "do this once per connection" step was implemented where "do this once per frame" was required — an easy mistake when state is already being tracked at the connection level and a per-frame value gets treated the same way.

# Hunter Analysis

## Original Hunter's Approach

The reporter (cruocco) found this through direct source code review of curl's WebSocket implementation rather than black-box traffic testing — they cite specific line numbers in `lib/ws.c` for both the mask generation and its two reuse points. No "Steps to Reproduce" were provided in the report (listed as "None"), consistent with a pure code-audit finding rather than an exploited PoC. This is a static-analysis / spec-conformance-review discovery: compare the implementation's actual behavior against the RFC's explicit normative requirements, line by line.

## My Alternative Approach

### Recon
On a black-box target (not open-source, no code access), this specific bug class isn't discoverable by code review — it would require either (a) access to the target's WebSocket client source, or (b) observing multiple frames on the wire and checking whether the masking key byte pattern repeats across frames, which is directly checkable from captured traffic.

### Testing Strategy
- Capture several consecutive WebSocket frames from the same connection (Burp WebSockets history, or raw packet capture).
- Extract the first 4 bytes of each frame's mask field.
- Compare across frames — if identical across multiple frames, the mask is static/reused rather than freshly generated, confirming the same class of flaw.
- This is a quick, cheap check worth running any time a target uses a custom or lesser-known WebSocket client library, not just well-audited ones like curl.

### Tools
- Burp Suite (WebSockets history / Hex view, to inspect raw frame bytes including the mask field)
- Wireshark, for lower-level packet capture if Burp's view isn't granular enough
- Any WebSocket client library's source, if available, for direct code review (as the original hunter did)

### Payloads / Requests
No crafted payload required to *detect* this — it's purely observational (compare mask bytes across frames). Exploiting it (cache poisoning per RFC 6455 §10.3) requires a much more involved chain against a vulnerable intermediary, outside the scope of this specific finding.

# Impact Analysis

## Technical Impact
An attacker able to observe and/or influence content on a long-lived WebSocket connection using this flawed implementation can predict the masking of all subsequent frames once they've determined (or guessed) the key from an early frame. In the worst case described by RFC 6455 itself, this enables crafting frame content that, if misinterpreted by a vulnerable caching proxy sitting in the path, can poison that cache — potentially serving malicious script to unrelated users behind the same proxy.

## Business Impact
For curl specifically — minimal direct business impact, since curl is a client library/tool, not a server exposed to arbitrary attacker-controlled proxies in most deployments. The real-world impact depends heavily on deployment context: software embedding curl's WebSocket client behind a vulnerable caching proxy is the scenario where this actually bites.

## Scope & Severity
Low severity was the correct call here, and it's a useful calibration data point: a genuine, confirmed spec-violation in a widely-deployed library does not automatically imply high real-world exploitability. The attack requires a specific, somewhat unusual deployment topology (vulnerable intermediary caching proxy in the path) to actually matter — absent that, the predictable mask is a property violation without a direct exploitation path. Attack complexity is high; required privileges are effectively "network position to observe/relay traffic," and affected users depend entirely on whether a vulnerable proxy sits in the path.

# Key Lessons & Patterns

## Vulnerability Patterns
- Pattern 1: Per-frame/per-request security values (nonces, masks, tokens) generated once at connection/session setup and incorrectly reused across multiple subsequent operations that each individually require a fresh value.
- Pattern 2: Spec violations found via direct code review against the normative RFC text, rather than black-box traffic fuzzing — worth doing on any open-source dependency a target uses.
- Pattern 3: Severity triage separating "this violates the spec and is objectively wrong" from "this is practically exploitable in common deployments" — these are not the same question, and good reports/triagers keep them distinct.

## Future Hunting Rules
- When testing any WebSocket-heavy target, capture several consecutive frames and diff their mask bytes — a quick, cheap check for this exact class of bug if the target uses a custom/in-house WS client.
- When a target depends on a well-known open-source library (like curl) for networking, check that library's own disclosed-vulnerability history — a library-level bug can affect every application built on it, and recently disclosed ones may not yet be patched in the target's deployed version.

# Personal Reflection

- What surprised me: a severity-Low finding went through the full CVE assignment and public disclosure process — confirms that CVEs are about *documenting a real spec violation*, not a signal of "this is critical, pay attention." Useful to recalibrate against treating "it got a CVE" as proof of high impact.
- What mistake did the developer make: treated a per-frame requirement as a per-connection one — an easy category error when state is already being tracked at the connection level.
- What did I learn: the value of comparing an implementation directly against RFC/spec normative language line-by-line as a vulnerability-discovery method, separate from traffic-based black-box testing.
- How will I apply this during hunting: when a target's WebSocket (or any protocol-heavy) feature is built on an identifiable open-source library, check that library's disclosed CVE/HackerOne history before assuming the implementation is sound — and for anything observable on the wire (like a mask), a quick repeated-value check across multiple instances is a nearly free way to catch "should be random, isn't" bugs.

# References

- Original report: https://hackerone.com/reports/3330839
- CVE: CVE-2025-10148
- Related RFC: RFC 6455 §5.3 (Client-to-Server Masking), §10.3 (Attacks on Infrastructure — Masking)
- Related CWE: CWE-323 (Reusing a Nonce, Key Pair in Encryption) — also discussed as CWE-340 candidate during triage
