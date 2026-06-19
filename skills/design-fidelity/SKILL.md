---
name: design-fidelity
description: Flutter UI fidelity workflow. Load when implementing screens from visual drafts. Screen image files live in assets/screens/ (or assets/designs/) in the repo — no manual image pasting needed. Implements screen-by-screen with an inline verify loop: spec → implement → verify → fix, then move to the next screen. Uses @design-interpreter (Kimi, native vision) to read draft files and produce specs, @implementer (deepseek-flash) to build, and @ui-verifier (Kimi, native vision) to compare renders vs draft files. Zero new dependencies — Kimi K2.6 is already in the config.
compatibility: opencode
metadata:
  workflow: design-fidelity
  adapted-from: opencode-config superpowers methodology
---

# Design-fidelity workflow — implement Flutter screens that match the drafts

You have screen draft files in the repo (e.g. `assets/screens/login.png`) and need to build a Flutter UI that faithfully matches them. The coding agents (DeepSeek) cannot see images. This skill routes the two vision-boundary steps to **Kimi K2.6** (native vision), which reads the draft files directly from the repo — no manual image pasting needed.

```
assets/screens/<screen>.png  (file in repo)
  └─ @design-interpreter (Kimi, vision) → reads file → docs/designs/<screen>.spec.md + tokens.md
       └─ @implementer (DeepSeek, text) → ThemeData/tokens → screen widget from spec
            └─ @ui-verifier (Kimi, vision) → reads file + rendered golden → diff → findings
                 └─ @implementer fixes → @ui-verifier re-checks → human sign-off → NEXT SCREEN
```

**The loop is per screen.** Each screen is fully specced, implemented, verified, and signed off before moving to the next. There is no "implement everything, then verify everything" batch.

**Go-budget note:** vision costs land only on `@design-interpreter` (once per screen) and `@ui-verifier` (a few passes per screen). The bulk of implementation stays on `@implementer` (deepseek-v4-flash — cheapest Go model). Budget-efficient by design.

---

## Before starting: discover the screens

Run `glob("assets/screens/**")` (or `assets/designs/**`) to list all draft files in the repo. This is the implementation queue. Work through them in a logical UI order (e.g. onboarding → main tabs → detail screens).

---

## Step 1 — Design system first (once, before any screen)

**Always implement `ThemeData` and token constants before the first screen.** If every widget uses correct tokens, the screens assemble correctly.

Dispatch `@design-interpreter` on ALL screen files first to build the token inventory:
- `docs/designs/tokens.md` — color palette, type scale, spacing scale, radii, elevation.

Then tell `@implementer`:
1. Create `lib/theme/app_theme.dart` with `ThemeData` populated from `docs/designs/tokens.md`.
2. Create token constant files (`app_colors.dart`, `app_spacing.dart`, etc.) — no magic numbers in widgets.
3. Verify: `flutter analyze` — no errors.

**Gate:** do not start implementing individual screens until the design system is committed and passing `flutter analyze`.

---

## Step 2 — Per-screen loop (repeat for each screen)

### 2a. Spec the screen

Dispatch `@design-interpreter` on `assets/screens/<screen>.png`. It reads the file directly (no pasting needed) and produces `docs/designs/<screen>.spec.md` — component tree, layout (rows/columns, alignment, padding/margins in logical px), exact colors (hex), typography (sp/weight), iconography, states, interaction notes.

**Gate:** if `@design-interpreter` flags ambiguities, resolve them before implementation. Don't let `@implementer` guess.

**Figma shortcut (optional):** if the draft is in Figma with Dev Mode, use the Figma MCP instead. It gives exact measurements with no vision step needed.

### 2b. Implement the screen

Dispatch `@implementer` with `docs/designs/<screen>.spec.md` as input:
- Build the widget, using only token constants from `lib/theme/` — no inline magic numbers.
- Run `flutter analyze` — must pass before handing to `@ui-verifier`.

### 2c. Verify: rendered screen vs draft

Dispatch `@ui-verifier` with:
- The screen name and the draft file path (`assets/screens/<screen>.png`).
- `@ui-verifier` reads the draft file directly, runs the golden test, reads the rendered PNG, and compares both with native vision.

It runs two checks:
- **Golden regression** — pixel diff vs committed baseline (catches drift).
- **Design fidelity** — direct visual comparison of rendered vs draft; reports concrete deviations (hex colors, dp measurements, missing elements).

### 2d. Fix loop

Repeat: **`@implementer` fixes the reported deviations → `@ui-verifier` re-checks** until deviations are minor (ideally zero critical, only cosmetic).

**Golden baseline update:** `@ui-verifier` will ask before running `--update-goldens`. Only approve when the current render is correct.

### 2e. Human sign-off on this screen

Before moving to the next screen, you are the final visual judge. `@ui-verifier` presents the golden PNG and the draft file side by side. Check: colors (exact hex, not approximate), spacing, typography, component shapes, element presence. If something is off that the automated diff missed, describe it concretely → send back to `@implementer`.

Once you sign off → move to the next screen in the queue.

---

## Image-reading mechanics

`@design-interpreter` and `@ui-verifier` run on Kimi K2.6 (native vision). They read draft files from the repo using the `read` tool — the same way they read `.dart` or `.md` files, except the content is an image. **No manual image pasting is needed as long as the draft files are in `assets/screens/` (or `assets/designs/`).**

If the `read` tool does not pass image bytes to the model in your opencode version (edge case — check by running `@design-interpreter` on a single file first), fall back to pasting the image while `@design-interpreter` or `@ui-verifier` is the active agent. Never paste to `@implementer` — it's text-only.

---

## Checklist (per screen)

- [ ] Screen draft file present: `assets/screens/<screen>.png`
- [ ] Design system (`ThemeData` + token constants) committed (one-time gate)
- [ ] `@design-interpreter` produced `docs/designs/<screen>.spec.md` — ambiguities resolved
- [ ] `@implementer` built screen from spec — no magic numbers, `flutter analyze` passes
- [ ] `@ui-verifier` run — golden baseline created
- [ ] Fidelity deviations listed and addressed
- [ ] Human sign-off — screenshots match the draft
- [ ] Move to next screen

## Anti-patterns

- Implementing multiple screens before verifying any — verify each screen before starting the next.
- Implementing a screen before the design system — every magic number is a future inconsistency.
- Letting `@implementer` guess values flagged as ambiguous in the spec — go back and read the draft.
- Accepting vague deviation reports ("looks a bit off") — demand hex values and dp measurements.
- Running `--update-goldens` silently after every change — only update the baseline when the render is approved.
- Skipping the golden baseline — without a committed baseline, the regression check is blind in subsequent passes.
