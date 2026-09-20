08/06/2026

# How I Would Have Found This: Critical IDOR - Delete any group of any organization remotely

## The Target

The target is Veris, a platform used to manage physical workplaces. Basically a organization/group management platform - essentially a digital front desk and office assistant.

## My Recon Approach

Since it is a group management platform my target has to be something pertaining to that. Groups. Since he said an attacker could “Delete any group of any organization remotely” I will have to create 3 test accounts (including attacking one) for 2 organizations and create several groups to each one. That's how I can confirm the bug is critical.

## The Vulnerability Class

Insecure Direct Object Reference (IDOR)

## My Testing Steps

1. Create 2 victim accounts ( account A and B)
2. Create attacker account ( account C) with no relation to accounts A & B
3. Create 2 organizations with several groups (ex. Group 1) for each of victim accounts
4. Log in as account C, Create a group (group 55)
5. Intercept the `Delete-group` request and locate the `id` parameter
6. Replace its value with the a group`id` account A
7. Send that modified request
8. Go to account A and observe the group is deleted without authorization.
9. Repeat it with multiple group IDs of different organizations
10. Confirm IDOR vulnerability exists and is systematic.

## Impact

An authenticated attacker could intercept a `Delete-group` request and substitute the ID with an arbitrary number and perform group deletion of any organizations without any authorization checks.

## Why I Would Have Caught It

I test every numeric ID parameter I find of endpoints. When I see `id=`, `group_id=` or any number in a request I would immediately try to change it to another value because I learned IDOR vulnerabilities rarely exist in isolation.

## What I Learned for My Hunting

This report taught me to test every related feature endpoint after finding a single IDOR. If and Once the group deletion is vulnerable. I will test further to resource deletion such as user deletion, project deletion or even file deletion.

## References

| CWE-639: Authorization Bypass Through User-Controlled Key | `https://cwe.mitre.org/data/definitions/639.html` |
| --- | --- |
| OWASP: Testing for Insecure Direct Object References | `https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/05-Authorization_Testing/04-Testing_for_Insecure_Direct_Object_References` |
| PortSwigger: IDOR Vulnerabilities | `https://portswigger.net/web-security/access-control/idor` |
