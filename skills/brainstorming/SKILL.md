---
name: brainstorming
description: Load BEFORE any code or plan for a new feature or non-trivial change. Refines a rough idea into an agreed design through questions and 2-3 explored approaches, then gates on explicit user approval. The design becomes the input to writing-plans. Prevents jumping from idea straight to code.
compatibility: opencode
metadata:
  workflow: design-first
  adapted-from: superpowers
---

# Brainstorming — turn a rough idea into an approved design

You have a request, not yet a design. Your job is to **understand the real problem and agree
on an approach** before a single line of code or a plan is written. This is the first gate of
the design-first methodology; `[[writing-plans]]` is the next step, once the design is approved.

Code written against a misunderstood problem is the most expensive code there is. Brainstorming
is cheap insurance.

## When this applies

- A new feature, or any change that touches more than one file / is more than a one-liner.
- Anything where the "what" or the "why" is even slightly fuzzy.

**Skip it** for: one-line fixes, typo/rename, pure read-only investigation. Use judgment — when
in doubt, do it; it's a few questions, not a ceremony.

## 1. Understand before proposing

Restate the request in your own words, then surface what you *don't* yet know. Ask the user
**a few targeted questions at a time — never a wall of twenty.** Prioritize questions whose
answers would actually change the design:

- What problem is this really solving? Who hits it, how often?
- What does "done" look like — the observable outcome?
- Hard constraints: existing patterns to match, perf, security, deadlines, things you must NOT break?
- What's explicitly out of scope?

Stop asking once you can describe the solution space confidently. Don't interrogate for its own sake.

## 2. Explore 2-3 distinct approaches

Not one idea elaborated — **genuinely different approaches**, each with honest tradeoffs.
Delegate exploration/research to subagents if you need facts about the codebase or a library
first (e.g. `@explorer`, `@researcher`). For each approach:

- **What it is** — one or two sentences.
- **Pros / cons** — be honest, including the cons of the one you like.
- **Cost / risk** — complexity, blast radius, things that could go wrong.

Then **recommend one** and say why. A recommendation, not a menu dump.

## 3. Present the design for approval

Present it in labelled sections so the user can react precisely:

```
## Problem
<What we're solving and why, 1-3 sentences.>

## Approaches considered
1. <name> — <one-liner>. Pros: … Cons: …
2. <name> — …
(3. …)

## Recommendation
<The chosen approach and the reasoning. Call out the key tradeoff accepted.>

## Non-goals (YAGNI)
<What this deliberately will NOT do.>

## Open questions
<Anything still unresolved that the user should weigh in on.>
```

## 4. HARD GATE — do not proceed without approval

**Do not write a plan and do not write code until the user explicitly approves the design.**
End with: **"Reply 'approve' to lock this design, or tell me what to change."** If the user
pushes back, revise and re-present — don't bulldoze toward implementation.

## 5. Hand off

Once approved:
- Save a short design note (e.g. `docs/designs/YYYY-MM-DD-<feature>.md`) capturing the problem,
  chosen approach, and non-goals — this is the input the planner builds on.
- This is the point where a **git worktree** is created (see `[[worktree-workflow]]`), *before*
  planning — so the plan and code land in isolation. Coordinate: create exactly one worktree.
- Hand the approved design to `[[writing-plans]]` (the planner) to turn it into executable tasks.

## Anti-patterns

- Jumping from idea to plan/code without agreeing on the problem. This skill exists to stop that.
- Presenting one approach as if it were the only option. Explore alternatives.
- A wall of questions up front. A few that matter, then iterate.
- Burying the recommendation in a neutral list. Take a position.
- Proceeding past the gate because the design "seems obvious." Get the explicit approval.
- Creating a second worktree when planning starts. One per feature, created here.
