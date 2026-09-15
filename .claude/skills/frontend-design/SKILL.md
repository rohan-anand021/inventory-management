---
name: frontend-design
description: Redesign a Vue 3 application's UI into a modern SaaS-style interface with a vertical left sidebar (instead of a top nav bar), consistent spacing, and a polished professional look. Use when asked to modernize, restyle, or "SaaS-ify" a Vue app's layout.
---

# Frontend Design: SaaS-Style Vue 3 Redesign

Converts a Vue 3 app's top-nav layout into a modern SaaS-style layout: a fixed vertical sidebar for navigation plus a light content area for everything else. Apply this to the root layout component (usually `App.vue`) and adjust any sibling components whose positioning assumes a top nav bar (e.g. sticky filter/toolbars using `top: <old-nav-height>`).

## Layout Structure

```
┌──────────┬─────────────────────────────┐
│          │  top-bar (light, sticky)    │
│ sidebar  ├─────────────────────────────┤
│ (dark,   │  page toolbar (optional)    │
│  fixed)  ├─────────────────────────────┤
│          │  main content (cards, etc.) │
└──────────┴─────────────────────────────┘
```

- **Sidebar**: `position: fixed; left: 0; top: 0; height: 100vh`, fixed width (200–260px). Contains the logo/product name at top, then a vertical list of nav links.
- **Content area**: `margin-left: <sidebar-width>`, full remaining width. Holds a slim top bar (user/profile/settings controls) and the routed page content below it.
- Anything with `position: sticky` that used to sit below a horizontal top nav must have its `top` offset updated to match the new top-bar height (not the old nav height).

## Color Treatment

Default to a **dark sidebar, light content** split — it's the most common SaaS pattern and gives the clearest separation between navigation and content:

- Sidebar background: a dark slate (e.g. `#0f172a`), inactive link text in a muted light slate (e.g. `#94a3b8`), hover state a subtle lighter fill (e.g. `rgba(255,255,255,0.06)`) with brighter text.
- Active nav item: reuse the app's existing primary accent color for the background/text highlight rather than introducing a new one, so the rest of the UI doesn't need to change. If the app already highlights active states or primary actions with a color, use that exact color here.
- Content area: keep existing light background and card styles (white cards, light borders) — don't restyle content components unless asked.

If the project's own conventions file (e.g. `CLAUDE.md`) specifies a design system/palette, follow it for anything other than the sidebar's dark background.

## Navigation Items

Each nav item is icon + label, not label-only and never emoji (many projects explicitly ban emoji in UI — check for that rule first):

- Use small (18–22px) inline SVG line icons — `viewBox="0 0 24 24"`, `fill="none"`, `stroke="currentColor"`, `stroke-width="1.5"`, rounded caps/joins — so they inherit the link's text color automatically across normal/hover/active states.
- Keep icons simple and generic (grid/home for overview, box for inventory, clipboard/list for orders, currency symbol for finance, trending arrow for forecasts, bar chart for reports, etc.) — don't import an icon library or external assets; inline the SVG markup directly in the component.
- Build the nav list from a small array/computed property of `{ path, label, icon }` and `v-for` over it, rather than hand-writing each `router-link`.

## Spacing & Polish

- Consistent padding scale for sidebar items (e.g. `0.625rem 0.875rem`) and content sections (e.g. `1.5rem 2rem`).
- Rounded corners on nav items and cards (6–10px), subtle transitions (`0.2s ease`) on hover/active states.
- Keep existing card/table/badge styles in the content area as-is unless the request also asks for those to change — this skill is about the navigation shell, not every component.

## Workflow

1. Read the current root layout component and any component whose CSS assumes a top-nav layout (search for `position: sticky` with a `top` value matching the old nav height).
2. Ask clarifying questions up front (via AskUserQuestion, if available) for anything not already decided: color treatment, icon usage, and whether to persist this as a reusable skill vs. a one-off change.
3. Implement the sidebar + content-area shell, migrate nav links into the sidebar with icons, move user/account controls (profile menu, language switcher, etc.) into the new top bar.
4. Fix any sticky-positioning offsets that referenced the old top-nav height.
5. Collapsible/icon-only sidebar (build when asked, as a follow-up iteration):
   - Add a `sidebarCollapsed` ref and a small toggle button (chevron icon, rotates 180° when collapsed) in the sidebar header.
   - Bind a `sidebar-collapsed` class on the root app element; under it, shrink the sidebar to ~72px and hide logo text/nav labels (`display: none`), centering icons.
   - Add a separate `@media (max-width: 1024px)` block that forces the same icon-only width/hidden-labels styling unconditionally — this is the "smaller screens" requirement and should work independently of the manual toggle state.
   - Add `title="<label>"` on each nav link so icon-only mode still exposes the label via native tooltip.
