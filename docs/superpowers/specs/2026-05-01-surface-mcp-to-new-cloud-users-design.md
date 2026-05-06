# Surface MCP To New Cloud Users Design

## Summary

Create a PoC experiment that surfaces n8n MCP access to new Cloud trial users
through one of two onboarding surfaces:

- a persistent tile in the simplified empty-state layout,
- or an auto-opened modal when the instance is first opened.

Both variants use the same single-screen onboarding modal so the experiment
tests the entry surface rather than the setup experience itself.

The modal should let eligible admins enable MCP access inline, then
automatically show the real connection details and setup instructions for
`Claude Code` and `Codex` in the same screen.

Implementation should use the repository's `n8n:setup-experiment` skill to
scaffold the experiment shell before manual host-surface wiring begins.

## Goals

- Measure whether surfacing MCP earlier increases discovery and setup.
- Compare two surfaces within one experiment:
  - empty-state tile,
  - first-open modal.
- Keep the setup flow identical across both variants.
- Reuse existing MCP APIs and settings logic where possible.
- Keep the PoC easy to remove or graduate later.

## Non-goals

- Redesign the existing `Settings > MCP` page.
- Support non-admin users in the experiment flow.
- Build a multi-step wizard.
- Support more than two client instruction sets in the PoC.
- Hardcode targeting rules like the signup date in frontend code.

## Current Repo Facts

- Editor UI experiments are defined in
  `packages/frontend/editor-ui/src/app/constants/experiments.ts`.
- Experiment participation tracking is centralized through
  `EXPERIMENTS_TO_TRACK`.
- The simplified empty-state layout already exists in
  `packages/frontend/editor-ui/src/app/components/layouts/EmptyStateLayout.vue`.
- Truly-empty instance detection already exists in
  `packages/frontend/editor-ui/src/features/workflows/readyToRun/composables/useEmptyStateDetection.ts`.
- First-open product modals already have precedents in the app root and view
  layer.
- MCP instance access already exists as a product surface under
  `packages/frontend/editor-ui/src/features/ai/mcpAccess/`.
- Existing MCP store logic already supports:
  - enabling MCP access,
  - fetching or creating an access token,
  - computing connection details from the editor base URL.
- The existing MCP settings page is not a good onboarding surface for this
  experiment because it is settings-oriented rather than first-run-oriented.
- The repo contains an `n8n:setup-experiment` skill under
  `.claude/plugins/n8n/skills/setup-experiment/` with canonical rules in
  `.agents/skills/setup-experiment/reference.md`.

## Product Decisions

### Experiment shape

Use one multi-variant experiment with three arms:

- `control`
- `variantTile`
- `variantFirstOpenModal`

Each user should be enrolled in exactly one arm.

### Audience

The PoC is for new Cloud trial users with an empty instance and admin ability.

The display surface should only appear when all of the following are true:

- the user is in the experiment,
- the deployment is Cloud,
- the user is trialing,
- the user is an instance owner or admin,
- the instance is truly empty,
- the user is on the overview workflows page.

Additional first-open logic applies only to the auto-open modal variant, not to
the tile variant.

The exact date threshold for "new users after a certain date" should be managed
in PostHog targeting, not hardcoded in the frontend.

### Admin-only gating

The experiment should be admin-only for the PoC.

This avoids building a degraded "ask your admin" branch and keeps the setup
flow focused on users who can actually:

- enable MCP access,
- generate a token,
- complete the setup immediately.

### Surfaces

#### Variant: empty-state tile

Render a dedicated MCP tile inside the simplified empty-state layout.

Behavior:

- the tile stays visible while the instance remains truly empty,
- clicking the tile opens the shared MCP onboarding modal,
- after MCP is enabled, the tile can remain available as a re-entry point to
  the instructions, but its copy should reflect that MCP is enabled.

#### Variant: first-open modal

Auto-open the shared MCP onboarding modal once on the first eligible open.

Behavior:

- the modal opens automatically when the user first lands in the eligible empty
  workflows experience,
- dismissing it prevents future forced auto-open,
- after dismissal, the empty state should show a passive reminder that MCP can
  be enabled later in `Settings > MCP`.

### Shared setup experience

Note: the modal-specific interaction details in this document are superseded
by
`docs/superpowers/specs/2026-05-05-mcp-onboarding-toggle-and-agent-prompt-design.md`.
Read this older doc for experiment-surface context only, and use the newer
spec as the source of truth for current modal behavior.

Both experiment variants were originally intended to use the same
`McpOnboardingModal`.

The original draft described the onboarding modal as a single-screen UI, not a
wizard.

That draft modal outline included:

- a short explanation of why MCP is useful,
- client-specific instruction switching for:
  - `Claude Code`,
  - `Codex`,
- a primary `Enable MCP access` action,
- connection details and copy affordances in the same screen.

### Single-screen enable flow

The original draft also assumed the modal would not split enablement and setup
into separate steps.

In that version, the same screen would:

- show the setup instructions,
- allow enabling MCP,
- update in place after MCP is enabled,
- reveal real connection details without navigation.

After the user clicked enable, the draft flow was:

1. enable instance-level MCP access,
2. automatically fetch or create the MCP token,
3. render the actual server URL and token/config in place,
4. keep the user inside the same modal.

### Client instructions

The PoC only needs targeted instruction content for:

- `Claude Code`
- `Codex`

Do not build a generalized client picker for a wide matrix of MCP clients in
this iteration.

### Dismissal behavior

#### Modal variant

- Auto-open only once.
- If the user dismisses the modal, do not force-open it again.
- Show passive reminder copy in the empty state after dismissal:
  `You can enable this later in Settings > MCP`.

#### Tile variant

- The tile remains the persistent entry point while the instance stays truly
  empty.
- No extra dismissal state is needed for the tile in the PoC.

## Technical Design

### Experiment scaffold

Use `n8n:setup-experiment` to scaffold the experiment shell before implementing
the host surfaces.

The scaffold should include:

- a new experiment constant in `experiments.ts`,
- participation tracking via `EXPERIMENTS_TO_TRACK`,
- a new experiment store,
- adjacent store tests.

The scaffold should not automatically wire:

- the empty-state tile,
- modal registration,
- modal auto-open behavior,
- i18n keys,
- the modal component itself.

### Recommended experiment contract

The new experiment store should expose:

- `isEnabled`
  - PostHog enrollment only
- `currentVariant`
- `isTileVariant`
- `isFirstOpenModalVariant`
- local persisted state:
  - `hasSeenFirstEligibleOpen`
  - `hasDismissedFirstOpenModal`
- derived display helpers:
  - `shouldShowTile`
  - `shouldAutoOpenModal`

Per the experiment setup standard, local conditions such as trial status,
admin-only checks, and truly-empty checks must be treated as display logic
layered on top of enrollment rather than mixed into `isEnabled`.

## Component boundaries

### 1. Experiment store

Responsibility:

- PostHog variant resolution,
- persisted local state for the first-open modal,
- experiment-specific telemetry helpers,
- display helper computation.

Suggested location:

- `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/stores/...`

### 2. Empty-state host wiring

Responsibility:

- render the tile variant in the simplified empty-state layout,
- open the shared onboarding modal on click,
- show passive reminder copy for dismissed modal users when appropriate.

Suggested host:

- `packages/frontend/editor-ui/src/app/components/layouts/EmptyStateLayout.vue`

### 3. First-open host wiring

Responsibility:

- detect the first eligible workflows landing,
- auto-open the shared onboarding modal exactly once for the modal variant.

Suggested host:

- `packages/frontend/editor-ui/src/app/views/WorkflowsView.vue`

This is the right host because the workflows overview is already the main
landing point for truly-empty instances.

### 4. Shared onboarding modal

Responsibility:

- render the onboarding content,
- call MCP enablement,
- fetch or create the token,
- render client-specific setup instructions,
- expose copy actions for connection parameters.

Suggested location:

- `packages/frontend/editor-ui/src/features/ai/mcpAccess/components/`
- or a nearby modal file under the MCP feature boundary.

Although triggered by the experiment, this modal should reuse existing MCP
domain logic rather than duplicating API calls in the experiment store.

## Data flow

### Variant selection

1. PostHog enrolls the user into one of `control`, `variantTile`,
   `variantFirstOpenModal`.
2. The experiment store exposes `currentVariant`.
3. Host surfaces layer local display constraints on top of the variant.

### Tile flow

1. User lands in the truly-empty overview page.
2. Empty-state host decides that the tile variant should show.
3. User clicks the tile.
4. Shared onboarding modal opens.
5. User clicks `Enable MCP access`.
6. Existing MCP logic enables instance-level MCP access.
7. Existing MCP logic fetches or creates the current user's MCP key.
8. Modal updates in place with:
   - server URL,
   - access token,
   - config snippet,
   - Claude Code instructions,
   - Codex instructions.

### First-open modal flow

1. User lands in the truly-empty overview page for the first eligible time.
2. Host logic checks:
   - modal variant,
   - local eligibility,
   - not already dismissed,
   - not already auto-opened for this experiment.
3. Shared onboarding modal auto-opens.
4. If dismissed:
   - record dismissal state,
   - do not force-open again,
   - render passive reminder copy in the empty state.
5. If enabled:
   - enable MCP,
   - fetch/create key,
   - hydrate instructions in place.

## Reuse strategy

Reuse existing MCP domain logic from `features/ai/mcpAccess` for:

- `setMcpAccessEnabled(...)`,
- `getOrCreateApiKey()`,
- connection details based on editor base URL,
- any existing MCP-specific telemetry semantics that still make sense.

Do not embed the full current `SettingsMCPView` inside the experiment modal.

Reason:

- that UI is settings-page-oriented,
- it includes tabs and admin-management concerns that are not needed for the
  PoC,
- it weakens the first-run onboarding experience.

## Error handling

The modal should handle the following cases explicitly.

### MCP enablement fails

- show an inline error or toast,
- keep the modal open,
- keep the user on the same screen,
- allow retry.

### Token fetch or generation fails

- MCP may already be enabled,
- show an inline error or toast for the token/config section,
- allow retry without closing the modal,
- do not silently close or route away.

### Duplicate clicks

- disable the primary enable action while the request is in flight,
- avoid issuing multiple enable or token-generation requests.

### Modal reopen

- if MCP is already enabled when the modal opens, skip the "pre-enable" state
  and show the active connection details immediately.

## Telemetry

Keep participation telemetry centralized through `EXPERIMENTS_TO_TRACK`.

Add only experiment-specific events.

Recommended events:

- `MCP onboarding surfaced`
  - payload:
    - `variant`
    - `surface`
- `MCP onboarding opened`
  - payload:
    - `variant`
    - `surface`
- `MCP onboarding dismissed`
  - payload:
    - `variant`
    - `surface`
- `MCP onboarding enable clicked`
  - payload:
    - `variant`
    - `surface`
- `MCP onboarding enabled`
  - payload:
    - `variant`
    - `surface`
- `MCP onboarding client selected`
  - payload:
    - `client`
- `MCP onboarding copied parameter`
  - payload:
    - `item`
    - `client`

Suggested parameter values:

- `surface`
  - `tile`
  - `first_open_modal`
- `client`
  - `claude_code`
  - `codex`
- `item`
  - `server_url`
  - `access_token`
  - `config`

## i18n and copy constraints

- All user-facing text must go through `@n8n/i18n`.
- The PoC should avoid over-explaining MCP.
- The empty-state tile and modal copy should focus on:
  - build faster with your AI assistant,
  - connect Claude Code or Codex to your n8n instance,
  - enable once, then copy the setup details.

## Testing Strategy

### Store tests

Cover:

- variant resolution,
- first-open modal persistence,
- dismissal behavior,
- derived display helpers for the modal and tile surfaces.

### Host-surface tests

Cover:

- tile is rendered only in tile variant and only when locally eligible,
- modal auto-opens only in first-open variant,
- modal does not auto-open again after dismissal,
- passive reminder copy appears after dismissal for the modal variant.

### Modal tests

Cover:

- initial render before enable,
- enabling MCP calls the expected MCP logic,
- token/config hydrates automatically after enable,
- client switch between `Claude Code` and `Codex`,
- copy actions emit telemetry,
- error state on enable failure,
- error state on token fetch failure,
- reopen behavior when MCP is already enabled.

### Integration boundaries

Do not broaden the PoC test scope into:

- full MCP settings page regression coverage,
- non-admin permission branching,
- broad multi-client documentation validation.

## Open design choices intentionally excluded from the PoC

The following are intentionally out of scope for this design:

- non-admin experiment behavior,
- advanced client matrix beyond Claude Code and Codex,
- automatic workflow connection or suggested workflow enablement inside the
  onboarding modal,
- driving users into the existing settings page as the primary onboarding path.

## Implementation Notes

- The experiment scaffold must be created with `n8n:setup-experiment`.
- Manual work remains after scaffold:
  - host-surface wiring in the empty-state layout,
  - modal registration and modal component creation,
  - i18n keys,
  - MCP-specific onboarding UI,
  - tests for the host surfaces and modal.
- The experiment should be easy to delete by removing:
  - the experiment constant and store,
  - the tile host wiring,
  - the first-open auto-open logic,
  - the onboarding modal.
