---
name: mockup
description: Use when a plan exists and the feature has a UI — creates design boards and static HTML mockups before implementation begins. Triggers on "mockup", "mock up the UI", "design first", or after /workflows:plan for UI features.
---

# HTML Mockup Workflow

Create static HTML mockups between planning and implementation. Two phases: choose a visual direction, then mock the UI.

## Input

Requires a plan file path. Read the plan to extract:
- Feature description and acceptance criteria
- UI-related requirements (screens, components, interactions)
- User types and key flows

If no plan file is provided, ask the user for one or check `docs/plans/` for recent plans.

## Phase 1: Design Boards

Present **3-5 design board options** as self-contained HTML files. **Each board must represent a fundamentally different aesthetic direction** — not variations on a theme.

### Ensuring Distinct Boards

Before generating, define each board's **mood/vibe in one phrase** and verify they don't overlap. Aim for contrast across these axes:

| Axis | Example spread |
|------|---------------|
| **Mood** | Playful & warm vs. Corporate & trustworthy vs. Dark & editorial vs. Minimalist & zen vs. Bold & brutalist |
| **Color temperature** | Warm earth tones vs. Cool blues/grays vs. Vibrant saturated vs. Muted pastels vs. High-contrast monochrome |
| **Energy level** | Calm and restrained vs. Energetic and dynamic vs. Luxurious and slow |
| **Era/influence** | Modern SaaS vs. Editorial/magazine vs. Retro/vintage vs. Futuristic/glassmorphism vs. Organic/handcrafted |

**Rule: If you can swap two boards' color palettes and they'd feel the same, they aren't distinct enough. Redesign.**

Each board shows:

- **Board title and mood** (e.g. "Warm Editorial" — a 2-3 word vibe label plus one sentence describing the feeling)
- **Color palette** (primary, secondary, accent, background, text colors — show swatches with hex values)
- **Typography** (font pairings for headings and body, sizes, weights — use Google Fonts via CDN). Each board should use **different font families**, not the same fonts in different weights.
- **Spacing and rhythm** (compact vs airy, grid approach)
- **Visual tone** (rounded vs sharp corners, shadows vs flat, border styles, texture)
- **A sample component** rendered in that style (e.g. a card, button row, or small form) so the direction feels tangible — this component should look and *feel* completely different across boards

**Output:** Write each board to a temp file and open in browser:

```
/tmp/mockup-board-1.html
/tmp/mockup-board-2.html
...
```

Use `open /tmp/mockup-board-1.html` (macOS) to show each one.

**Ask the user** which board they prefer, or if they want to mix elements from multiple boards. Iterate until they pick a direction.

## Phase 2: UI Mocks

Using the chosen design direction, create **3-5 static HTML mockups** of the application UI. Each mock should:

- Be a single self-contained HTML file (inline CSS, Google Fonts via CDN, no build tools)
- Represent a **different layout or structural approach** to the same feature — not just color variations
- Include realistic placeholder content (not "Lorem ipsum" — use content that fits the domain)
- Cover the primary user flow from the plan's acceptance criteria
- Be responsive (use flexbox/grid, test at mobile and desktop widths)
- Include light interactivity with vanilla JS where it aids understanding (tab switches, modals, hover states)

**Output:** Write mocks to temp files and open:

```
/tmp/mockup-v1.html
/tmp/mockup-v2.html
...
```

**Ask the user** which mock they prefer or what to combine/change. Iterate until approved.

## Saving the Result

When the user approves a final mock:

1. Save the approved design board to `docs/plans/YYYY-MM-DD-<feature>-design-board.html` (same date/name prefix as the plan)
2. Save the approved mock to `docs/plans/YYYY-MM-DD-<feature>-mockup.html`
3. Note in the plan file (append a `## Mockup` section) with a relative link to the mockup

## Post-Mockup Options

Ask the user:

1. **Iterate further** — refine the approved mock
2. **Start `/workflows:work`** — begin implementing with the mock as reference
3. **Done for now** — save and stop

## Guidelines

- Keep HTML simple and scannable — the mock is a communication tool, not production code
- Prefer system fonts as fallbacks, but use Google Fonts CDN for the design font choices
- Use CSS custom properties (`--color-primary`, etc.) so the design tokens are visible and easy to reference during implementation
- Do not use any framework, bundler, or npm package — pure HTML/CSS/JS only
- Each file should be under 500 lines — if it's getting longer, simplify
