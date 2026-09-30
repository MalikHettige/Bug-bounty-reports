# **Mattermost** — **Reset password link sent over unsecured http protocol**

## Report Information

**Original Report:** https://hackerone.com/reports/1888915

**Date:** 30-09-2026

**Platform:** HackerOne 

**Bounty:** $750 + swag

**Severity:** High (7.3)

**Vulnerability Type: Cleartext transmission of a sensitive token, leading to account takeover**

**Tags:** #IDOR #AccessControl #H1 #AccountTakeover #Transport #CleartextTransmission #H1

# Executive Summary

Mattermost Cloud sent password reset emails containing `http://`links instead of `https://`. When a victim clicks one, the reset token travels in plaintext, so anyone on the same network can capture it and reset the password. This is a full account takeover.

# Vulnerability Overview

## Vulnerability Details

The bug is in the forgot-password flow at `/reset_password`endpoint. 
Attack flow: victim requests a reset, gets the email, clicks the link, and the first request goes out over HTTP with the token in the URL. **A passive sniffer** on the same network reads it, then the attacker opens the link and sets a new password. 
Required conditions: the attacker is on the victim's network path, and the victim clicks before the token expires or is used.

## Root Cause Analysis

The application generated password-reset links using the `http://` scheme instead of `https://`, and no server-side control (forced redirect, HSTS enforcement) existed to compensate. The report doesn't specify why the scheme was wrong — likely candidates are a misconfigured Site URL setting or a reverse proxy that terminates TLS without informing the app it's behind HTTPS — but this wasn't confirmed by Mattermost in the disclosed thread.

# Hunter Analysis

- Signed up for a fresh workspace.
- Triggered forgot-password on their own account.
- Opened the email and copied the link into Sublime Text rather than clicking it.
- Visually inspected the scheme in the raw text.

The "interesting observation" I assume here is the method that was used, not skill. The attacker looked at raw text instead of trusting the rendered email client, which usually hides the scheme behind a clickable button.

## Original Hunter's Approach

[How the researcher discovered the vulnerability.]

Document:

- Recon approach
- Testing methodology
- Tools used
- Interesting observations
- Researcher's thought process

## My Alternative Approach

### Recon

- What would I identify first?
- What endpoints/features would I focus on?

### Testing Strategy

For each flow, 

- I would view raw source of the message rather than the rendered version.
- Check the scheme on every link.
- Check response headers for `Strict-Transport-Security` on the domain.
- Check whether the HTTP version of the link 302s to HTTPS or actually serves the page.

### Tools

- Burp Suite (repeater to replay and inspect headers rather than just clicking through)
- Browser DevTools
- `curl -I` against the HTTP link to check redirect behavior

### Payloads / Requests — For each sensitive-token endpoint, check scheme + redirect behavior

```jsx
curl -I http://target/reset_password_complete?token=X
curl -I http://target/verify_email?token=X
curl -I http://target/invite/accept?token=X
curl -I http://target/sso/saml/acs
curl -I http://target/oauth/callback?code=X
```

# Check whether HSTS is present on the base domain

```
curl -I https://target/ | grep -i strict-transport-security
```

# Impact Analysis

**Technical Impact**

An attacker positioned on the same network as the victim (`AV:A`) can passively capture the plaintext HTTP request containing the password-reset token. No active interception is required, sniffing alone is sufficient (`AC:L`). Once the attacker has the token, they can:

- Complete the password reset themselves, before or instead of the victim.
- Set a new password and lock the legitimate user out.
- Log in as the victim with full account privileges (`C:H`, `I:H`).

There's no downstream service disruption from the bug itself (`A:N`), and no privileges are needed beforehand (`PR:N`). The victim must click the link (`UI:R`), so it needs user interaction but no cooperation, they just have to do the thing they were already going to do.

**Business Impact**

- **Account compromise at scale in shared-network scenarios.** Any environment where multiple users are on the same physical or wifi network, offices, co-working spaces, conferences, increases exposure, since it puts multiple potential victims within sniffing range of one attacker.
- **Trust erosion.** Mattermost is often deployed for internal team communication, so a compromised account can mean exposure of private channels, other users' messages, and internal business context, not just the one account.
- **Regulatory/compliance exposure**, depending on what's discussed in the workspace, a takeover could trigger data-breach obligations for the customer running that Mattermost instance.
- **Low attacker cost, no detection footprint.** Passive sniffing leaves no logs on the server side, so the org has limited ability to detect that a takeover happened this way versus, say, a leaked password.

**Scope & Severity**

- Affected users: every user who triggers a password reset while on a network an attacker can observe, essentially the whole user base, since the bug is in link generation, not scoped to any one account type.
- Attack complexity: low once positioned on the network; the constraint is physical/network proximity, not technical skill.
- Privileges required: none.
- Severity reasoning: this justifies the triager's High (7.3). It's a full account-takeover primitive (`C:H`/`I:H`) gated only by network adjacency, which is a modest bar, especially on public wifi, conference networks, or any shared office LAN. The reporter's own Medium submission undersold it; the fix (force HTTPS scheme in generated links, add HSTS) is trivial relative to the impact it prevents.

# Key Lessons & Patterns

## Vulnerability Patterns

- Pattern 1: Sensitive tokens (reset, invite, magic-link) transmitted over cleartext HTTP.
- Pattern 2: Missing HSTS enforcement on auth-adjacent endpoints.
- Pattern 3: Severity misjudged by reporters when `AV:A` conditions apply — passive network position, no active MITM needed.

## Future Hunting Rules

- Read the raw source of every email, and check every link's scheme.
- A sensitive token in a URL over HTTP is a finding on its own.
- Program triagers can rate higher than expected, must **submit the finding with a solid impact argument.**
- Check `curl -I` on the HTTP version **of every sensitive endpoint** to see if it redirects to HTTPS.
- Check for the `Strict-Transport-Security` header on the base domain.

# Personal Reflection

Most developers fails to implement authorization checks or keep sensitive parts of important requests hidden by making secured `http`protocols. I learned that all I have to do is just give a little search on sensitive endpoints aside from just password reset such as Email verification, passwordless login links, export download links, invite links or backup-code delivery or SSO/SAML assertion consumer service (ACS) URLs or even some of account-settings links. Same checklist applies to all of them. Best to view the raw email source, check the link scheme, check if the token is single-use, and check its expiry window. 

The reset-password case is just the easiest one to stumble into because literally every app has that flow.

### What I learned

- **HTTP** sends everything in plain-text. Anyone between me and the server can read it: the coffee shop wifi owner, a compromised router, an ISP.
- **HTTPS** is HTTP wrapped in Transport Layer Security (TLS - a security protocol that encrypts necessary data), so that same person sees only encrypted junk.
- **Sniffing** just means passively capturing network traffic that passes by, without altering it. Wireshark or tcpdump reading packets off an interface is sniffing.
- **HSTS** (HTTP Strict Transport Security) is a response header a site sends: `Strict-Transport-Security: max-age=...`. **Once a browser has seen it once for a domain, it refuses to make plain HTTP requests to that domain again** for the given duration, it rewrites them to HTTPS internally before anything goes out. 
It's exactly the control that would have stopped this Mattermost bug on a repeat visit, though not on the very first request before the browser has ever seen the header.

On how traffic gets exposed, conceptually:

- **Open or weak wifi**: anyone with a radio in range can capture packets, no interception step needed.
- **Shared network segments (ARP spoofing)**: on a switched LAN, telling a victim's machine that you're the router lets you sit in the traffic path.
- **Rogue access points**: an attacker-run wifi network the victim joins voluntarily or automatically.
- **Compromised network infrastructure**: a malicious or breached router, ISP, or corporate proxy.

### References

**TryHackMe:**

- L2 MAC Flooding & ARP Spoofing — https://tryhackme.com/room/layer2
- Search "Man-in-the-Middle Detection" and "Wireshark" on TryHackMe's room catalog: https://tryhackme.com/hacktivities

**HackTheBox Academy:**

- Module catalog: https://academy.hackthebox.com/catalogue — search "Network Enumeration with Nmap" and "Pentesting Network" from here, since I can't confirm exact module URLs stayed stable.

**Standards / methodology:**

- OWASP WSTG — Testing for Weak Transport Layer Security: https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/09-Testing_for_Weak_Cryptography/01-Testing_for_Weak_Transport_Layer_Security
- CWE-319, Cleartext Transmission of Sensitive Information: https://cwe.mitre.org/data/definitions/319.html
- RFC 6797, HTTP Strict Transport Security: https://datatracker.ietf.org/doc/html/rfc6797

**Background on the attack technique:**

- Wikipedia, ARP spoofing: https://en.wikipedia.org/wiki/ARP_spoofing
- bettercap docs on spoofing modules: https://www.bettercap.org/modules/ethernet/spoofers/introduction/

**The report itself:**

- https://hackerone.com/reports/1888915
