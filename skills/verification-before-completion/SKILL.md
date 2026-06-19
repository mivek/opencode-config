---
name: verification-before-completion
description: Load before declaring any task done. Enforces that every change is demonstrated to actually work — not asserted. "It works on my machine" or "this should fix it" is never the end. Run the actual verification and show the evidence.
compatibility: opencode
metadata:
  workflow: verification
  adapted-from: superpowers
---

# Verification before completion — evidence over claims

Core principle: **every change must be demonstrated to work, not asserted.** "This should fix it", "it works now", "the logic looks correct" are not completion. Running the verification and showing the result is completion.

## Before saying "done"

Run through this checklist. Each item needs **evidence**, not a claim.

1. **Did the tests run, and pass?** Not "the tests should pass" — paste the actual test output. Count: how many ran, how many passed.
2. **Did the build succeed?** Run the actual build/compile. `./mvnw verify`, `npm run build`, `tsc --noEmit` — whatever applies. Show it succeeding.
3. **Does the original problem actually go away?** If you fixed a bug, reproduce the original failing case and show it now works. If you added a feature, demonstrate the feature doing what was asked.
4. **Did you check for regressions?** Run the surrounding test suite, not just your new test. A fix that breaks three other things isn't done.
5. **Linter/static checks clean?** `eslint`, `ruff`, `cargo clippy`, `go vet` as applicable.

## The standard

A claim like "I've implemented the rate limiter" is incomplete. The complete version:

> Implemented the rate limiter in `RateLimitFilter.java`. Ran `./mvnw test` — 47 tests pass (3 new). The new test `rejectsAfterLimit` confirms requests beyond the window get a 429. Full `./mvnw verify` succeeds. No regressions in the existing suite.

The difference: the second version shows it works. The first hopes it works.

## When verification fails

If verification reveals the change doesn't actually work — good, you caught it before the user did. Return to the relevant skill:
- Test fails unexpectedly → `systematic-debugging`
- The approach was wrong → back to the plan
Don't paper over a failing verification by weakening the check.

## Anti-patterns

- "This should work now" as a completion statement. Run it and show it.
- Running only your new test, not the surrounding suite. Regressions hide there.
- Claiming a build passes without running it.
- Marking a bug fixed without reproducing the original failing case.
- Declaring victory on partial evidence because the change "looks right". Looking right and being right are different.
