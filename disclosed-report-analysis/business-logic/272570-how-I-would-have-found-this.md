# Bitwarden — Organization Admin Privilege Escalation To Owner

## Report Information

**Original Report:** [hackerone.com/reports/272570](https://hackerone.com/reports/272570)

**Platform:** HackerOne

**Program:** Bitwarden

**Reporter:** rhynorater (Justin Gardner)

**Reported:** September 28, 2017 | **Resolved:** September 28, 2017 | **Disclosed:** October 28, 2017

**Bounty:** Hidden

**Severity:** Medium (4 ~ 6.9)

**Weakness:** Business Logic Errors

**CVE:** None

**Scope:** vault.bitwarden.com (changed from bitwarden.com by the program)

**Tags:** #BusinessLogic #PrivilegeEscalation #BrokenAccessControl #RBAC

**Original title:** "Application Logic Error - Admin -> Owner Privledge Esc / Organization takeover" (renamed by the program to the current title)

---

# Executive Summary

An organization member with the **admin** role could promote themselves to **owner** through the normal "edit user" flow, then remove the original owner and take over the whole organization. The server let any admin set any role, including owner, and did not check that only an owner can grant or manage the owner role. Bitwarden patched it in production about 1.5 hours after the report was submitted, in that night's release.

---

# Vulnerability Overview

## Vulnerability Details

- **Where:** organization user management in the Bitwarden web vault (invite/edit/remove users).
- **Functionality:** role assignment inside an organization (user / admin / owner).
- **Attack flow (from the report):**
  1. Create accountA and accountB.
  2. Create an organization under accountA and invite accountB as **admin**.
  3. Accept the invite with accountB, then confirm accountB from accountA.
  4. Log in as accountB and go to the organization → invite users → edit accountB.
  5. Change accountB's role to **owner**. The change succeeds.
  6. Remove the original owner. After a login/logout cycle the original owner is gone from the organization.
- **Requirements:** an existing admin role in the target organization. No other privileges needed.

## Root Cause Analysis

The fix commit (`2444346`, "only owners can manage owners") shows exactly what was missing. Bitwarden added the same guard in four places, each checking that the acting user is an **owner of that organization** before allowing an owner-level action:

| Code path | Added check | Error message |
|-----------|-------------|---------------|
| `OrganizationService.InviteUserAsync` | inviting a user as owner requires the inviter to be an owner | "Only owners can invite new owners." |
| `OrganizationService.SaveUserAsync` | saving a user with owner type requires the saver to be an owner | "Only owners can update other owners." |
| `OrganizationService.DeleteUserAsync` | deleting an owner requires the deleter to be an owner | "Only owners can delete other owners." |
| `OrganizationUsersController.PutGroups` | updating groups for an owner requires the caller to be an owner | "Only owners can update other owners." |

So the flaw was **missing server-side authorization on role changes**: the app checked that the caller was allowed to manage users, but not that the caller's role was high enough to touch the owner role. The web UI allowing the change meant nothing was stopping it at the API layer either. The delete-user gap is what made full takeover possible, since without it an admin could promote themselves but not remove the original owner.

---

# Hunter Analysis

## Original Hunter's Approach

- Tested role boundaries with **two accounts** in one organization, the standard approach for authorization testing: one account holding the lower role, one holding the higher.
- Went straight at the role hierarchy: invited accountB as admin, then tried to change its own role upward through the normal edit-user UI instead of looking for an exotic bug.
- Proved full impact rather than stopping at "role changed": removed the original owner and confirmed the takeover persisted after a login/logout cycle.
- Wrote a clean numbered reproduction with the impact stated in one line ("Anyone who is an admin ... can take total control of the organization").
- Requested disclosure "for knowledge of other hackers" after the fix.
- Side note: also asked to be IP-whitelisted because rate limiting kept blocking his testing, and the program asked him to talk in their dev chat first.

## My Alternative Approach

### Recon
- Map every role in the app and write down what each role should be allowed to do. Here: user, admin, owner.
- List every endpoint that reads or changes roles or membership: invite, edit, remove, group assignment.

### Testing Strategy
- Two (or three) accounts, one per role. For each action, try it from the **lowest role that can reach the UI** and from every role above and below.
- Test **vertical** boundaries (admin → owner) on every role-related endpoint, not just the obvious edit one. The fix touched four separate code paths, which suggests the same missing check existed in multiple places.
- Test the full takeover chain, not just the first step: promote self, then remove or demote the higher role.
- Also test whether an admin can **demote or remove an existing owner** directly, and whether an admin can assign owner via invite instead of edit.

### Tools
- Burp Suite with two browser sessions, one per account
- The app's own UI first, then replay the requests with modified role values

### Payloads / Requests
The report describes the UI flow rather than the raw request. Reproducing it in Burp means capturing the edit-user request as accountB and changing the role/type field to the owner value.

---

# Impact Analysis

## Technical Impact
- Full privilege escalation inside an organization: admin → owner.
- Ability to remove the original owner and lock them out.

## Business Impact
- An organization's shared secrets (a password manager's core purpose) could be controlled by a malicious or compromised admin.
- Loss of trust in role separation for organization customers, who rely on the owner role being protected.
- An insider or compromised admin account could take over the org and remove the legitimate owner.

## Scope & Severity
- **Rated:** Medium (4 ~ 6.9).
- **Privileges required:** an existing admin role in the target organization. It's not exploitable by an outsider, which is what keeps it at Medium.
- **Complexity:** low. It only needs the normal UI flow.
- **Impact within the org:** total takeover of that organization.

---

# Timeline

| Time (UTC, Sept 28, 2017) | Event |
|---------------------------|-------|
| 1:05am | Report submitted |
| 1:10am | Program acknowledges, offers to talk in dev chat |
| 1:13am | Report title changed |
| 2:38am | Program says it's patched in production with that night's release, links the fix commit |
| 2:41am | Triaged |
| 2:49am | Reporter confirms the fix looks good |
| 2:51am | Resolved |
| 2:52am | Reporter requests disclosure |
| Oct 28, 2:53am | Disclosed |

Report to production fix: about 1.5 hours. Report to resolved: under 2 hours.

---

# Key Lessons & Patterns

## Vulnerability Patterns
- **Pattern 1:** in role-based systems, apps often check "can this user manage users?" but not "is this user's role high enough to touch the target role?". Always test the hierarchy, not just the permission.
- **Pattern 2:** the same missing check tends to exist in several code paths (invite, edit, delete, group update here). Finding one gap means checking all its siblings.
- **Pattern 3:** the impact of a privilege escalation comes from the whole chain. Promote-self alone is bad, promote-self plus remove-owner is a takeover.

## Future Hunting Rules
- On any multi-role product, create one account per role and test every role-touching action from every lower role, in both directions.
- Don't stop at the first successful escalation. Follow it to the worst realistic outcome and show that.
- Read fix commits on open-source targets: they show which sibling code paths had the same flaw, which is a ready-made checklist for similar apps.

---

# Personal Reflection

- **What surprised me:** how simple it was: the normal UI let an admin pick "owner" for themselves.
- **Developer mistake:** authorization checked the action but not the role hierarchy, and the gap existed in four places, not one.
- **What I learned:** two-account, per-role testing is the fastest way to find this class of bug, and the fix diff is a useful map of related weak spots.
- **How I'll apply it:** for every target with roles, build a small role/action matrix early and test each cell, especially "can role X grant or remove role Y above it?".

---

# References

- Original report: [hackerone.com/reports/272570](https://hackerone.com/reports/272570)
- Fix commit: [bitwarden/server@2444346 "only owners can manage owners"](https://github.com/bitwarden/server/commit/2444346ea91451496081c3b36254e00362527d18)
- OWASP: Broken Access Control; CWE-269 (Improper Privilege Management), CWE-285 (Improper Authorization)
