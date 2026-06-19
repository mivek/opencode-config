---
name: writing-plans
description: Load with an approved design to produce a concrete implementation plan. Breaks work into bite-sized tasks (2-5 min each) with exact file paths, key signatures, and verification steps. Maps the file structure before defining tasks. The plan should be executable by someone with no project context.
compatibility: opencode
metadata:
  workflow: planning
  adapted-from: superpowers
---

# Writing plans — turn a design into executable tasks

You have an approved design. Now produce a plan concrete enough that an implementer with **no project context and an aversion to testing** could follow it correctly.

## Before defining tasks: map the file structure

List which files will be created or modified, and what each is responsible for. This catches scope problems early. If the design spans multiple independent subsystems, flag it — suggest splitting into separate plans, one per subsystem, each producing working software on its own.

## Task granularity

Break the work into **bite-sized tasks, roughly 2-5 minutes of work each**. Small tasks prevent over-engineering and make progress verifiable. Each task must have:

- **Exact file path(s)** — `src/main/java/com/example/RangeValidator.java`, not "the validator"
- **Key signatures** — function/method/class shapes. Pseudocode is fine; full bodies are the implementer's job.
- **A verification step** — how does the implementer know this task is done? A test that passes, a command that succeeds, an output that appears.
- **The test approach** — what test proves this task works (per TDD, the test comes first)

## Order by dependency

Each task should build on the previous. The implementer works through them in order. Don't create tasks that require future tasks to be testable.

## Emphasize what NOT to build

State explicit non-goals. YAGNI is a first-class part of the plan — call out the speculative features you are deliberately NOT building.

## Plan format

Save to `docs/plans/YYYY-MM-DD-<feature>.md`. Structure:

```
# <Feature> — Implementation Plan

## Goal
<One sentence.>

## File structure
- **<path>** — <responsibility>
- ...

## Tasks

### Task 1: <name>
- **Files:** <exact paths>
- **Change:** <what, with key signatures>
- **Test first:** <the failing test to write>
- **Verification:** <command / expected outcome>

### Task 2: ...

## Out of scope (YAGNI)
<What this plan deliberately doesn't build.>

## Risks
<Assumptions to validate, things that could go wrong.>
```

## Scope check (backstop)

Before finalizing: if the plan has grown to cover multiple unrelated subsystems, stop. It should have been decomposed during brainstorming. Suggest splitting now rather than executing a monster plan.

## Anti-patterns

- Tasks too big to verify in one step ("implement the feature"). Break them down.
- Vague file references ("update the relevant files"). Name them exactly.
- Skipping the test approach per task. The test is how you know it works.
- Padding the plan with rationale the implementer doesn't need. Signal over volume.
- A monolithic plan for what's really 3 independent features.
