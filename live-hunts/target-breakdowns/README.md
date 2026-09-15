# Target Breakdown README

## What this is

A repeatable framework for going deep on a single target — mapping its
expected auth behavior and business logic *before* testing what it
actually does. Reserved for targets that survive initial recon and are
worth a serious time investment, not every target.

## Why this exists

Recon shows what's there. A breakdown forces you to state what *should*
happen before testing what *does* — that gap is where real bugs live.
Section 7 turns each breakdown into a pattern log for your own future
hunts, same value as reading disclosed reports, but from your own testing.

## Goal: 6 breakdowns by January 2027

Deliberately small — the goal is depth, not volume. Six thorough
breakdowns build auth-matrix and business-logic thinking that fast recon
alone doesn't teach. After January, breakdowns continue without a fixed
count. Progress tracked separately in `PROGRESS.md`.

## Why it matters for the $10K target

[Confirm framing] Bugs worth real bounty money tend to live in the gap
between assumed and actual behavior — shallow recon rarely finds them.
Six deep dives by January is a bet that depth on fewer targets beats
breadth across many.

## How to use it

1. Copy into `0X-<target-name>.md`
2. Fill Sections 1–4 before testing (asset inventory, auth matrix,
   business logic)
3. Run Section 5 as a checklist every time, even on "boring" targets
4. Rank Section 6 by impact × likelihood — doubles as severity-writing practice
5. Keep Section 7 updated as you go, not just at the end
6. Update `PROGRESS.md` when a breakdown counts as [define the bar]

## Files here

- `README.md` — this file
- `PROGRESS.md` — goal tracker
- `0X-<target-name>.md` — individual breakdowns
