# How I Would Have Found This: Reauthentication Bypass via Email Manipulation

## 1. The Target

Liberapay is a payment or donation platform where users manage emails
and passwords.

## 2. My Recon Approach

I would sign up and get familiarize with the account before start hunting. I would then test every  settings feature such as change password, add email, change email, delete email, reset password.  would also note what endpoints or features require reauth ( like password change) and which don’t.

## 3. The Vulnerability Class

Business Logic Flaw — Inconsistent Security Controls

## 4. My Testing Steps

1. Attacker gains temporary session access to victim's account
2. Navigate to account settings → Add email
3. Add attacker's own email address
4. Confirm email via link sent to attacker's inbox
5. Make attacker's email primary
6. Log out of victim's session
7. Go to login page → Forgot password
8. Enter attacker's email (now victim's primary email)
9. Click reset link sent to attacker's inbox
10. Set new password
11. Log in with new password — permanent account takeover

## 5. Impact

Attacker gains the capability to  control victim's account, payment methods or even donation history. Worst possible outcome would be permanent account takeover and victim cannot recover account without contacting support. This impact negatively : reputation damage for the platform

## 6. Why I Would Have Caught It

I would test the security boundaries between related features. If password change requires reauth I would go check if email change does too. Inconsistent
controls are a pattern I hunt for.

## 7. What I Learned for My Hunting

Reading the report I learned that I should test all account features, not just password. And to argue back with evidence professionally with the triage if the report was misinterpreted.

## 8. References

| CWE-306: Missing Authentication for Critical Function | `https://cwe.mitre.org/data/definitions/306.html` |
| --- | --- |
| OWASP: Testing for Business Logic Flaws | `https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/10-Business_Logic_Testing/` |
| PortSwigger: Business Logic Vulnerabilities | `https://portswigger.net/web-security/logic-flaws` |
