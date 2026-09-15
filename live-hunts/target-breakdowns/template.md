Notes on how it's structured:

- **Section 3 (auth matrix) and section 4 (business logic)** are the parts that make a breakdown "yours" — they force you to state what *should* happen before you test what *does* happen. That gap is where bugs live.
- **Section 5** is a checklist so you're not relying on memory to cover all vuln classes on every target — run it every time, even on "boring" targets.
- **Section 6** is deliberately ranked by impact × likelihood, not novelty — this is also good training for report-writing later, since severity justification is exactly this same reasoning restated for a triager.
- **Section 7** exists so once you're hunting, the breakdown becomes a living document instead of a one-time exercise — you're tracking which hypotheses paid off, which builds your own pattern library over time (same value as the 200 disclosed reports, but from your own testing).

# TARGET BREAKDOWN — [target name]

Date: _______  Program: _______  Platform: H1 / Bugcrowd / Intigriti / HackenProof

## 1. ASSET INVENTORY

| Subdomain | Status | Purpose (guess) | Tech stack | Flag? |
| --- | --- | --- | --- | --- |
|  |  |  |  |  |

Flag = admin/staging/dev/internal/api/beta/vpn/uat, or anything recon auto-flagged.

## 2. ENTRY POINTS

List every form, upload, API endpoint, webhook config, auth flow, file handling feature.

| Entry point | What it does | Auth required? |
| --- | --- | --- |
|  |  |  |

## 3. AUTH BOUNDARY MATRIX

For every role in the app (Owner/Admin/User/Guest/etc.), map what SHOULD be blocked.

| Action | Role A can? | Role B can? | Cross-tenant (User A → User B's data) possible? |
| --- | --- | --- | --- |
| View resource X |  |  |  |
| Edit resource X |  |  |  |
| Delete resource X |  |  |  |
| Invite/assign roles |  |  |  |

Explicitly test: does the frontend hide an action, or does the backend actually reject it?

## 4. BUSINESS LOGIC FLOWS

For EACH flow (signup, checkout, refund, invite, password reset, webhook, file upload, etc.), run the 3-question drill:

**Flow name:** _______

1. **Job in one sentence:** What is this feature supposed to do?
2. **What does the server have to trust for this to work correctly?**
3. **What happens if that trust is misplaced?** → this becomes a hypothesis below.

(Repeat block per flow — don't skip ones that "seem boring," those are usually the least-tested ones.)

## 5. VULN-CLASS CHECKLIST (run against every entry point)

- **IDOR / BAC** — does changing an ID/reference expose another user's/tenant's data? Test both read AND write.
- **Auth/Session** — JWT alg confusion, weak secret, session fixation, missing invalidation on logout/password change, predictable reset tokens.
- **SSRF** — any field where the server fetches a URL you supply (webhooks, "import from URL," PDF generators, image proxies). Test internal ranges, cloud metadata IP, DNS rebinding.
- **Injection** — SQLi, NoSQLi, command injection, template injection — anywhere user input reaches a query, shell, or template engine.
- **Business logic / race conditions** — can a multi-step process (checkout, refund, coupon redeem) be triggered twice concurrently, or steps skipped/reordered?
- **File upload** — type/size validation bypass, stored XSS via SVG/HTML, path traversal on filename.
- **CORS/CSRF** — reflected arbitrary origin with credentials allowed? State-changing GET requests? Missing CSRF tokens on sensitive actions?
- **Privilege escalation** — role param tampering, mass assignment (extra fields accepted on profile/object update endpoints).
- **Info disclosure** — verbose errors, exposed debug endpoints, source maps, `.git`/`.env` exposure, API docs revealing undocumented endpoints.
- **Rate limiting / brute force** — login, OTP, password reset, invite endpoints — do they lock out or throttle?

## 6. RANKED HYPOTHESIS LIST

Pull from sections 3-5. Rank by (impact × likelihood), not just "coolest bug."

| # | Hypothesis | Why (which assumption breaks) | Vuln class | Priority |
| --- | --- | --- | --- | --- |
| 1 |  |  |  |  |
| 2 |  |  |  |  |

## 7. NOTES / OPEN QUESTIONS

Anything unclear that needs testing to resolve, not reasoning.

## POST-TEST LOG (fill in once you're allowed to test)

| Hypothesis # | Result | If bug: severity/impact argued | Report status |
| --- | --- | --- | --- |
