# MCP Empty-State Tile Variants Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Convert the surface-MCP experiment to two tile-entry variants, `variant-1` and `variant-2`, with simplified tile visuals and variant-specific CTA copy.

**Architecture:** Keep the existing MCP onboarding modal as the tile click destination. Update the experiment store so both active variants resolve to the tile surface, remove first-open auto-open behavior from `WorkflowsView.vue`, and replace the tile-only MCP bridge with a small scoped logo-row component used by `EmptyStateLayout.vue`.

**Tech Stack:** Vue 3, TypeScript, Pinia, Vitest, `@n8n/i18n`, `@n8n/design-system`, SCSS modules.

---

## File Structure

- Modify: `packages/frontend/editor-ui/src/app/constants/experiments.ts`
  - Rename the active experiment keys to `variant1` and `variant2`, backed by PostHog values `variant-1` and `variant-2`.
- Modify: `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.ts`
  - Make `isTileVariant` true for both active variants.
  - Keep `isFirstOpenModalVariant` as a retained compatibility computed that is always false for the current experiment iteration.
- Modify: `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.test.ts`
  - Cover both active variants and telemetry payloads using the new keys.
- Modify: `packages/frontend/editor-ui/src/app/views/WorkflowsView.vue`
  - Remove the first-open modal auto-open watcher path.
- Modify: `packages/frontend/editor-ui/src/app/views/WorkflowsView.test.ts`
  - Replace the positive auto-open assertion with coverage that the retained first-open modal is not auto-opened.
- Create: `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/components/SurfaceMcpTileLogos.vue`
  - Render the two tile-only logo chips: Anthropic/Claude and OpenAI.
- Modify: `packages/frontend/editor-ui/src/app/components/layouts/EmptyStateLayout.vue`
  - Render the simplified MCP tile with badge, logo row, and variant-specific CTA.
- Modify: `packages/frontend/editor-ui/src/app/components/layouts/EmptyStateLayout.test.ts`
  - Cover variant-specific CTA copy, badge states, logo row rendering, and click behavior.
- Modify: `packages/frontend/@n8n/i18n/src/locales/en.json`
  - Add variant CTA keys and rename the enabled badge key from `active` to `enabled`.
- Existing and unchanged: `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/components/SurfaceMcpFirstOpenIntroModal.vue`
  - Keep the retained intro modal component for a later experiment iteration.

---

### Task 1: Rename Experiment Variants In Constants And Store

**Files:**
- Modify: `packages/frontend/editor-ui/src/app/constants/experiments.ts:102-109`
- Modify: `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.ts:24-35`
- Modify: `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.test.ts:58-105`

- [ ] **Step 1: Write the failing store tests**

Replace the variant tests and copied-parameter test in `surfaceMcpToNewCloudUsers.store.test.ts` with this block:

```ts
it.each([
	SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT.variant1,
	SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT.variant2,
])('derives tile entry state from PostHog variant %s', (variant) => {
	mockGetVariant.mockReturnValue(variant);

	expect(store.$id).toBe(STORES.EXPERIMENT_SURFACE_MCP_TO_NEW_CLOUD_USERS);
	expect(store.currentVariant).toBe(variant);
	expect(store.isEnabled).toBe(true);
	expect(store.isTileVariant).toBe(true);
	expect(store.isFirstOpenModalVariant).toBe(false);
});

it('treats the control variant as experiment enrollment', () => {
	mockGetVariant.mockReturnValue(SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT.control);

	expect(store.currentVariant).toBe(SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT.control);
	expect(store.isEnabled).toBe(true);
	expect(store.isTileVariant).toBe(false);
	expect(store.isFirstOpenModalVariant).toBe(false);
});

it('persists the first eligible open marker for the retained intro modal', () => {
	store.markFirstEligibleOpenSeen();

	expect(firstOpenSeenStorage.value).toBe('true');
	expect(store.hasSeenFirstEligibleOpen).toBe(true);
});

it('persists first-open modal dismissal for the retained intro modal', () => {
	store.dismissFirstOpenModal();

	expect(firstOpenDismissedStorage.value).toBe('true');
	expect(store.hasDismissedFirstOpenModal).toBe(true);
	expect(firstOpenSeenStorage.value).toBe('true');
	expect(store.hasSeenFirstEligibleOpen).toBe(true);
});

it('tracks copied parameter payload with the current variant', () => {
	mockGetVariant.mockReturnValue(SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT.variant1);
	expectTypeOf<CopiedParameter>().toEqualTypeOf<'agent-prompt'>();

	store.trackCopiedParameter('tile', 'cursor', 'agent-prompt');

	expect(mockTrack).toHaveBeenCalledWith('MCP onboarding copied parameter', {
		surface: 'tile',
		client: 'cursor',
		parameter: 'agent-prompt',
		variant: SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT.variant1,
	});
});
```

- [ ] **Step 2: Run store test to verify it fails**

Run from `packages/frontend/editor-ui`:

```bash
pnpm test src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.test.ts
```

Expected: FAIL because `variant1` and `variant2` are not defined on `SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT` yet.

- [ ] **Step 3: Rename the experiment keys**

In `packages/frontend/editor-ui/src/app/constants/experiments.ts`, replace the `SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT` block with:

```ts
export const SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT = createExperiment(
	'080_surface_mcp_to_new_cloud_users',
	{
		control: 'control',
		variant1: 'variant-1',
		variant2: 'variant-2',
	},
);
```

- [ ] **Step 4: Update the store variant computeds**

In `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.ts`, replace the `isTileVariant` and `isFirstOpenModalVariant` computeds with:

```ts
const isTileVariant = computed(() => {
	const variant = currentVariant.value;

	return (
		variant === SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT.variant1 ||
		variant === SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT.variant2
	);
});
const isFirstOpenModalVariant = computed(() => false);
```

- [ ] **Step 5: Run store test to verify it passes**

Run from `packages/frontend/editor-ui`:

```bash
pnpm test src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.test.ts
```

Expected: PASS for all tests in `surfaceMcpToNewCloudUsers.store.test.ts`.

- [ ] **Step 6: Commit**

```bash
git add packages/frontend/editor-ui/src/app/constants/experiments.ts packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.ts packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.test.ts
git commit -m "refactor: rename MCP tile experiment variants"
```

---

### Task 2: Disable First-Open Auto-Open Host Behavior

**Files:**
- Modify: `packages/frontend/editor-ui/src/app/views/WorkflowsView.vue:47-49, 157, 164, 474-495, 538-549`
- Modify: `packages/frontend/editor-ui/src/app/views/WorkflowsView.test.ts:645-703`

- [ ] **Step 1: Write the failing WorkflowsView test**

In `WorkflowsView.test.ts`, replace the test named `auto-opens the MCP intro modal once for the first-open variant` with:

```ts
it('does not auto-open the retained MCP intro modal', async () => {
	const readyToRunStore = mockedStore(useReadyToRunStore);
	const projectsStore = mockedStore(useProjectsStore);
	const surfaceMcpStore = mockedStore(useSurfaceMcpToNewCloudUsersStore);
	const uiStore = mockedStore(useUIStore);

	vi.spyOn(readyToRunStore, 'getSimplifiedLayoutVisibility').mockReturnValue(true);
	projectsStore.currentProject = { scopes: ['workflow:create'] } as Project;
	mcpEligibility.value = true;
	surfaceMcpStore.isFirstOpenModalVariant = true;
	surfaceMcpStore.hasSeenFirstEligibleOpen = false;

	renderComponent({ pinia });
	await waitAllPromises();

	expect(surfaceMcpStore.markFirstEligibleOpenSeen).not.toHaveBeenCalled();
	expect(surfaceMcpStore.trackSurfaced).not.toHaveBeenCalledWith('first_open_modal');
	expect(surfaceMcpStore.trackOpened).not.toHaveBeenCalledWith('first_open_modal');
	expect(uiStore.openModal).not.toHaveBeenCalled();
	expect(uiStore.openModalWithData).not.toHaveBeenCalled();
});
```

Remove this now-unused import from the same test file:

```ts
import { SURFACE_MCP_FIRST_OPEN_INTRO_MODAL_KEY } from '@/experiments/surfaceMcpToNewCloudUsers/constants';
```

Keep the existing tests that assert no auto-open for tile variants and after first eligible open has already been seen. They should continue to pass after the implementation.

- [ ] **Step 2: Run WorkflowsView test to verify it fails**

Run from `packages/frontend/editor-ui`:

```bash
pnpm test src/app/views/WorkflowsView.test.ts
```

Expected: FAIL because `WorkflowsView.vue` still calls `uiStore.openModal(SURFACE_MCP_FIRST_OPEN_INTRO_MODAL_KEY)` when `isFirstOpenModalVariant` is true.

- [ ] **Step 3: Remove first-open auto-open code from WorkflowsView**

In `packages/frontend/editor-ui/src/app/views/WorkflowsView.vue`, delete these imports:

```ts
import { SURFACE_MCP_FIRST_OPEN_INTRO_MODAL_KEY } from '@/experiments/surfaceMcpToNewCloudUsers/constants';
import { useSurfaceMcpToNewCloudUsersEligibility } from '@/experiments/surfaceMcpToNewCloudUsers/composables/useSurfaceMcpToNewCloudUsersEligibility';
import { useSurfaceMcpToNewCloudUsersStore } from '@/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store';
```

Delete these setup constants:

```ts
const surfaceMcpStore = useSurfaceMcpToNewCloudUsersStore();
const { isEligible: isSurfaceMcpEligible } = useSurfaceMcpToNewCloudUsersEligibility();
```

Delete the entire `maybeOpenSurfaceMcpModal` function:

```ts
const maybeOpenSurfaceMcpModal = () => {
	if (!shouldUseSimplifiedLayout.value) {
		return;
	}

	if (!isSurfaceMcpEligible.value) {
		return;
	}

	if (!surfaceMcpStore.isFirstOpenModalVariant) {
		return;
	}

	if (surfaceMcpStore.hasSeenFirstEligibleOpen) {
		return;
	}

	surfaceMcpStore.markFirstEligibleOpenSeen();
	surfaceMcpStore.trackSurfaced('first_open_modal');
	surfaceMcpStore.trackOpened('first_open_modal');
	uiStore.openModal(SURFACE_MCP_FIRST_OPEN_INTRO_MODAL_KEY);
};
```

Delete the watcher that calls `maybeOpenSurfaceMcpModal()`:

```ts
watch(
	[
		shouldUseSimplifiedLayout,
		isSurfaceMcpEligible,
		() => surfaceMcpStore.isFirstOpenModalVariant,
		() => surfaceMcpStore.hasSeenFirstEligibleOpen,
	],
	() => {
		maybeOpenSurfaceMcpModal();
	},
	{ immediate: true },
);
```

- [ ] **Step 4: Run WorkflowsView test to verify it passes**

Run from `packages/frontend/editor-ui`:

```bash
pnpm test src/app/views/WorkflowsView.test.ts
```

Expected: PASS for all tests in `WorkflowsView.test.ts`.

- [ ] **Step 5: Commit**

```bash
git add packages/frontend/editor-ui/src/app/views/WorkflowsView.vue packages/frontend/editor-ui/src/app/views/WorkflowsView.test.ts
git commit -m "refactor: keep MCP intro modal inactive"
```

---

### Task 3: Build The Simplified Variant Tile

**Files:**
- Create: `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/components/SurfaceMcpTileLogos.vue`
- Modify: `packages/frontend/editor-ui/src/app/components/layouts/EmptyStateLayout.vue:1-22, 106-112, 209-260, 430-503`
- Modify: `packages/frontend/editor-ui/src/app/components/layouts/EmptyStateLayout.test.ts:1-16, 195-224`
- Modify: `packages/frontend/@n8n/i18n/src/locales/en.json:2841-2846`

- [ ] **Step 1: Write the failing EmptyStateLayout tests**

Add this import to `packages/frontend/editor-ui/src/app/components/layouts/EmptyStateLayout.test.ts`:

```ts
import { SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT } from '@/app/constants/experiments';
```

Replace the existing MCP tile tests in the `when recommended templates feature is disabled` block with:

```ts
it('renders the variant 1 MCP tile CTA with a New badge', () => {
	surfaceMcpStore.isTileVariant = true;
	surfaceMcpStore.currentVariant = SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT.variant1;
	mcpStore.mcpAccessEnabled = false;

	const { getByTestId, queryByText } = renderComponent();
	const card = getByTestId('mcp-onboarding-card');

	expect(card).toHaveTextContent('Build from your agents');
	expect(getByTestId('mcp-onboarding-badge')).toHaveTextContent('New');
	expect(getByTestId('mcp-tile-logo-row')).toBeInTheDocument();
	expect(queryByText(/Connect MCP clients like Claude Code and Cursor/)).not.toBeInTheDocument();
});

it('renders the variant 2 MCP tile CTA with a New badge', () => {
	surfaceMcpStore.isTileVariant = true;
	surfaceMcpStore.currentVariant = SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT.variant2;
	mcpStore.mcpAccessEnabled = false;

	const { getByTestId, queryByText } = renderComponent();
	const card = getByTestId('mcp-onboarding-card');

	expect(card).toHaveTextContent('Connect to your AI');
	expect(getByTestId('mcp-onboarding-badge')).toHaveTextContent('New');
	expect(getByTestId('mcp-tile-logo-row')).toBeInTheDocument();
	expect(queryByText(/Connect MCP clients like Claude Code and Cursor/)).not.toBeInTheDocument();
});

it('renders the Enabled badge when MCP access is enabled', () => {
	surfaceMcpStore.isTileVariant = true;
	surfaceMcpStore.currentVariant = SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT.variant1;
	mcpStore.mcpAccessEnabled = true;

	const { getByTestId } = renderComponent();

	expect(getByTestId('mcp-onboarding-badge')).toHaveTextContent('Enabled');
});

it('opens the onboarding modal when the MCP card is clicked', async () => {
	surfaceMcpStore.isTileVariant = true;
	surfaceMcpStore.currentVariant = SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT.variant1;
	const { getByTestId } = renderComponent();

	await userEvent.click(getByTestId('mcp-onboarding-card'));

	expect(uiStore.openModalWithData).toHaveBeenCalledWith({
		name: 'mcpOnboardingModal',
		data: { surface: 'tile' },
	});
});
```

- [ ] **Step 2: Run EmptyStateLayout test to verify it fails**

Run from `packages/frontend/editor-ui`:

```bash
pnpm test src/app/components/layouts/EmptyStateLayout.test.ts
```

Expected: FAIL because the new CTA keys, `mcp-onboarding-badge` test id, and `mcp-tile-logo-row` component are not implemented yet.

- [ ] **Step 3: Add the tile-only logo row component**

Create `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/components/SurfaceMcpTileLogos.vue` with:

```vue
<script setup lang="ts">
import { N8nIcon } from '@n8n/design-system';
</script>

<template>
	<div :class="$style.logoRow" aria-hidden="true" data-test-id="mcp-tile-logo-row">
		<span :class="[$style.logoChip, $style.logoChipClaude]">
			<N8nIcon icon="anthropic" :class="$style.logoIcon" />
		</span>
		<span :class="[$style.logoChip, $style.logoChipOpenAi]">
			<svg
				viewBox="0 0 32 32"
				xmlns="http://www.w3.org/2000/svg"
				fill="none"
				aria-hidden="true"
				:class="$style.openAiIcon"
			>
				<path
					d="M16 5.5c2.5 0 4.5 2 4.5 4.5v1.2l1-.6c2.2-1.3 5-.5 6.2 1.7 1.3 2.2.5 5-1.7 6.2l-1 .6 1 .6c2.2 1.3 3 4 1.7 6.2-1.3 2.2-4 3-6.2 1.7l-1-.6v1.2c0 2.5-2 4.5-4.5 4.5s-4.5-2-4.5-4.5v-1.2l-1 .6c-2.2 1.3-5 .5-6.2-1.7-1.3-2.2-.5-5 1.7-6.2l1-.6-1-.6c-2.2-1.3-3-4-1.7-6.2 1.3-2.2 4-3 6.2-1.7l1 .6V10c0-2.5 2-4.5 4.5-4.5Z"
					stroke="currentColor"
					stroke-width="2"
					stroke-linecap="round"
					stroke-linejoin="round"
				/>
				<path
					d="M11.75 13.2 20.25 18M20.25 13.2 11.75 18M16 10.4v11.2"
					stroke="currentColor"
					stroke-width="2"
					stroke-linecap="round"
				/>
			</svg>
		</span>
	</div>
</template>

<style lang="scss" module>
.logoRow {
	display: flex;
	align-items: center;
	justify-content: center;
	gap: var(--spacing--2xs);
	margin-bottom: var(--spacing--xs);
}

.logoChip {
	display: inline-flex;
	align-items: center;
	justify-content: center;
	width: 40px;
	height: 40px;
	border-radius: var(--radius--lg);
	background: var(--background--surface);
	border: 1px solid var(--border-color--subtle);
	box-shadow: 0 1px 2px rgb(0 0 0 / 4%);
}

.logoChipClaude {
	color: var(--color--orange-500);
}

.logoChipOpenAi {
	color: var(--color--green-600);
}

.logoIcon {
	font-size: 22px;
}

.openAiIcon {
	width: 22px;
	height: 22px;
}
</style>
```

- [ ] **Step 4: Update i18n keys**

In `packages/frontend/@n8n/i18n/src/locales/en.json`, replace the old MCP tile keys with:

```json
"workflows.empty.mcp.tile.variant1.cta": "Build from your agents",
"workflows.empty.mcp.tile.variant2.cta": "Connect to your AI",
"workflows.empty.mcp.tile.badge.new": "New",
"workflows.empty.mcp.tile.badge.enabled": "Enabled",
```

Remove these old keys if no code references them after Step 5:

```json
"workflows.empty.mcp.tile.title": "Set up MCP",
"workflows.empty.mcp.tile.description": "Connect MCP clients like Claude Code and Cursor to build, run, and iterate on workflows in your instance",
"workflows.empty.mcp.tile.enabledTitle": "MCP is enabled",
"workflows.empty.mcp.tile.enabledDescription": "Reopen the setup steps for Claude Code, Cursor, or Codex.",
"workflows.empty.mcp.tile.badge.active": "Active",
```

- [ ] **Step 5: Update EmptyStateLayout script and template**

In `packages/frontend/editor-ui/src/app/components/layouts/EmptyStateLayout.vue`, update imports to include the experiment constant, `BaseTextKey`, and the new logo component:

```ts
import type { BaseTextKey } from '@n8n/i18n';
import { SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT } from '@/app/constants/experiments';
import SurfaceMcpTileLogos from '@/experiments/surfaceMcpToNewCloudUsers/components/SurfaceMcpTileLogos.vue';
```

Remove this import from the same file:

```ts
import SurfaceMcpBridgeGraphic from '@/experiments/surfaceMcpToNewCloudUsers/components/SurfaceMcpBridgeGraphic.vue';
```

Add this computed below `containerStyle`:

```ts
const mcpTileCtaKey = computed<BaseTextKey>(() =>
	surfaceMcpStore.currentVariant === SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT.variant2
		? 'workflows.empty.mcp.tile.variant2.cta'
		: 'workflows.empty.mcp.tile.variant1.cta',
);
```

Replace the MCP `N8nCard` block with:

```vue
<N8nCard
	v-if="showMcpTile"
	:class="[$style.actionCard, $style.mcpCard]"
	hoverable
	data-test-id="mcp-onboarding-card"
	@click="openMcpOnboardingFromTile"
>
	<span
		:class="[$style.mcpBadge, { [$style.mcpBadgeEnabled]: mcpStore.mcpAccessEnabled }]"
		data-test-id="mcp-onboarding-badge"
	>
		<N8nIcon
			v-if="mcpStore.mcpAccessEnabled"
			icon="check"
			size="xsmall"
			:stroke-width="2.5"
		/>
		{{
			i18n.baseText(
				mcpStore.mcpAccessEnabled
					? 'workflows.empty.mcp.tile.badge.enabled'
					: 'workflows.empty.mcp.tile.badge.new',
			)
		}}
	</span>
	<div :class="$style.mcpCardContent">
		<SurfaceMcpTileLogos />
		<N8nText size="large" :class="$style.mcpCta">
			{{ i18n.baseText(mcpTileCtaKey) }}
		</N8nText>
	</div>
</N8nCard>
```

- [ ] **Step 6: Update EmptyStateLayout styles**

In the MCP tile style section of `EmptyStateLayout.vue`, replace `.mcpCard`, `.mcpCardEnabled`, `.mcpCardContent`, `.mcpGraphic`, `.mcpTitle`, `.mcpBadge`, and `.mcpBadgeActive` with:

```scss
.mcpCard {
	position: relative;
	overflow: hidden;
}

.mcpCardContent {
	display: flex;
	flex-direction: column;
	align-items: center;
	justify-content: center;
	padding: var(--spacing--md);
	width: 100%;
}

.mcpCta {
	letter-spacing: var(--letter-spacing--tight);
}

.mcpBadge {
	position: absolute;
	top: var(--spacing--3xs);
	right: var(--spacing--3xs);
	display: inline-flex;
	align-items: center;
	gap: var(--spacing--5xs);
	padding: 2px var(--spacing--3xs);
	border-radius: var(--radius--full);
	background: var(--color--orange-100);
	color: var(--color--orange-800);
	font-size: var(--font-size--3xs);
	font-weight: var(--font-weight--bold);
	letter-spacing: var(--letter-spacing--wider);
	text-transform: uppercase;
	line-height: 1;
	z-index: 1;
}

.mcpBadgeEnabled {
	background: var(--color--green-100);
	color: var(--color--green-800);
}
```

- [ ] **Step 7: Run EmptyStateLayout test to verify it passes**

Run from `packages/frontend/editor-ui`:

```bash
pnpm test src/app/components/layouts/EmptyStateLayout.test.ts
```

Expected: PASS for all tests in `EmptyStateLayout.test.ts`.

- [ ] **Step 8: Commit**

```bash
git add packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/components/SurfaceMcpTileLogos.vue packages/frontend/editor-ui/src/app/components/layouts/EmptyStateLayout.vue packages/frontend/editor-ui/src/app/components/layouts/EmptyStateLayout.test.ts packages/frontend/@n8n/i18n/src/locales/en.json
git commit -m "feat: simplify MCP experiment tile variants"
```

---

### Task 4: Run Focused Verification

**Files:**
- Verify: `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.test.ts`
- Verify: `packages/frontend/editor-ui/src/app/components/layouts/EmptyStateLayout.test.ts`
- Verify: `packages/frontend/editor-ui/src/app/views/WorkflowsView.test.ts`
- Verify: `packages/frontend/editor-ui`

- [ ] **Step 1: Run store tests**

Run from `packages/frontend/editor-ui`:

```bash
pnpm test src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.test.ts
```

Expected: PASS.

- [ ] **Step 2: Run empty-state layout tests**

Run from `packages/frontend/editor-ui`:

```bash
pnpm test src/app/components/layouts/EmptyStateLayout.test.ts
```

Expected: PASS.

- [ ] **Step 3: Run WorkflowsView tests**

Run from `packages/frontend/editor-ui`:

```bash
pnpm test src/app/views/WorkflowsView.test.ts
```

Expected: PASS.

- [ ] **Step 4: Run editor-ui typecheck**

Run from `packages/frontend/editor-ui`:

```bash
pnpm typecheck
```

Expected: PASS.

- [ ] **Step 5: Run editor-ui lint**

Run from `packages/frontend/editor-ui`:

```bash
pnpm lint
```

Expected: PASS.

- [ ] **Step 6: Check final git status**

Run from the repository root:

```bash
git status --short
```

Expected: only intentional docs changes remain uncommitted if plan and spec files were not included in the implementation commits.
