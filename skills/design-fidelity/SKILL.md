---
name: design-fidelity
description: UI fidelity workflow for any stack. Load when implementing screens from visual drafts. Screen image files live in assets/screens/ (or assets/designs/) in the repo — no manual image pasting needed. Implements screen-by-screen with an inline verify loop: spec → implement → verify → fix, then move to the next screen. Uses @design-interpreter (Mimo v2.5, native vision) to read draft files and produce framework-neutral specs, @implementer (deepseek-flash) to build, and @ui-verifier (Mimo v2.5, native vision) to compare renders vs draft files. Works for Flutter, React, Vue, React Native, and any other UI stack. Zero new dependencies — Mimo v2.5 is already in the config.
compatibility: opencode
metadata:
  workflow: design-fidelity
  adapted-from: opencode-config superpowers methodology
---

# Design-fidelity workflow — implement UI screens that match the drafts

You have screen draft files in the repo (e.g. `assets/screens/login.png`) and need to build a UI that faithfully matches them. The coding agents (DeepSeek) cannot see images. This skill routes the two vision-boundary steps to **Mimo v2.5** (native vision), which reads the draft files directly from the repo — no manual image pasting needed.

```
assets/screens/<screen>.png  (file in repo)
  └─ @design-interpreter (Mimo v2.5, vision)
       → reads file → docs/designs/<screen>.spec.md (neutral spec) + tokens.md
           └─ @implementer (DeepSeek, text)
                → token anchor + screen from spec
                    └─ @ui-verifier (Mimo v2.5, vision)
                         → reads file + rendered screenshot → diff → findings
                             └─ @implementer fixes → @ui-verifier re-checks → human sign-off → NEXT SCREEN
```

**The loop is per screen.** Each screen is fully specced, implemented, verified, and signed off before moving to the next. There is no "implement everything, then verify everything" batch.

**Go-budget note:** vision costs land only on `@design-interpreter` (once per screen) and `@ui-verifier` (a few passes per screen). The bulk of implementation stays on `@implementer` (deepseek-v4-flash — cheapest Go model). Budget-efficient by design.

---

## Before starting: discover the screens and detect the stack

Run `glob("assets/screens/**")` (or `assets/designs/**`) to list all draft files. This is the implementation queue. Work through them in a logical UI order (e.g. onboarding → main tabs → detail screens).

Also detect the project's UI stack so `@design-interpreter` can include stack hints and `@ui-verifier` picks the right test runner:

| Signal | Stack |
|---|---|
| `pubspec.yaml` present | Flutter |
| `playwright.config.*` present | Web (Playwright) |
| `package.json` with `react-native` or `expo` | React Native |
| `package.json` otherwise | Web (React / Vue / plain HTML) |

Pass the detected stack to `@design-interpreter` when delegating — it adds stack-specific hints to the neutral spec.

---

## Step 1 — Design system first (once, before any screen)

**Always implement the token/theme anchor before the first screen.** If every component uses correct tokens, the screens assemble correctly.

Dispatch `@design-interpreter` on ALL screen files first to build the token inventory:
- `docs/designs/tokens.md` — color palette, type scale, spacing scale, radii, elevation.

Then tell `@implementer` to create the project's token anchor in the stack's idiom:

| Stack | Token anchor |
|---|---|
| Flutter | `lib/theme/app_theme.dart` with `ThemeData` + token constant files (`app_colors.dart`, `app_spacing.dart`, …) |
| React / CSS | CSS custom properties in `src/styles/tokens.css` (or Tailwind config extension in `tailwind.config.js`) |
| React / JS theme | A typed theme object in `src/theme/index.ts` with color, type, spacing exports |
| React Native | `src/theme/index.ts` + `StyleSheet` constants |
| Vue | CSS variables in `src/assets/tokens.css` or a composable |

**Gate:** run the project's lint/type-check (e.g. `flutter analyze`, `pnpm run typecheck`, `pnpm run lint`) and confirm it passes before implementing individual screens.

---

## Step 2 — Per-screen loop (repeat for each screen)

### 2a. Spec the screen

Dispatch `@design-interpreter` on `assets/screens/<screen>.png`. Pass the detected stack. It reads the file directly (no pasting needed) and produces:
- `docs/designs/<screen>.spec.md` — neutral component tree, layout (flex direction/justify/align/gap, padding/margins in **px**), exact colors (hex), typography (px/weight), iconography, states, interaction notes, **plus optional stack hints** if the stack is known.

**Gate:** if `@design-interpreter` flags ambiguities, resolve them before implementation. Don't let `@implementer` guess.

**Figma shortcut (optional):** if the draft is in Figma with Dev Mode, use the Figma MCP instead. It gives exact measurements with no vision step needed.

### 2b. Implement the screen

Dispatch `@implementer` with `docs/designs/<screen>.spec.md` as input:
- Build the component/widget using only token constants from the project's theme — no inline magic numbers.
- The neutral spec is the source of truth; stack hints are advisory.
- Run the project's lint/type-check — must pass before handing to `@ui-verifier`.

### 2c. Verify: rendered screen vs draft

Dispatch `@ui-verifier` with:
- The screen name and the draft file path (`assets/screens/<screen>.png`).
- `@ui-verifier` detects the stack, picks the right test runner, reads the draft file directly, captures the rendered output, and compares both with native vision.

It runs two checks:
- **Visual regression** — compares against committed baseline (golden PNG for Flutter, Playwright snapshot for web). Catches unintended drift.
- **Design fidelity** — direct visual comparison of rendered vs draft; reports concrete deviations (hex colors, px measurements, missing elements).

### 2d. Fix loop

Repeat: **`@implementer` fixes the reported deviations → `@ui-verifier` re-checks** until deviations are minor (ideally zero critical, only cosmetic).

**Baseline update:** `@ui-verifier` will ask before updating baselines (`--update-goldens` / `--update-snapshots`). Only approve when the current render is correct.

### 2e. Human sign-off on this screen

Before moving to the next screen, you are the final visual judge. `@ui-verifier` presents the rendered screenshot and the draft file side by side. Check: colors (exact hex, not approximate), spacing, typography, component shapes, element presence. If something is off that the automated diff missed, describe it concretely → send back to `@implementer`.

Once you sign off → move to the next screen in the queue.

---

## Image-reading mechanics

`@design-interpreter` and `@ui-verifier` run on Mimo v2.5 (native vision). They read draft files from the repo using the `read` tool — the same way they read code files, except the content is an image. **No manual image pasting is needed as long as the draft files are in `assets/screens/` (or `assets/designs/`).**

If the `read` tool does not pass image bytes to the model in your opencode version (edge case — check by running `@design-interpreter` on a single file first), fall back to pasting the image while `@design-interpreter` or `@ui-verifier` is the active agent. Never paste to `@implementer` — it's text-only.

---

## Checklist (per screen)

- [ ] Screen draft file present: `assets/screens/<screen>.png`
- [ ] Stack detected and passed to agents
- [ ] Token anchor committed (one-time gate)
- [ ] Project lint/type-check passing before first screen
- [ ] `@design-interpreter` produced `docs/designs/<screen>.spec.md` — ambiguities resolved
- [ ] `@implementer` built screen from spec — no magic numbers, lint/type-check passes
- [ ] `@ui-verifier` run — visual baseline created
- [ ] Fidelity deviations listed and addressed
- [ ] Human sign-off — screenshot matches the draft
- [ ] Move to next screen

## Anti-patterns

- Implementing multiple screens before verifying any — verify each screen before starting the next.
- Implementing a screen before the token anchor — every magic number is a future inconsistency.
- Letting `@implementer` guess values flagged as ambiguous in the spec — go back and read the draft.
- Accepting vague deviation reports ("looks a bit off") — demand hex values and px measurements.
- Updating visual baselines silently after every change — only update when the render is approved.
- Skipping the visual baseline — without a committed baseline, the regression check is blind in subsequent passes.
- Passing the wrong stack to `@design-interpreter` — stack hints must match the target framework or be omitted.
- Treating stack hints as the spec — the neutral spec is the source of truth.
