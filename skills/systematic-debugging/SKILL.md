---
name: systematic-debugging
description: Load when encountering any bug, test failure, or unexpected behavior, BEFORE proposing fixes. Enforces a four-phase root-cause process. Phase 1 is a hard gate — no fix until the root cause is understood. RIGID skill. Most needed exactly when you're tempted to guess.
compatibility: opencode
metadata:
  workflow: debugging
  type: rigid
  adapted-from: superpowers
---

# Systematic debugging — root cause before fixes

Core principle: **ALWAYS find the root cause before attempting a fix. Symptom fixes are failure.** Random fixes waste time, mask the real issue, and introduce new bugs.

This is a **rigid skill**. It applies to test failures, runtime bugs, performance problems, build failures — any unexpected behavior. It's most needed precisely when you're under pressure and tempted to just try something.

**Phase 1 is a hard gate: you cannot propose any fix until Phase 1 is complete.**

## Phase 1 — Root cause investigation (the gate)

Goal: understand the failure before touching code.

1. **Read the error completely.** Don't skim. Read the full stack trace, note exact line numbers, read the actual message. The error often contains the exact solution.
2. **Reproduce it reliably.** If you can't reproduce it consistently, that's your first problem to solve. An intermittent bug you can't trigger on demand can't be verified as fixed.
3. **Trace the data flow.** For each component boundary the data crosses: what enters, what exits, is the config/environment propagated correctly? Add temporary diagnostic logging at boundaries if needed. Run once to gather evidence of WHERE it breaks, then analyze.
4. **Find where it actually breaks** — not where the symptom appears. The 500 error surfaces in the controller; the cause might be a connection pool three layers down.

Only when you can name the root cause with evidence do you leave Phase 1.

> 95% of "there's no clear root cause" cases are incomplete investigation, not genuinely mysterious bugs.

## Phase 2 — Pattern analysis

Compare the failing code against working examples. Is there a similar piece of code that works? What's different — environment, config, input, ordering? Subtle differences between the broken path and a working path often reveal the cause.

## Phase 3 — Hypothesis testing

Form ONE hypothesis. Test it with the **minimal** change that would confirm or refute it. One variable at a time. Don't change five things and see if the symptom goes away — you won't know what worked, and you'll cause new bugs.

If the hypothesis is wrong, return to Phase 1 with what you learned. Don't pivot to a second random guess.

## Phase 4 — Implementation (with a test)

Once the root cause is confirmed:
1. Write a test that reproduces the bug (it should fail now — this is the RED of TDD)
2. Fix the root cause (not the symptom)
3. Watch the test pass
4. Verify no regressions (run the surrounding suite)

The first fix sets the pattern — do it right from the start.

## Phase 4.5 — When fixes keep failing

If 3+ fixes have failed, **STOP**. This is not a failed hypothesis — it may be a wrong architecture. Signs you should question the design rather than keep patching:
- Each fix reveals another problem in the same area
- The fix requires special-casing more and more inputs
- You're adding defensive checks at multiple layers for the same issue

When you hit this, discuss with the user before attempting more fixes. The question becomes "should we refactor this, not patch it?"

## Signals that mean "STOP, return to Phase 1"

- You're about to change code you don't fully understand
- You're trying a fix "to see if it helps"
- You can describe the symptom but not the mechanism
- You've made the symptom disappear but can't explain why

## Anti-patterns

- Proposing a fix before completing Phase 1. The gate exists for a reason.
- Guess-and-check: changing things hoping one sticks. Slower than systematic, not faster.
- Fixing the symptom (catch the exception, retry the call) instead of the cause.
- Changing multiple variables at once. You lose the ability to know what worked.
- Declaring "no root cause, must be flaky" without thorough investigation. Usually it's incomplete investigation.
- Skipping the regression test. An untested fix doesn't stick.
