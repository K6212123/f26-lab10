# RACE.md

One section per milestone. Fill each one in order as you go.

The prediction in milestone 1 is written BEFORE you run your race test. A TA
may ask about it. Commit it on its own so the order is visible.

Keep it short and specific. Point at methods, call sites, and the invariant.

---

## Milestone 1: from the ticket to a test that fails every time

### Suspect list

Everything your review turned up that touches shared state, with a one-line
verdict for each. Add or remove rows as needed.

| # | What you found (method, field, or class) | Verdict, in one line |
|---|---|---|
| 1 | `BookingService.book`: `hasOverlap` then `insert` | Race: each call is atomic, but the check-and-insert pair is not. |
| 2 | `InMemoryBookingStore.bookings` | Safe per method: access is synchronized and snapshots do not expose the list. |
| 3 | `BookingService.sequence` | Safe: `AtomicLong` gives concurrent callers distinct ids. |
| 4 | `BookingStats` counters | Racy by design, but approximate counters never control booking decisions. |
| 5 | `BookingService.catalog` | Safe: built during construction and read-only afterward. |
| 6 | `cancelAndPromote` | Safe: cancellation and promotion share one synchronized store call. |

**Which one explains the ticket, and how you know.** Tie it to the invariant
the ticket names.

`BookingService.book` explains it: two callers can both observe no overlap before
either inserts. Both then insert `CONFIRMED` bookings, violating the invariant
that a room has at most one confirmed booking covering any minute.

### Predicted interleaving (write this before you run your race test)

Two threads, one row per step, in the form from lecture. Describe what each
thread does at that step and leave the other column empty. Add or remove rows
as your prediction needs.

| Step | Thread A | Thread B |
|---|---|---|
| t1 | `hasOverlap` returns `false`; pause before `insert` | |
| t2 | | `hasOverlap` returns `false` |
| t3 | | Insert confirmed booking B |
| t4 | Resume and insert confirmed booking A | |
| | **End state:** two overlapping `CONFIRMED` bookings | |

If the run turned out different from your prediction, add one line on what
you missed. Leave the table as you first wrote it.

### Your race test

**How you aimed `PausingStore`, and the result.** Method name and call index, why
that pair pins the schedule you predicted, and the output of two separate
runs (assertion failure and log lines).

Aim: `hasOverlap`, call 1. The gate parks A after its check returns `false` but
before `book` can insert. B then checks and inserts before A is released, pinning
the predicted schedule. Both runs were red:

```text
Run 1
2026-11-05T23:22:29.332Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
2026-11-05T23:22:29.336Z INFO booking.confirmed booking=bk-2 room=WEH-5409 start=540 end=600
AssertionFailedError: a room may have at most one confirmed booking covering any minute
==> expected: <1> but was: <2>

Run 2
2026-11-05T23:22:41.956Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
2026-11-05T23:22:41.961Z INFO booking.confirmed booking=bk-2 room=WEH-5409 start=540 end=600
AssertionFailedError: a room may have at most one confirmed booking covering any minute
==> expected: <1> but was: <2>
```

---

## Milestone 2: the patch a teammate sent

**Your milestone-1 test on the patched code.** Green or red. Paste the result.

**The re-aimed run.** Method name, call index, the pasted result, and one line
on why that aim tests the patch rather than sidestepping it.

**Your verdict.** Accept or reject, and what your milestone-1 result does and
does not show about the patch.

---

## Milestone 3: the fix you would ship

**Which option, and what you changed.** Name the lock or the atomic operation
and the two operations it now covers. Paste the diff of your change and the
passing run of your milestone-1 test and the full suite.

**Is your milestone-1 test still testing anything?** After your fix, describe
what that test now exercises. Did `PausingStore` pause a call during the green run? How do
you know?

**What runs one at a time now that ran in parallel before.** Name concretely which callers
queue behind which. "Nothing, it is just correct now" does not count as an
answer.

**What you left unfixed.** For anything your review flagged that you chose not
to fix, one line each on why.
