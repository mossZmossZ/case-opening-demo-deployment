# CLAUDE.md

Behavioral guidelines to reduce common LLM coding mistakes. Merge with project-specific instructions as needed.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

**These guidelines are working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes.

---

## 5. Project-Specific Rules (Zenith Case Opening Demo)

### Navigation
- Navigation is a `screen` state string in `App.jsx` — **no React Router**. Adding a screen = add a `screen === 'xxx'` case and pass an `onXxx` callback prop. Never install a routing library without CEO approval.

### Public Name Display
- All player names shown in public UI (Live Drops, Leaderboard) **must** be passed through `censorName()` from `client/src/lib/utils.js`. Never display raw names outside admin-only views.

### API Endpoints
- Ask CEO before adding any new public (`/api/game/*`) or admin (`/api/admin/*`) endpoints.
- New game routes go in `server/src/routes/game.js` **before** `export default router`.

### UI Standards
- All interactive elements must have visible `focus-visible:ring-*` states — never `outline-none` without a replacement.
- Decorative Material Symbols icons must have `aria-hidden="true"`.
- Use web-design-guidelines skill (`/web-design-guidelines`) when building or modifying screens.

### Ask CEO Before
- Changing DB schema (models in `server/src/models/`)
- Adding npm dependencies
- Modifying user flow order (Welcome → Game → Result → Summary)
- Changing deployment config (Docker Compose, nginx, env vars)
- Altering admin auth logic