# Main session: work queue

The main session owns the design documents, the roadmap and these queues; it reviews and merges stream PRs and
settles their design deviations. Start each session from [`../../HANDOVER.md`](../../HANDOVER.md) and [`main-handbook.md`](main-handbook.md).

## Standing

- Review every stream PR (live in the browser where it touches a page), settle its "Design deviations" in the design,
  merge when ready. Keep [`../roadmap.md`](../roadmap.md) and the stream queues current.
- New work is written as a queue item (goal, paths, done-when, waits on, read) in the owning stream's file.
- **Operator pages without project vocabulary** (M-22 S8; user, 2026-11-20): whoever edits a page under `docs/`
  that an operator reads takes the phase names, item IDs and issue numbers out of what they touch. No pass of its
  own.

## Queue

Finished items move to [`../archive/work/`](../archive/work/) at each release; find one by its ID.

### [ ] M-4 The live block
- **Do:** run it in the roadmap's order with the streams and the user; the three-way gold-set run and the hand
  grading use `scripts/grading/` (README there).
- **Done when:** the go-live gate is met, G1 is finished and the pilot is live; the roadmap moves P1b to Done.
