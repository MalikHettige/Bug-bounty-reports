# Bug-bounty-reports

**Self-taught Application Security Researcher from Sri Lanka.** No mentor.
No bootcamp. Just public reports, PortSwigger labs, and 12-hour days.


This repo is how I learn: I take disclosed reports, reverse-engineer the
methodology, and write how *I* would have found each bug. Not summaries.
Not copy-paste. My own recon, my own testing steps, my own mistakes.
Real triage data is the closest thing to ground truth for pattern
recognition.
Real VDP/paid-program submissions live here too, in `live-hunts/`, once
there's something to submit. Lab practice (PortSwigger, HTB) lives in a
separate repo — [Bug-bounty-writeups](https://github.com/MDHettige17BBH/Bug-bounty-writeups).

**Focus areas:** Access Control | Authentication | Business Logic

**Progress:** see [PROGRESS.md](./PROGRESS.md)

## Structure

```
Bug-bounty-reports/
├── README.md
├── PROGRESS.md
├── Disclosed reports-template.md
├── live-hunts/                  ← my own real submissions
│   ├── idor/
│   ├── authentication/
│   └── business-logic/
├── disclosed-report-analysis/   ← other people's published reports, analyzed
│   ├── idor/
│   ├── authentication/
│   └── business-logic/
|   └── race-conditions/
└── lessons-learned/             ← rejections, duplicates, N/A findings — failure analysis
```

## Pre-report research (before writing any analysis)

1. Read the disclosed report (5 min).
2. Open the program site, find the feature, Google the vulnerable
   feature name.
3. Look at public screenshots.
4. Read the program's security page or API docs.
5. Ask: what does a developer do there? (upload apps, add photos, set
   pricing — whatever the feature actually does)
6. Ask: where in that flow would an object reference (photo ID, org ID,
   etc.) appear in a URL?

Create a free account only when necessary, click around, look at URLs.
Ten minutes of clicking teaches more than an hour of reading about the
program.

## Contact

- **X:** [Twitter - where I document the most](https://x.com/MalikHettige)
- **Medium:** [Blog account](https://medium.com/@MalikHettige)
- **HackerOne:** [malikdishan](https://hackerone.com/malikdishan)
- **Discord:** lokimdmischef
- **Email:** malikdishan09@gmail.com
- **Location:** Colombo, Sri Lanka

Open to collaboration, mentorship, and program invitations.
