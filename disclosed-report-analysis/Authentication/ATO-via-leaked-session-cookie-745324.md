# HackerOne — Account Takeover via Leaked Session Cookie

## Report Information

**Date analyzed:** October 1, 2026

**Original Report:** https://hackerone.com/reports/745324

**Platform:** HackerOne (self-program — HackerOne triaging a report against itself)

**Bounty:** $20,000 + swag

**Severity:** High (8.3) technically, treated as Critical for bounty purposes

**Vulnerability Type:** Insufficiently Protected Credentials (session cookie disclosure + missing session-binding controls)

**Tags:** #AccountTakeover #SessionManagement #InsiderRisk #IncidentResponse #H1

---

# Executive Summary

A HackerOne Security Analyst accidentally pasted their own valid session cookie into a public report comment while reproducing a submission. A hacker (haxta4ok00) noticed it, used the cookie to log into the analyst's HackerOne account, and reported the disclosure immediately. Because HackerOne's session cookies weren't bound to the originating IP or device, the cookie worked from anywhere, granting access to multiple customers' vulnerability reports, including sensitive report contents and internal comments, until it was revoked roughly two hours later. This report is unusual: the "vulnerability" wasn't found through active testing, it was a human error plus a missing defense-in-depth control, caught and reported opportunistically.

---

# Vulnerability Overview

## Vulnerability Details

1. A Security Analyst tried to reproduce a submitted report and failed. In their reply explaining what they'd tried, they pasted a `curl` command copied from their browser console, forgetting to strip the session cookie embedded in it.
2. The reporting hacker noticed the cookie was live and usable, and instead of just flagging it, used it to log into the analyst's account to demonstrate impact.
3. Session cookies at HackerOne weren't bound to IP address or device at the time — a cookie was valid from anywhere it was presented, a known and accepted tradeoff for mobile/proxy users.
4. With a valid session, the hacker had the same access as the analyst: HAS Inbox, Triage Inbox, general Inbox, and full Report View across every program that analyst supported, exposing report metadata and, where opened, full report contents and internal comments.
5. The hacker self-reported within 20 minutes of gaining access. HackerOne revoked the cookie roughly two hours after disclosure (three minutes after triage actually started), then ran a full incident response investigation.

## Root Cause Analysis

Two independent failures stacked:

- **Primary/triggering cause:** human error — a live session cookie was pasted into a public-facing comment. Not a code bug; a process/tooling gap (no warning when sensitive-looking strings are about to be submitted).
- **Enabling/systemic cause:** no additional binding on session cookies (IP or device) beyond the initial SSO login. Once a cookie existed, it was portable and reusable by anyone who obtained it, by any means, with no secondary check that the presenter was the original holder.

HackerOne's own RCA is unusually explicit about *why* this control didn't already exist: IP binding would degrade UX for users on mobile networks or proxies where the IP changes mid-session, a deliberate risk-acceptance decision made before this incident, not an oversight. That's a useful detail for your writeup — the missing control wasn't unknown, it was a known tradeoff that got re-evaluated only after this incident forced the question.

---

# Hunter Analysis

## Original Hunter's Approach

There wasn't a testing methodology here in the usual sense — this was **opportunistic discovery of an exposed secret**, not an actively hunted bug:

1. The hacker was already engaged on a separate, unrelated report.
2. Noticed a session cookie sitting in the analyst's reply.
3. Recognized what it was and what it granted access to.
4. Used it to confirm the impact was real (this is the part HackerOne later pushed back on — see below) rather than just theorizing about it.
5. Reported it essentially immediately.

This is worth naming as its own hunting category separate from active testing: **secrets-in-plaintext monitoring** — watching for credentials, tokens, and cookies accidentally exposed in comments, commit history, logs, support tickets, or error messages, rather than actively probing for a logic flaw.

## What Went Right, and What Didn't, in How the Hacker Handled It

This report is genuinely useful to study for **professional conduct**, not just technical content, and it's worth including that angle explicitly in your writeup:

- **Reported immediately, no ransom, no threats** — textbook good-faith disclosure.
- **HackerOne still flagged a concern afterward:** the hacker continued browsing multiple reports and pages *after* initially confirming access, more than was needed to prove the vulnerability was real. HackerOne's own words: "we didn't find it necessary for you to have opened all the reports and pages in order to validate you had access." They paid the full bounty and didn't penalize this instance, but explicitly updated their policy afterward and put the hacker on notice that this pattern could disqualify a future bounty.
- **Lesson for your own conduct as a hunter:** proof of access should stop at the minimum needed to demonstrate impact. Screenshot or describe what you *could* see, don't continue exploring once you've proven the primitive works. This is a real professional line that affects payout and trust, not just an abstract ethics point.

## My Alternative Approach (as a defensive/architecture exercise)

Since there's no active testing technique to reconstruct here, the valuable exercise is architectural: what controls would you check for or recommend on any platform handling session cookies for privileged internal users?

### Recon

- Identify whether an application binds sessions to anything beyond the credential itself — IP, device fingerprint, user-agent consistency.
- Check whether the platform has any automated detection for secrets (tokens, cookies, keys) being pasted into user-facing text fields, comments, or support tickets.

### Testing Strategy

- If you ever legitimately obtain a session identifier during authorized testing (e.g., your own test account), check whether reusing it from a different IP/device/browser is silently accepted or blocked.
- Review how quickly a platform can revoke a single session versus requiring a full password reset — fast, granular revocation matters enormously in an incident like this one.

### Notes

This category of finding is far more likely to come from log review, dorking, and passive monitoring (e.g. searching your own or client's public GitHub, support forums, Slack exports, or — as here — a bug bounty platform's own report text) than from active exploitation. Worth a separate checklist entry on your methodology page distinct from your logic-flaw checks.

---

# Impact Analysis

## Technical Impact

- Full session-level access to a Security Analyst's HackerOne account, across every program they supported.
- Exposure of report metadata (titles, states, assignees, reporters) across HAS Inbox, Triage Inbox, and Inbox views.
- Exposure of full report contents and internal (non-public) comments for any report actually opened via Report View.
- Access was bounded by the analyst's own program scope — not a platform-wide compromise, but still multi-customer.

## Business Impact

- **Customer trust impact at platform-operator level** — HackerOne is itself a security intermediary; a breach of its own analysts' access undermines the core promise the platform sells to its customers (confidential handling of vulnerability data).
- **Multi-tenant blast radius** — because one analyst supports multiple customer programs, a single leaked credential exposed several organizations' sensitive report data simultaneously, not just one.
- **Forced public accountability** — HackerOne published a full, detailed RCA (unusually transparent), which is itself a reputational cost/benefit trade: short-term exposure of "we messed up," long-term credibility gain for handling it openly.
- **Regulatory/notification obligations** — affected customers had to be individually notified that their report data had been viewable, a real operational and trust cost regardless of whether anything was misused.

## Scope & Severity

- **Affected users:** customers whose reports were assigned to or handled by the specific compromised analyst — not HackerOne's entire customer base.
- **Attack complexity:** trivial once the cookie was exposed — this is the core point. No skill was needed to exploit it, all the "difficulty" was in the analyst's initial mistake, not in anything the hacker engineered.
- **Privileges required:** none beyond finding the leaked cookie.
- **Severity reasoning:** CVSS base scored High (8.3) — `AV:N/AC:H/PR:N/UI:R/S:C/C:H/I:H/A:H`. HackerOne chose to pay as Critical rather than the CVSS-implied High, explicitly because environmental/business impact (the amount and sensitivity of data actually reachable) justified it over the formulaic score. Worth noting in your writeup as an example of a company overriding a CVSS number upward based on real-world context, useful precedent to cite if you ever want to argue for a higher payout than a pure CVSS calculation implies.

---

# Key Lessons & Patterns

## Vulnerability Patterns

- Sensitive credentials (session cookies, tokens, API keys) pasted into reproduction steps, debug output, or support communication — a recurring, low-tech, high-impact class independent of any code vulnerability.
- Sessions with no secondary binding (IP, device) beyond the original credential — once a session token leaks, by any means, it's fully portable.
- Defense-in-depth tradeoffs (like skipping IP-binding for UX reasons) that are reasonable in isolation but turn a single human mistake into a full account compromise.

## Future Hunting Rules

- Treat any comment thread, support ticket, changelog, or pasted terminal output (in scope, and only where authorized) as a potential secrets-leak surface, not just application logic.
- When you do gain access via a found credential, stop at the minimum proof of impact — document, don't tour.
- When writing up severity, consider arguing environmental/business impact explicitly if you believe a formulaic CVSS score undersells real-world exposure — HackerOne's own bounty decision here is a citable precedent for that argument.
- Note whether a target platform binds sessions to anything beyond the token itself; if it doesn't, any future leaked-credential finding you make there deserves a severity bump for that reason alone.

# References

- Original report: https://hackerone.com/reports/745324
- CWE-522, Insufficiently Protected Credentials: https://cwe.mitre.org/data/definitions/522.html
- OWASP Session Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
- OWASP WSTG — Testing for Session Management: https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/06-Session_Management_Testing/README
- PortSwigger Web Security Academy — Authentication vulnerabilities (session handling section): https://portswigger.net/web-security/authentication
- HackerOne's own bug bounty policy update referenced in this report (proof-of-impact conduct standard) — check HackerOne's current public policy page for the live version.
