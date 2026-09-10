---
program:     <Program Name> (Platform — VDP/BBP)
platform:    <HackerOne / Bugcrowd / Intigriti / etc.>
category:    <authentication / idor / business-logic / etc.>
cwe:         CWE-XXX — <Official CWE Name>
date_found:  YYYY-MM-DD
status:      <submitted / triaged / accepted / duplicate / rejected>
report:      <link to platform report, once available>
---

# <Program> — <Short Descriptive Title of the Vulnerability>

## Scope and setup

- What scope that were given (domains, wildcard, exclusions)
- Program characteristics worth noting (new program, response efficiency, reward range if public)
- Recon methodology used (tools, passive vs active, what me ran)
- **Dead ends tested and ruled out** — list them explicitly with why they failed or were excluded. This proves rigor, not just luck.

## Finding

- Exact steps to reach the vulnerable behavior
- What was sent, what I expected, what actually happened
- The specific technical detail that makes it a bug (e.g. comparison against sibling/similar endpoints, a state that shouldn't be reachable, a check that's missing)

## Root cause

- The underlying technical reason the bug exists (framework, code pattern, likely developer mistake)
- Include a code snippet if me can infer/see the pattern (e.g. missing decorator/attribute, broken validation logic)
- **CWE justification** — explicitly state why this CWE and not an adjacent one. This is the part that shows real understanding, not just labeling.

## Impact

- What an attacker could actually do with this — be concrete, not generic ("could lead to X" is weaker than "attacker can inject Y into Z")
- Which CIA property is affected (confidentiality/integrity/availability) and why
- Note explicitly what me did NOT confirm (be honest about the edge of my access/visibility — this builds credibility, doesn't hurt it)
- CVSS vector reasoning if me're scoring it, tied to what me actually observed

## The pattern to generalize

- The reusable lesson — what should me look for again in future hunts because of this finding?
- This section makes every report also a training document for my future self

## Outcome

- Final triage result and platform response
- If duplicate/rejected: note it factually, no self-flagellation
- Link to lessons-learned entry if I keep one
