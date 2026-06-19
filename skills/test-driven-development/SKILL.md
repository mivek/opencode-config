---
name: test-driven-development
description: Load during implementation. Enforces RED-GREEN-REFACTOR — write a failing test first, watch it fail, write minimal code to pass, watch it pass, then refactor. This is a RIGID skill — follow it exactly, do not adapt away the discipline. Applies to Java/Quarkus (JUnit/RestAssured), Next.js (Jest/Vitest), and mobile code.
compatibility: opencode
metadata:
  workflow: tdd
  type: rigid
  adapted-from: superpowers
---

# Test-Driven Development — RED, GREEN, REFACTOR

This is a **rigid skill**. The discipline IS the value. Don't adapt it away because a change seems small.

Core principle: **write the test first.** Then watch it fail. Then write the minimal code to pass. Code written before its test gets deleted and redone test-first.

## The cycle

### RED — write one failing test
Write a single minimal test showing what should happen. Clear name, tests real behavior, one thing.

```java
// Good — Quarkus / JUnit
@Test
void rejectsNegativeRange() {
    assertThrows(IllegalArgumentException.class,
        () -> new RangeValidator().validate(-1));
}
```

The test must describe behavior, not implementation. "rejectsNegativeRange" — yes. "testValidate2" — no.

### Watch it fail — MANDATORY, never skip
Run the test. Confirm it fails, and fails for the **right reason** (the behavior is missing — not a typo, not a compile error in the test itself).
- Test passes already? You're testing existing behavior. Fix the test.
- Test errors (compile/setup)? Fix the error, re-run until it fails correctly.

This step proves the test can actually catch the bug. A test you never watched fail is worthless — it might pass regardless of the code.

### GREEN — minimal code to pass
Write the **simplest** code that makes the test pass. Not the elegant version, not the general version — the minimal one. Resist adding anything the test doesn't require (YAGNI).

### Watch it pass
Run it. Confirm green. If it doesn't pass, fix the code (not the test, unless the test was wrong).

### REFACTOR — clean up, tests stay green
Now improve the code: naming, structure, duplication (DRY). Re-run tests after each change. Tests must stay green throughout. If a refactor breaks a test, the refactor was wrong or the test was too coupled to implementation.

### Commit
One logical change = one commit, with tests included. The test and the code it validates travel together.

## Stack-specific notes

### Java / Quarkus
- `@QuarkusTest` for integration-level (boots the app), plain JUnit for pure unit tests
- RestAssured for HTTP: `given().when().get("/api/x").then().statusCode(200)`
- Run one test: `./mvnw test -Dtest=ClassName#methodName`
- Mock externals with `@InjectMock` (needs `quarkus-junit5-mockito`)

### Next.js / mobile (Jest / Vitest)
- Test user-visible behavior, not component internals
- For React Native, test logic and hooks in isolation; reserve E2E for flows

## Anti-patterns

- Writing code before the test. The test comes first, always.
- Skipping "watch it fail." You don't know the test works until you've seen it fail.
- Writing tests after the code that conveniently match whatever the code does. Test against the spec, not the implementation.
- Over-mocking until the test verifies nothing real. Test behavior.
- Making a failing test pass by weakening the test. Fix the code.
- Adding speculative generality in GREEN. Minimal code. Generalize only when a second test demands it.
