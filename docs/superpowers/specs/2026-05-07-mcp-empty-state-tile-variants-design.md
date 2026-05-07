# MCP Empty-State Tile Variants Design

## Summary

Change the surface-MCP experiment so both active variants use the empty-state tile
entry point. The previous first-open intro modal stays in the codebase and remains
registered, but no experiment variant auto-opens it in this iteration.

The tile becomes visually closer to the neighboring empty-state cards: a status
badge, two logo chips above one CTA line, no title and description split, and no
MCP bridge graphic.

## Goals

- Test two CTA messages while keeping the entry surface identical.
- Make the MCP tile visually consistent with the existing empty-state action
  cards.
- Keep the existing MCP onboarding modal as the destination after tile click.
- Keep the first-open intro modal available for a later experiment iteration.
- Keep the change small and reversible inside the existing experiment boundary.

## Non-goals

- Delete the first-open intro modal or its modal registration.
- Redesign the MCP onboarding setup modal.
- Change MCP backend APIs, token behavior, or setup prompts.
- Add dismissal behavior for the tile.
- Add broader reusable brand-logo infrastructure outside this experiment.

## Current repo facts

- The experiment constant lives in
  `packages/frontend/editor-ui/src/app/constants/experiments.ts`.
- The experiment store currently exposes `isTileVariant` and
  `isFirstOpenModalVariant` from the current PostHog variant.
- The empty-state tile host is
  `packages/frontend/editor-ui/src/app/components/layouts/EmptyStateLayout.vue`.
- Clicking the MCP tile opens `MCP_ONBOARDING_MODAL_KEY` with
  `data: { surface: 'tile' }`.
- The first-open intro modal component lives at
  `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/components/SurfaceMcpFirstOpenIntroModal.vue`.
- `WorkflowsView.vue` currently auto-opens the first-open intro modal when the
  first-open variant is eligible.
- The design system currently has an `anthropic` icon, but no OpenAI icon in
  `N8nIcon`.

## Product decisions

### Experiment variants

Use generic PostHog variant keys:

```text
control
variant-1
variant-2
```

Both `variant-1` and `variant-2` render the same tile entry point. They differ
only by CTA copy.

Mapping:

```text
variant-1 -> Build from your agents
variant-2 -> Connect to your AI
```

These generic keys keep experiment reporting stable if the CTA copy changes.

### First-open modal is retained but inactive

The first-open intro modal stays in the codebase and stays registered in the MCP
module descriptor. Its tests can remain because the component still exists.

For this iteration, `WorkflowsView.vue` must not auto-open the first-open intro
modal for any active variant. Any first-open-specific store state can stay if it
is still used by retained tests or useful for a later iteration, but display
logic should no longer treat any current variant as a first-open modal arm.

### Tile layout

The MCP tile should match the visual rhythm of the neighboring empty-state cards:

```text
[ Claude logo chip ] [ OpenAI logo chip ]

Build from your agents
```

or:

```text
[ Claude logo chip ] [ OpenAI logo chip ]

Connect to your AI
```

Tile rules:

- Use the existing `N8nCard` host and card sizing.
- Center the content vertically and horizontally.
- Render two logo chips above the CTA line.
- Keep a small status badge in the tile corner.
- Show `New` while MCP access is disabled.
- Show `Enabled` after MCP access is enabled.
- Remove tile subtext.
- Remove the MCP bridge graphic from the tile only.
- Keep the hero bridge graphic in the onboarding modal.
- Keep hover behavior subtle and consistent with the existing empty-state cards.

### Logo handling

Use the existing design-system `anthropic` icon for the Claude-side chip.

Because `N8nIcon` does not currently include an OpenAI icon, render a small
OpenAI mark locally within the experiment tile implementation. Keep it scoped to
the tile rather than adding a new design-system icon in this iteration.

### Enabled state

The tile remains a re-entry point after MCP is enabled while the empty-state
surface remains visible. Do not add enabled-state-specific title or description
copy in this iteration. The same variant CTA continues to show, and only the
badge changes from `New` to `Enabled`.

### Telemetry

Keep the existing tile telemetry surface value:

```text
surface: 'tile'
```

The `variant` payload will naturally distinguish `variant-1` from `variant-2`.
No new telemetry events are needed.

## Technical design

### Experiment constants and store

- Rename experiment variants in `experiments.ts` to `variant1: 'variant-1'` and
  `variant2: 'variant-2'`.
- Keep the existing public computed name `isTileVariant`, but make it true for
  both `variant-1` and `variant-2` to minimize host changes.
- Remove first-open auto-open behavior from active variant selection. The store
  may keep first-open helpers if keeping the retained intro modal tests small is
  preferable.

### Empty-state host

- Keep `showMcpTile` local eligibility logic the same, except it should show for
  both `variant-1` and `variant-2`.
- Add a small computed or helper that selects the tile CTA i18n key from the
  current variant.
- Keep click behavior unchanged: open the MCP onboarding modal with
  `surface: 'tile'`.

### First-open host

- Remove or disable the `WorkflowsView.vue` watcher path that auto-opens
  `SURFACE_MCP_FIRST_OPEN_INTRO_MODAL_KEY`.
- Keep modal registration in `module.descriptor.ts` unchanged.

### Tile graphic

Prefer the smallest implementation:

- Either convert `SurfaceMcpBridgeGraphic size="tile"` into a two-logo tile-only
  graphic while leaving `hero` behavior intact, or replace only the tile usage
  in `EmptyStateLayout.vue` with a small local logo row.
- Recommendation: replace only the tile usage. The bridge component can continue
  serving the setup modal hero without mixing two visual concepts in one file.

### Copy and i18n

Use variant-specific i18n keys under the existing empty-state MCP tile namespace:

```text
workflows.empty.mcp.tile.variant1.cta -> Build from your agents
workflows.empty.mcp.tile.variant2.cta -> Connect to your AI
workflows.empty.mcp.tile.badge.new -> New
workflows.empty.mcp.tile.badge.enabled -> Enabled
```

Remove unused tile title, description, and old enabled-state keys only if no
other code references them after the implementation. Replace the old `Active`
badge copy with `Enabled`.

## Testing strategy

- Store tests cover that `variant-1` and `variant-2` both resolve as tile-entry
  variants.
- Empty-state layout tests cover:
  - tile renders for `variant-1`,
  - tile renders for `variant-2`,
  - variant-specific CTA copy appears,
  - clicking either tile opens the existing MCP onboarding modal with
    `surface: 'tile'`,
  - the tile shows `New` when MCP access is disabled,
  - the tile shows `Enabled` when MCP access is enabled,
  - the tile has no subtext.
- Workflows view tests cover that no current variant auto-opens the first-open
  intro modal.
- Existing first-open intro modal component tests can remain to protect the
  retained component.

## Implementation notes

- Follow design-system styling conventions: use `N8nCard`, `N8nIcon`, `N8nText`,
  semantic tokens, and spacing variables.
- Do not hardcode user-facing copy in Vue templates.
- Keep the implementation scoped to the existing experiment and empty-state host.
- Do not delete the first-open intro modal files in this iteration.
