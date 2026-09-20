# [Program] — [Report Title]

## Report Information

**Original Report:** [Link]

**Platform:** HackerOne / Bugcrowd / Intigriti / Other

**Bounty:** $X,XXX  
**Severity:** Critical / High / Medium / Low

**Vulnerability Type:** IDOR / Stored XSS / Business Logic / etc.

**Tags:** #IDOR #AccessControl #H1

---

# Executive Summary

[2-3 sentences explaining:
- What vulnerability was found
- Who/what was affected
- Why it was important]

---

# Vulnerability Overview

## Vulnerability Details

[Technical explanation of the vulnerability.]

Include:
- Where the issue existed
- The vulnerable functionality
- The attack flow
- Required conditions

---

## Root Cause Analysis

[Explaination why the vulnerability existed.]

Examples:
- Missing authorization checks
- Improper input validation
- Incorrect business logic implementation
- Trusting client-side data
- Missing ownership verification

---

# Hunter Analysis

## Original Hunter's Approach

[How the researcher discovered the vulnerability.]

Document:
- Recon approach
- Testing methodology
- Tools used
- Interesting observations
- Researcher's thought process

---

## My Alternative Approach

[How I would approach finding this vulnerability.]

Include:

### Recon
- What would I identify first?
- What endpoints/features would I focus on?

### Testing Strategy
- What assumptions would I test?
- What attack scenarios would I try?

### Tools
- Burp Suite
- Browser DevTools
- Custom scripts
- Other tools

### Payloads / Requests
[Important requests, parameters, or techniques.]

---

# Impact Analysis

## Technical Impact

[What an attacker could technically achievee.]

Examples:
- Access unauthorized data
- Modify another user's resources
- Execute actions without permission

---

## Business Impact

[Real-world consequences for the organization.]

Examples:
- Data exposure
- Account compromise
- Loss of customer trust
- Financial/reputation damage

---

## Scope & Severity

[Estimate the affected area.]
- Number of affected users
- Required privileges
- Attack complexity
- Severity reasoning

---

# Key Lessons & Patterns

## Vulnerability Patterns

- Pattern 1:
- Pattern 2:
- Pattern 3:

## Future Hunting Rules

- 

---

# Personal Reflection

[my personal notes.]
- What surprised me?
- What mistake did the developer make?
- What did I learn?
- How will I apply this during hunting?

---

# References

- Original report:
- Related OWASP category:
- Related research/articles:
