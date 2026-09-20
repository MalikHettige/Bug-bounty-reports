# Progress of Disclosed reports observations

Reports analyzed: 21 / 200+ (2026 target)

| Report | Title | Vulnerability | What I'd Check For / Pattern |
|---|---|---|---|
| 318751c | Access to Private Photos of Apps in App section | IDOR | Any endpoint returning user-scoped media/files by a sequential or guessable ID — swap the ID, check for missing ownership validation on the object, not just auth on the request |
| 120121c | Delete any group of any organization remotely | Critical IDOR | Destructive actions (delete/remove) on org-scoped resources — check if `org_id`/`group_id` is validated against the *authenticated* user's org, not just checked for existence |
| 642886c | Reauthentication for changing password bypass | Authentication bypass | Multi-step sensitive actions (password change, email change) — check if the "confirm identity" step is actually re-verified server-side, or just a client-side gate that can be skipped |
| 3219944 | Scheduled data leak to other accounts by "projectID" | IDOR | Background/scheduled jobs or exports that reference an object ID — these often skip the same authz checks the live UI enforces, since they're "internal" |
| 1849626 | Fee discounts can be redeemed many times, resulting in unlimited fee-free transactions | Business Logic Errors (Race Condition) | Any one-time-use resource (discount, coupon, invite) — fire concurrent requests at the redeem endpoint, check if the "used" flag is set atomically or has a TOCTOU window |
| 415081 | IDOR to add secondary users in www.paypal.com/businessmanage/users/api/v1/users | IDOR | "Add user/member" endpoints on account/team management — check if the target account's ID in the request body/URL is validated against the caller's own account scope |

Update this table as each new disclosed-report-analysis file lands in `disclosed-report-analysis/`. One row per report, matching the folder it lives in.
