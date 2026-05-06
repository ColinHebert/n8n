# Surface MCP To New Cloud Users Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a PoC PostHog experiment that surfaces instance-level MCP to eligible new Cloud trial admins through either an empty-state tile or a first-open modal, with both variants sharing one single-screen onboarding modal.

**Architecture:** Keep experiment enrollment and persisted UI state inside a dedicated Pinia store under `src/experiments/surfaceMcpToNewCloudUsers/`, and keep product-specific MCP setup UI inside the existing `features/ai/mcpAccess/` area. Reuse the current MCP store and connection primitives, but build a dedicated onboarding modal and wire the two experiment surfaces manually so only the entry point changes across variants.

**Tech Stack:** Vue 3, Pinia, Vitest, PostHog feature flags, `useStorage`, existing MCP frontend store/API, `@n8n/design-system`, i18n JSON.

---

## File Structure

- Modify: `packages/frontend/editor-ui/src/app/constants/experiments.ts`
  Add the multi-variant experiment constant and participation tracking entry.
- Modify: `packages/frontend/@n8n/stores/src/constants.ts`
  Register the experiment Pinia store key.
- Create: `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.ts`
  Own PostHog enrollment, persisted first-open state, and experiment-specific telemetry helpers.
- Create: `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.test.ts`
  Cover variant selection, persisted dismissal/seen state, and telemetry helpers.
- Create: `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/composables/useSurfaceMcpToNewCloudUsersEligibility.ts`
  Centralize local display gating: Cloud deployment, trialing, admin-or-owner, truly empty instance.
- Create: `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/composables/useSurfaceMcpToNewCloudUsersEligibility.test.ts`
  Lock down shared gating so tile and modal use the same rules.
- Modify: `packages/frontend/editor-ui/src/features/ai/mcpAccess/mcp.constants.ts`
  Add the new onboarding modal key.
- Modify: `packages/frontend/editor-ui/src/features/ai/mcpAccess/module.descriptor.ts`
  Register the onboarding modal with the existing MCP module.
- Create: `packages/frontend/editor-ui/src/features/ai/mcpAccess/components/onboarding/MCPOnboardingClientSetup.vue`
  Render Claude Code and Codex instructions plus copyable config snippets.
- Create: `packages/frontend/editor-ui/src/features/ai/mcpAccess/modals/MCPOnboardingModal.vue`
  Single-screen modal that enables MCP inline and reveals connection details in place.
- Create: `packages/frontend/editor-ui/src/features/ai/mcpAccess/modals/MCPOnboardingModal.test.ts`
  Cover enable flow, pre-enabled behavior, and client switching.
- Modify: `packages/frontend/editor-ui/src/features/workflows/composables/useWorkflowsEmptyState.ts`
  Add computed helpers for the tile and post-dismiss reminder.
- Modify: `packages/frontend/editor-ui/src/app/components/layouts/EmptyStateLayout.vue`
  Render the experiment tile and passive reminder in the simplified empty state.
- Modify: `packages/frontend/editor-ui/src/app/components/layouts/EmptyStateLayout.test.ts`
  Cover tile rendering, reminder rendering, and tile click behavior.
- Modify: `packages/frontend/editor-ui/src/app/views/WorkflowsView.vue`
  Auto-open the onboarding modal once for the first-open variant after the eligible empty state is visible.
- Modify: `packages/frontend/editor-ui/src/app/views/WorkflowsView.test.ts`
  Cover one-time auto-open, tile variant non-open behavior, and already-seen behavior.
- Modify: `packages/frontend/@n8n/i18n/src/locales/en.json`
  Add empty-state tile/reminder copy and modal copy for Claude Code and Codex instructions.

## Shared Decisions To Keep During Execution

- Experiment name: `080_surface_mcp_to_new_cloud_users`
- Experiment constant: `SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT`
- Variants:
  - `control`
  - `variantTile`
  - `variantFirstOpenModal`
- Store key: `STORES.EXPERIMENT_SURFACE_MCP_TO_NEW_CLOUD_USERS`
- Persisted keys:
  - `N8N_SURFACE_MCP_TO_NEW_CLOUD_USERS_FIRST_OPEN_SEEN`
  - `N8N_SURFACE_MCP_TO_NEW_CLOUD_USERS_FIRST_OPEN_DISMISSED`
- Modal key: `MCP_ONBOARDING_MODAL_KEY = 'mcpOnboardingModal'`
- Surface payloads:
  - `'tile'`
  - `'first_open_modal'`
- Client payloads:
  - `'claude_code'`
  - `'codex'`

### Task 1: Scaffold the Experiment Store and Eligibility Gate

**Files:**
- Modify: `packages/frontend/editor-ui/src/app/constants/experiments.ts`
- Modify: `packages/frontend/@n8n/stores/src/constants.ts`
- Create: `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.ts`
- Create: `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.test.ts`
- Create: `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/composables/useSurfaceMcpToNewCloudUsersEligibility.ts`
- Create: `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/composables/useSurfaceMcpToNewCloudUsersEligibility.test.ts`

- [ ] **Step 1: Add the experiment constant and store key**

Write these exact additions first so the new store and tests have stable names to import:

```ts
// packages/frontend/editor-ui/src/app/constants/experiments.ts
export const SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT = createExperiment(
	'080_surface_mcp_to_new_cloud_users',
	{
		control: 'control',
		variantTile: 'variant-tile',
		variantFirstOpenModal: 'variant-first-open-modal',
	},
);

export const EXPERIMENTS_TO_TRACK = [
	// ...existing names...
	SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT.name,
];
```

```ts
// packages/frontend/@n8n/stores/src/constants.ts
export const STORES = {
	// ...existing entries...
	EXPERIMENT_SURFACE_MCP_TO_NEW_CLOUD_USERS: 'surfaceMcpToNewCloudUsers',
} as const;
```

- [ ] **Step 2: Write the failing store and eligibility tests**

Create the store test with these assertions:

```ts
// packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.test.ts
import { createPinia, setActivePinia } from 'pinia';
import { ref } from 'vue';
import { SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT } from '@/app/constants/experiments';

const mockTrack = vi.fn();
vi.mock('@/app/composables/useTelemetry', () => ({
	useTelemetry: vi.fn(() => ({ track: mockTrack })),
}));

const storageRefs = new Map<string, ReturnType<typeof ref<string | null>>>();
vi.mock('@/app/composables/useStorage', () => ({
	useStorage: vi.fn((key: string) => {
		if (!storageRefs.has(key)) storageRefs.set(key, ref<string | null>(null));
		return storageRefs.get(key)!;
	}),
}));

const mockGetVariant = vi.fn();
vi.mock('@/app/stores/posthog.store', () => ({
	usePostHog: vi.fn(() => ({ getVariant: mockGetVariant })),
}));

import {
	useSurfaceMcpToNewCloudUsersStore,
} from './surfaceMcpToNewCloudUsers.store';

describe('surfaceMcpToNewCloudUsers store', () => {
	beforeEach(() => {
		setActivePinia(createPinia());
		mockTrack.mockReset();
		mockGetVariant.mockReset();
		storageRefs.clear();
	});

	it('derives the tile variant from PostHog only', () => {
		mockGetVariant.mockReturnValue(
			SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT.variantTile,
		);

		const store = useSurfaceMcpToNewCloudUsersStore();

		expect(store.isEnabled).toBe(true);
		expect(store.isTileVariant).toBe(true);
		expect(store.isFirstOpenModalVariant).toBe(false);
	});

	it('marks the first eligible open as seen', () => {
		const store = useSurfaceMcpToNewCloudUsersStore();

		store.markFirstEligibleOpenSeen();

		expect(store.hasSeenFirstEligibleOpen).toBe(true);
	});

	it('persists first-open dismissal separately from enrollment', () => {
		const store = useSurfaceMcpToNewCloudUsersStore();

		store.dismissFirstOpenModal();

		expect(store.hasSeenFirstEligibleOpen).toBe(true);
		expect(store.hasDismissedFirstOpenModal).toBe(true);
	});

	it('tracks copy telemetry with surface and client metadata', () => {
		mockGetVariant.mockReturnValue(
			SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT.variantFirstOpenModal,
		);

		const store = useSurfaceMcpToNewCloudUsersStore();
		store.trackCopiedParameter('first_open_modal', 'codex', 'setup-config');

		expect(mockTrack).toHaveBeenCalledWith('MCP onboarding copied parameter', {
			surface: 'first_open_modal',
			client: 'codex',
			parameter: 'setup-config',
			variant: SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT.variantFirstOpenModal,
		});
	});
});
```

Create the eligibility test with these assertions:

```ts
// packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/composables/useSurfaceMcpToNewCloudUsersEligibility.test.ts
import { createPinia, setActivePinia } from 'pinia';
import { mockedStore } from '@/__tests__/utils';
import { useSettingsStore } from '@/app/stores/settings.store';
import { useCloudPlanStore } from '@/app/stores/cloudPlan.store';
import { useUsersStore } from '@/features/settings/users/users.store';

const mockIsTrulyEmpty = vi.fn();
vi.mock(
	'@/features/workflows/readyToRun/composables/useEmptyStateDetection',
	() => ({
		useEmptyStateDetection: vi.fn(() => ({ isTrulyEmpty: mockIsTrulyEmpty })),
	}),
);

import { useSurfaceMcpToNewCloudUsersEligibility } from './useSurfaceMcpToNewCloudUsersEligibility';

describe('useSurfaceMcpToNewCloudUsersEligibility', () => {
	beforeEach(() => {
		setActivePinia(createPinia());
		mockIsTrulyEmpty.mockReset().mockReturnValue(true);
	});

	it('returns true only when the local product gate is fully satisfied', () => {
		const settingsStore = mockedStore(useSettingsStore);
		const cloudPlanStore = mockedStore(useCloudPlanStore);
		const usersStore = mockedStore(useUsersStore);

		settingsStore.isCloudDeployment = true;
		cloudPlanStore.userIsTrialing = true;
		usersStore.isAdminOrOwner = true;

		const { isEligible } = useSurfaceMcpToNewCloudUsersEligibility();
		expect(isEligible.value).toBe(true);
	});

	it('returns false for non-admin users even if enrolled', () => {
		const settingsStore = mockedStore(useSettingsStore);
		const cloudPlanStore = mockedStore(useCloudPlanStore);
		const usersStore = mockedStore(useUsersStore);

		settingsStore.isCloudDeployment = true;
		cloudPlanStore.userIsTrialing = true;
		usersStore.isAdminOrOwner = false;

		const { isEligible } = useSurfaceMcpToNewCloudUsersEligibility();
		expect(isEligible.value).toBe(false);
	});
});
```

- [ ] **Step 3: Implement the store and shared eligibility composable**

Create the store with this public API:

```ts
// packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.ts
import { useTelemetry } from '@/app/composables/useTelemetry';
import { useStorage } from '@/app/composables/useStorage';
import { SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT } from '@/app/constants';
import { usePostHog } from '@/app/stores/posthog.store';
import { STORES } from '@n8n/stores';
import { defineStore } from 'pinia';
import { computed } from 'vue';

export type McpOnboardingSurface = 'tile' | 'first_open_modal';
export type McpOnboardingClient = 'claude_code' | 'codex';
export type McpOnboardingParameter = 'server-url' | 'access-token' | 'setup-config';

const FIRST_OPEN_SEEN_KEY = 'N8N_SURFACE_MCP_TO_NEW_CLOUD_USERS_FIRST_OPEN_SEEN';
const FIRST_OPEN_DISMISSED_KEY = 'N8N_SURFACE_MCP_TO_NEW_CLOUD_USERS_FIRST_OPEN_DISMISSED';

export const useSurfaceMcpToNewCloudUsersStore = defineStore(
	STORES.EXPERIMENT_SURFACE_MCP_TO_NEW_CLOUD_USERS,
	() => {
		const telemetry = useTelemetry();
		const posthogStore = usePostHog();
		const firstOpenSeen = useStorage(FIRST_OPEN_SEEN_KEY);
		const firstOpenDismissed = useStorage(FIRST_OPEN_DISMISSED_KEY);

		const currentVariant = computed(() =>
			posthogStore.getVariant(SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT.name),
		);
		const isEnabled = computed(() => Boolean(currentVariant.value));
		const isTileVariant = computed(
			() => currentVariant.value === SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT.variantTile,
		);
		const isFirstOpenModalVariant = computed(
			() =>
				currentVariant.value ===
				SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT.variantFirstOpenModal,
		);
		const hasSeenFirstEligibleOpen = computed(() => firstOpenSeen.value === 'true');
		const hasDismissedFirstOpenModal = computed(() => firstOpenDismissed.value === 'true');

		function markFirstEligibleOpenSeen() {
			firstOpenSeen.value = 'true';
		}

		function dismissFirstOpenModal() {
			markFirstEligibleOpenSeen();
			firstOpenDismissed.value = 'true';
		}

		function trackSurfaced(surface: McpOnboardingSurface) {
			telemetry.track('MCP onboarding surfaced', {
				surface,
				variant: currentVariant.value,
			});
		}

		function trackOpened(surface: McpOnboardingSurface) {
			telemetry.track('MCP onboarding opened', {
				surface,
				variant: currentVariant.value,
			});
		}

		function trackDismissed(surface: McpOnboardingSurface) {
			telemetry.track('MCP onboarding dismissed', {
				surface,
				variant: currentVariant.value,
			});
		}

		function trackEnableClicked(surface: McpOnboardingSurface) {
			telemetry.track('MCP onboarding enable clicked', {
				surface,
				variant: currentVariant.value,
			});
		}

		function trackEnabled(surface: McpOnboardingSurface) {
			telemetry.track('MCP onboarding enabled', {
				surface,
				variant: currentVariant.value,
			});
		}

		function trackClientSelected(client: McpOnboardingClient) {
			telemetry.track('MCP onboarding client selected', {
				client,
				variant: currentVariant.value,
			});
		}

		function trackCopiedParameter(
			surface: McpOnboardingSurface,
			client: McpOnboardingClient,
			parameter: McpOnboardingParameter,
		) {
			telemetry.track('MCP onboarding copied parameter', {
				surface,
				client,
				parameter,
				variant: currentVariant.value,
			});
		}

		function reset() {
			firstOpenSeen.value = null;
			firstOpenDismissed.value = null;
		}

		return {
			currentVariant,
			isEnabled,
			isTileVariant,
			isFirstOpenModalVariant,
			hasSeenFirstEligibleOpen,
			hasDismissedFirstOpenModal,
			markFirstEligibleOpenSeen,
			dismissFirstOpenModal,
			trackSurfaced,
			trackOpened,
			trackDismissed,
			trackEnableClicked,
			trackEnabled,
			trackClientSelected,
			trackCopiedParameter,
			reset,
		};
	},
);
```

Create the shared gating composable like this:

```ts
// packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/composables/useSurfaceMcpToNewCloudUsersEligibility.ts
import { computed } from 'vue';
import { useCloudPlanStore } from '@/app/stores/cloudPlan.store';
import { useSettingsStore } from '@/app/stores/settings.store';
import { useUsersStore } from '@/features/settings/users/users.store';
import { useEmptyStateDetection } from '@/features/workflows/readyToRun/composables/useEmptyStateDetection';

export function useSurfaceMcpToNewCloudUsersEligibility() {
	const settingsStore = useSettingsStore();
	const cloudPlanStore = useCloudPlanStore();
	const usersStore = useUsersStore();
	const { isTrulyEmpty } = useEmptyStateDetection();

	const isEligible = computed(() => {
		return (
			settingsStore.isCloudDeployment &&
			cloudPlanStore.userIsTrialing &&
			usersStore.isAdminOrOwner &&
			isTrulyEmpty()
		);
	});

	return {
		isEligible,
	};
}
```

- [ ] **Step 4: Run the foundational tests**

Run:

```bash
pushd packages/frontend/editor-ui
pnpm test src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.test.ts
pnpm test src/experiments/surfaceMcpToNewCloudUsers/composables/useSurfaceMcpToNewCloudUsersEligibility.test.ts
popd
```

Expected:

- both Vitest runs pass
- no snapshot updates are needed

- [ ] **Step 5: Commit the scaffold and store**

Run:

```bash
git add \
	packages/frontend/editor-ui/src/app/constants/experiments.ts \
	packages/frontend/@n8n/stores/src/constants.ts \
	packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.ts \
	packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.test.ts \
	packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/composables/useSurfaceMcpToNewCloudUsersEligibility.ts \
	packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/composables/useSurfaceMcpToNewCloudUsersEligibility.test.ts
git commit -m "feat: scaffold surface mcp experiment" -m "Add the PostHog experiment constant, shared eligibility gate, and persisted store for the MCP onboarding experiment." -m "Co-authored-by: Codex <noreply@openai.com>"
```

### Task 2: Build the Shared MCP Onboarding Modal

**Files:**
- Modify: `packages/frontend/editor-ui/src/features/ai/mcpAccess/mcp.constants.ts`
- Modify: `packages/frontend/editor-ui/src/features/ai/mcpAccess/module.descriptor.ts`
- Create: `packages/frontend/editor-ui/src/features/ai/mcpAccess/components/onboarding/MCPOnboardingClientSetup.vue`
- Create: `packages/frontend/editor-ui/src/features/ai/mcpAccess/modals/MCPOnboardingModal.vue`
- Create: `packages/frontend/editor-ui/src/features/ai/mcpAccess/modals/MCPOnboardingModal.test.ts`
- Modify: `packages/frontend/@n8n/i18n/src/locales/en.json`

- [ ] **Step 1: Register the new modal key with the MCP module**

Add the new constant and modal definition:

```ts
// packages/frontend/editor-ui/src/features/ai/mcpAccess/mcp.constants.ts
export const MCP_CONNECT_WORKFLOWS_MODAL_KEY = 'mcpConnectWorkflowsModal';
export const MCP_ONBOARDING_MODAL_KEY = 'mcpOnboardingModal';
```

```ts
// packages/frontend/editor-ui/src/features/ai/mcpAccess/module.descriptor.ts
import {
	MCP_CONNECT_WORKFLOWS_MODAL_KEY,
	MCP_ONBOARDING_MODAL_KEY,
	MCP_SETTINGS_VIEW,
} from '@/features/ai/mcpAccess/mcp.constants';

modals: [
	{
		key: MCP_CONNECT_WORKFLOWS_MODAL_KEY,
		component: async () => await import('./modals/MCPConnectWorkflowsModal.vue'),
		initialState: { open: false },
	},
	{
		key: MCP_ONBOARDING_MODAL_KEY,
		component: async () => await import('./modals/MCPOnboardingModal.vue'),
		initialState: {
			open: false,
			data: { surface: 'tile' },
		},
	},
],
```

- [ ] **Step 2: Write the failing modal test**

Create the modal test with these three cases:

```ts
// packages/frontend/editor-ui/src/features/ai/mcpAccess/modals/MCPOnboardingModal.test.ts
import { createComponentRenderer } from '@/__tests__/render';
import { createTestingPinia } from '@pinia/testing';
import userEvent from '@testing-library/user-event';
import { useMCPStore } from '@/features/ai/mcpAccess/mcp.store';
import { mockedStore } from '@/__tests__/utils';

const mockTrack = vi.fn();
vi.mock('@/app/composables/useTelemetry', () => ({
	useTelemetry: vi.fn(() => ({ track: mockTrack })),
}));

vi.mock('@n8n/stores/useRootStore', () => ({
	useRootStore: vi.fn(() => ({
		urlBaseEditor: 'https://example.n8n.cloud/',
	})),
}));

import MCPOnboardingModal from './MCPOnboardingModal.vue';

const renderComponent = createComponentRenderer(MCPOnboardingModal, {
	pinia: createTestingPinia(),
	props: {
		data: { surface: 'first_open_modal' },
	},
	global: {
		stubs: {
			Modal: { template: '<div><slot name="content" /><slot name="footer" /></div>' },
		},
	},
});

describe('MCPOnboardingModal', () => {
	it('enables MCP and loads connection details inline', async () => {
		const mcpStore = mockedStore(useMCPStore);
		mcpStore.mcpAccessEnabled = false;
		mcpStore.currentUserMCPKey = null;
		mcpStore.setMcpAccessEnabled = vi.fn().mockResolvedValue(true);
		mcpStore.getOrCreateApiKey = vi.fn().mockImplementation(async () => {
			mcpStore.currentUserMCPKey = { apiKey: 'n8n-test-token' };
		});

		const { getByRole, findByDisplayValue } = renderComponent();
		await userEvent.click(getByRole('button', { name: 'Enable MCP access' }));

		expect(mcpStore.setMcpAccessEnabled).toHaveBeenCalledWith(true);
		expect(mcpStore.getOrCreateApiKey).toHaveBeenCalled();
		expect(await findByDisplayValue('n8n-test-token')).toBeInTheDocument();
	});

	it('loads connection details immediately when MCP is already enabled', async () => {
		const mcpStore = mockedStore(useMCPStore);
		mcpStore.mcpAccessEnabled = true;
		mcpStore.currentUserMCPKey = null;
		mcpStore.getOrCreateApiKey = vi.fn().mockImplementation(async () => {
			mcpStore.currentUserMCPKey = { apiKey: 'existing-token' };
		});

		renderComponent();

		expect(mcpStore.getOrCreateApiKey).toHaveBeenCalled();
	});

	it('switches between Claude Code and Codex setup instructions', async () => {
		const mcpStore = mockedStore(useMCPStore);
		mcpStore.mcpAccessEnabled = true;
		mcpStore.currentUserMCPKey = { apiKey: 'n8n-test-token' };

		const { getByRole, getByText } = renderComponent();
		await userEvent.click(getByRole('radio', { name: 'Codex' }));

		expect(getByText('Create or update `.codex/config.toml`.')).toBeInTheDocument();
	});
});
```

- [ ] **Step 3: Implement the client setup component**

Create the shared client instructions component so the modal shell stays small:

```vue
<!-- packages/frontend/editor-ui/src/features/ai/mcpAccess/components/onboarding/MCPOnboardingClientSetup.vue -->
<script setup lang="ts">
import { computed } from 'vue';
import { N8nButton, N8nMarkdown, N8nText } from '@n8n/design-system';
import { useClipboard } from '@/app/composables/useClipboard';
import { useI18n } from '@n8n/i18n';

const props = defineProps<{
	client: 'claude_code' | 'codex';
	serverUrl: string;
	accessToken: string;
	isTokenReady: boolean;
}>();

const emit = defineEmits<{
	copy: [parameter: 'setup-config'];
}>();

const i18n = useI18n();
const { copy } = useClipboard();

const envSnippet = computed(() => `export N8N_MCP_TOKEN="${props.accessToken}"`);

const claudeConfigSnippet = computed(() =>
	JSON.stringify(
		{
			mcpServers: {
				n8n: {
					type: 'http',
					url: props.serverUrl,
					headers: {
						Authorization: 'Bearer ${N8N_MCP_TOKEN}',
					},
				},
			},
		},
		null,
		2,
	),
);

const codexConfigSnippet = computed(
	() => `[mcp_servers.n8n]\nurl = "${props.serverUrl}"\nbearer_token_env_var = "N8N_MCP_TOKEN"`,
);

const configSnippet = computed(() =>
	props.client === 'claude_code' ? claudeConfigSnippet.value : codexConfigSnippet.value,
);

const pathText = computed(() =>
	props.client === 'claude_code'
		? i18n.baseText('settings.mcp.onboarding.claudeCode.path')
		: i18n.baseText('settings.mcp.onboarding.codex.path'),
);

async function copySetup(text: string) {
	await copy(text);
	emit('copy', 'setup-config');
}
</script>

<template>
	<div :class="$style.container">
		<div>
			<N8nText tag="p">{{ i18n.baseText('settings.mcp.onboarding.step.env') }}</N8nText>
			<N8nMarkdown :content="`\`\`\`bash\n${envSnippet}\n\`\`\``" />
			<N8nButton
				variant="secondary"
				size="small"
				:disabled="!isTokenReady"
				@click="copySetup(envSnippet)"
			>
				{{ i18n.baseText('generic.copy') }}
			</N8nButton>
		</div>
		<div>
			<N8nText tag="p">{{ i18n.baseText('settings.mcp.onboarding.step.config') }}</N8nText>
			<N8nText tag="p" size="small" color="text-light">{{ pathText }}</N8nText>
			<N8nMarkdown :content="`\`\`\`${client === 'claude_code' ? 'json' : 'toml'}\n${configSnippet}\n\`\`\``" />
			<N8nButton
				variant="secondary"
				size="small"
				:disabled="!isTokenReady"
				@click="copySetup(configSnippet)"
			>
				{{ i18n.baseText('generic.copy') }}
			</N8nButton>
		</div>
	</div>
</template>
```

- [ ] **Step 4: Implement the onboarding modal and its copy**

Create the modal shell with inline enablement and token loading:

```vue
<!-- packages/frontend/editor-ui/src/features/ai/mcpAccess/modals/MCPOnboardingModal.vue -->
<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref } from 'vue';
import Modal from '@/app/components/Modal.vue';
import { useToast } from '@/app/composables/useToast';
import { useI18n } from '@n8n/i18n';
import { N8nButton, N8nNotice, N8nRadioButtons, N8nText } from '@n8n/design-system';
import { createEventBus } from '@n8n/utils/event-bus';
import { useRootStore } from '@n8n/stores/useRootStore';
import { MCP_ENDPOINT, MCP_ONBOARDING_MODAL_KEY } from '@/features/ai/mcpAccess/mcp.constants';
import { useMCPStore } from '@/features/ai/mcpAccess/mcp.store';
import ConnectionParameter from '@/features/ai/mcpAccess/components/header/connectPopover/ConnectionParameter.vue';
import MCPOnboardingClientSetup from '@/features/ai/mcpAccess/components/onboarding/MCPOnboardingClientSetup.vue';
import { useSurfaceMcpToNewCloudUsersStore } from '@/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store';

const props = defineProps<{
	data?: {
		surface?: 'tile' | 'first_open_modal';
	};
}>();

const i18n = useI18n();
const toast = useToast();
const rootStore = useRootStore();
const mcpStore = useMCPStore();
const experimentStore = useSurfaceMcpToNewCloudUsersStore();
const modalBus = createEventBus();

const activeClient = ref<'claude_code' | 'codex'>('claude_code');
const isLoadingConnectionDetails = ref(false);
const isEnabling = ref(false);
const enabledDuringThisOpen = ref(false);

const surface = computed(() => props.data?.surface ?? 'tile');
const serverUrl = computed(() => `${rootStore.urlBaseEditor}${MCP_ENDPOINT}`);
const accessToken = computed(() => {
	const value = mcpStore.currentUserMCPKey?.apiKey ?? '';
	return value.includes('******') ? '<your-access-token>' : value;
});
const canShowConnectionDetails = computed(
	() => mcpStore.mcpAccessEnabled && Boolean(mcpStore.currentUserMCPKey?.apiKey),
);

async function loadConnectionDetails() {
	try {
		isLoadingConnectionDetails.value = true;
		await mcpStore.getOrCreateApiKey();
	} catch (error) {
		toast.showError(error, i18n.baseText('settings.mcp.error.fetching.apiKey'));
	} finally {
		isLoadingConnectionDetails.value = false;
	}
}

async function enableMcpAccess() {
	try {
		isEnabling.value = true;
		experimentStore.trackEnableClicked(surface.value);
		await mcpStore.setMcpAccessEnabled(true);
		await loadConnectionDetails();
		enabledDuringThisOpen.value = true;
		experimentStore.trackEnabled(surface.value);
	} catch (error) {
		toast.showError(error, i18n.baseText('settings.mcp.toggle.error'));
	} finally {
		isEnabling.value = false;
	}
}

function handleModalClosed() {
	if (surface.value === 'first_open_modal' && !enabledDuringThisOpen.value && !mcpStore.mcpAccessEnabled) {
		experimentStore.dismissFirstOpenModal();
		experimentStore.trackDismissed(surface.value);
	}
}

function handleClientChange(value: string) {
	activeClient.value = value as 'claude_code' | 'codex';
	experimentStore.trackClientSelected(activeClient.value);
}

function handleCopy(parameter: 'server-url' | 'access-token' | 'setup-config') {
	experimentStore.trackCopiedParameter(surface.value, activeClient.value, parameter);
}

onMounted(async () => {
	modalBus.on('closed', handleModalClosed);
	if (mcpStore.mcpAccessEnabled && !mcpStore.currentUserMCPKey) {
		await loadConnectionDetails();
	}
});

onBeforeUnmount(() => {
	modalBus.off('closed', handleModalClosed);
});
</script>

<template>
	<Modal
		:name="MCP_ONBOARDING_MODAL_KEY"
		:title="i18n.baseText('settings.mcp.onboarding.title')"
		width="720px"
		:event-bus="modalBus"
		:close-on-click-modal="true"
	>
		<template #content>
			<div :class="$style.content">
				<N8nText tag="p" size="large">
					{{ i18n.baseText('settings.mcp.onboarding.description') }}
				</N8nText>

				<N8nRadioButtons
					:model-value="activeClient"
					:options="[
						{ value: 'claude_code', label: i18n.baseText('settings.mcp.onboarding.client.claudeCode') },
						{ value: 'codex', label: i18n.baseText('settings.mcp.onboarding.client.codex') },
					]"
					@update:model-value="handleClientChange"
				/>

				<div :class="$style.actions">
					<N8nButton
						variant="primary"
						:loading="isEnabling"
						:disabled="mcpStore.mcpAccessEnabled"
						@click="enableMcpAccess"
					>
						{{ i18n.baseText('settings.mcp.onboarding.enable') }}
					</N8nButton>
					<N8nNotice v-if="!canShowConnectionDetails && !isLoadingConnectionDetails">
						{{ i18n.baseText('settings.mcp.onboarding.pending') }}
					</N8nNotice>
				</div>

				<div v-if="canShowConnectionDetails" :class="$style.connection">
					<ConnectionParameter
						id="mcp-onboarding-server-url"
						:label="i18n.baseText('settings.mcp.connectPopover.serverUrl')"
						:value="serverUrl"
						@copy="handleCopy('server-url')"
					/>
					<ConnectionParameter
						id="mcp-onboarding-access-token"
						:label="i18n.baseText('settings.mcp.connectPopover.tab.accessToken')"
						:value="accessToken"
						:allow-copy="accessToken !== '<your-access-token>'"
						@copy="handleCopy('access-token')"
					/>
				</div>

				<MCPOnboardingClientSetup
					:client="activeClient"
					:server-url="serverUrl"
					:access-token="accessToken"
					:is-token-ready="canShowConnectionDetails"
					@copy="handleCopy"
				/>
			</div>
		</template>
	</Modal>
</template>
```

Add these i18n keys in the same task:

```json
"settings.mcp.onboarding.title": "Set up MCP for your instance",
"settings.mcp.onboarding.description": "Connect Claude Code or Codex so they can discover and run workflows from this n8n instance.",
"settings.mcp.onboarding.enable": "Enable MCP access",
"settings.mcp.onboarding.pending": "Enable MCP access to reveal your server URL and access token.",
"settings.mcp.onboarding.client.claudeCode": "Claude Code",
"settings.mcp.onboarding.client.codex": "Codex",
"settings.mcp.onboarding.step.env": "1. Set an environment variable",
"settings.mcp.onboarding.step.config": "2. Add this MCP config",
"settings.mcp.onboarding.claudeCode.path": "Create or update `.mcp.json` in your project root.",
"settings.mcp.onboarding.codex.path": "Create or update `.codex/config.toml`."
```

- [ ] **Step 5: Run the modal test**

Run:

```bash
pushd packages/frontend/editor-ui
pnpm test src/features/ai/mcpAccess/modals/MCPOnboardingModal.test.ts
popd
```

Expected:

- the modal test file passes
- the enable flow test asserts both `setMcpAccessEnabled(true)` and `getOrCreateApiKey()`

- [ ] **Step 6: Commit the modal**

Run:

```bash
git add \
	packages/frontend/editor-ui/src/features/ai/mcpAccess/mcp.constants.ts \
	packages/frontend/editor-ui/src/features/ai/mcpAccess/module.descriptor.ts \
	packages/frontend/editor-ui/src/features/ai/mcpAccess/components/onboarding/MCPOnboardingClientSetup.vue \
	packages/frontend/editor-ui/src/features/ai/mcpAccess/modals/MCPOnboardingModal.vue \
	packages/frontend/editor-ui/src/features/ai/mcpAccess/modals/MCPOnboardingModal.test.ts \
	packages/frontend/@n8n/i18n/src/locales/en.json
git commit -m "feat: add mcp onboarding modal" -m "Register a shared onboarding modal that enables MCP inline and renders Claude Code and Codex setup steps." -m "Co-authored-by: Codex <noreply@openai.com>"
```

### Task 3: Add the Empty-State Tile and Post-Dismiss Reminder

**Files:**
- Modify: `packages/frontend/editor-ui/src/features/workflows/composables/useWorkflowsEmptyState.ts`
- Modify: `packages/frontend/editor-ui/src/app/components/layouts/EmptyStateLayout.vue`
- Modify: `packages/frontend/editor-ui/src/app/components/layouts/EmptyStateLayout.test.ts`
- Modify: `packages/frontend/@n8n/i18n/src/locales/en.json`

- [ ] **Step 1: Extend the empty-state composable with experiment-specific visibility helpers**

Add these imports and computed values:

```ts
// packages/frontend/editor-ui/src/features/workflows/composables/useWorkflowsEmptyState.ts
import { useSurfaceMcpToNewCloudUsersEligibility } from '@/experiments/surfaceMcpToNewCloudUsers/composables/useSurfaceMcpToNewCloudUsersEligibility';
import { useSurfaceMcpToNewCloudUsersStore } from '@/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store';

const surfaceMcpToNewCloudUsersStore = useSurfaceMcpToNewCloudUsersStore();
const { isEligible: isSurfaceMcpEligible } = useSurfaceMcpToNewCloudUsersEligibility();

const showMcpTile = computed(() => {
	return (
		isSurfaceMcpEligible.value &&
		surfaceMcpToNewCloudUsersStore.isTileVariant &&
		!showAppSelection.value &&
		!showBuilderPrompt.value &&
		!showRecommendedTemplatesInline.value
	);
});

const showMcpReminder = computed(() => {
	return (
		isSurfaceMcpEligible.value &&
		surfaceMcpToNewCloudUsersStore.isFirstOpenModalVariant &&
		surfaceMcpToNewCloudUsersStore.hasDismissedFirstOpenModal
	);
});

return {
	// ...existing exports...
	showMcpTile,
	showMcpReminder,
};
```

- [ ] **Step 2: Render the tile and reminder in the simplified empty state**

Modify the empty-state view like this:

```vue
<!-- packages/frontend/editor-ui/src/app/components/layouts/EmptyStateLayout.vue -->
<script setup lang="ts">
import { computed, watch } from 'vue';
import { useUIStore } from '@/app/stores/ui.store';
import { useMCPStore } from '@/features/ai/mcpAccess/mcp.store';
import { MCP_ONBOARDING_MODAL_KEY } from '@/features/ai/mcpAccess/mcp.constants';
import { useSurfaceMcpToNewCloudUsersStore } from '@/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store';

const uiStore = useUIStore();
const mcpStore = useMCPStore();
const surfaceMcpStore = useSurfaceMcpToNewCloudUsersStore();

const {
	showAppSelection,
	showBuilderPrompt,
	showRecommendedTemplatesInline,
	showMcpTile,
	showMcpReminder,
	// ...
} = useWorkflowsEmptyState();

watch(
	showMcpTile,
	(value, previousValue) => {
		if (value && !previousValue) {
			surfaceMcpStore.trackSurfaced('tile');
		}
	},
	{ immediate: true },
);

const openMcpOnboardingFromTile = () => {
	surfaceMcpStore.trackOpened('tile');
	uiStore.openModalWithData({
		name: MCP_ONBOARDING_MODAL_KEY,
		data: { surface: 'tile' },
	});
};
</script>

<template>
	<!-- fallback section only -->
	<template v-else>
		<N8nHeading tag="h1" size="2xlarge" bold :class="$style.welcomeTitle">
			{{ emptyStateHeading }}
		</N8nHeading>
		<div :class="$style.fallbackContent">
			<N8nText tag="p" size="large" color="text-base">
				{{ emptyStateDescription }}
			</N8nText>
			<N8nText
				v-if="showMcpReminder"
				size="small"
				color="text-light"
				data-test-id="mcp-onboarding-reminder"
				:class="$style.reminder"
			>
				{{ i18n.baseText('workflows.empty.mcp.reminder') }}
			</N8nText>

			<div
				v-if="canCreateWorkflow"
				:class="[
					$style.actionCardsContainer,
					{
						[$style.singleCard]: !showReadyToRunCard && !showMcpTile,
						[$style.threeCards]: showReadyToRunCard && showMcpTile,
					},
				]"
			>
				<N8nCard
					v-if="showMcpTile"
					:class="$style.actionCard"
					hoverable
					data-test-id="mcp-onboarding-card"
					@click="openMcpOnboardingFromTile"
				>
					<div :class="$style.cardContent">
						<N8nIcon :class="$style.cardIcon" icon="mcp" color="foreground-dark" :stroke-width="1.5" />
						<N8nText size="large" class="mt-xs">
							{{
								i18n.baseText(
									mcpStore.mcpAccessEnabled
										? 'workflows.empty.mcp.tile.enabledTitle'
										: 'workflows.empty.mcp.tile.title',
								)
							}}
						</N8nText>
						<N8nText size="small" color="text-light">
							{{
								i18n.baseText(
									mcpStore.mcpAccessEnabled
										? 'workflows.empty.mcp.tile.enabledDescription'
										: 'workflows.empty.mcp.tile.description',
								)
							}}
						</N8nText>
					</div>
				</N8nCard>

				<!-- existing ready-to-run and new-workflow cards stay in place -->
			</div>
		</div>
	</template>
</template>
```

Add the supporting style and copy:

```scss
.actionCardsContainer.threeCards {
	grid-template-columns: repeat(3, 192px);
}

.reminder {
	margin-top: var(--spacing--xs);
}
```

```json
"workflows.empty.mcp.tile.title": "Set up MCP",
"workflows.empty.mcp.tile.description": "Enable Claude Code and Codex to discover and run workflows from this instance.",
"workflows.empty.mcp.tile.enabledTitle": "MCP is enabled",
"workflows.empty.mcp.tile.enabledDescription": "Reopen the setup steps for Claude Code or Codex.",
"workflows.empty.mcp.reminder": "You can enable this later in Settings > MCP."
```

- [ ] **Step 3: Extend the empty-state test**

Add these tests:

```ts
// packages/frontend/editor-ui/src/app/components/layouts/EmptyStateLayout.test.ts
import { ref } from 'vue';
import { useUIStore } from '@/app/stores/ui.store';
import { useMCPStore } from '@/features/ai/mcpAccess/mcp.store';
import { useSurfaceMcpToNewCloudUsersStore } from '@/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store';

const mcpEligibility = ref(true);
vi.mock(
	'@/experiments/surfaceMcpToNewCloudUsers/composables/useSurfaceMcpToNewCloudUsersEligibility',
	() => ({
		useSurfaceMcpToNewCloudUsersEligibility: vi.fn(() => ({
			isEligible: mcpEligibility,
		})),
	}),
);

let uiStore: ReturnType<typeof mockedStore<typeof useUIStore>>;
let mcpStore: ReturnType<typeof mockedStore<typeof useMCPStore>>;
let surfaceMcpStore: ReturnType<typeof mockedStore<typeof useSurfaceMcpToNewCloudUsersStore>>;

beforeEach(() => {
	uiStore = mockedStore(useUIStore);
	mcpStore = mockedStore(useMCPStore);
	surfaceMcpStore = mockedStore(useSurfaceMcpToNewCloudUsersStore);
	mcpStore.mcpAccessEnabled = false;
	mcpEligibility.value = true;
	surfaceMcpStore.isTileVariant = false;
	surfaceMcpStore.isFirstOpenModalVariant = false;
	surfaceMcpStore.hasDismissedFirstOpenModal = false;
});

it('renders the MCP card for the tile variant', () => {
	surfaceMcpStore.isTileVariant = true;
	const { getByTestId } = renderComponent();
	expect(getByTestId('mcp-onboarding-card')).toBeInTheDocument();
});

it('opens the onboarding modal when the MCP card is clicked', async () => {
	surfaceMcpStore.isTileVariant = true;
	const { getByTestId } = renderComponent();

	await userEvent.click(getByTestId('mcp-onboarding-card'));

	expect(uiStore.openModalWithData).toHaveBeenCalledWith({
		name: 'mcpOnboardingModal',
		data: { surface: 'tile' },
	});
});

it('renders the passive reminder after first-open dismissal', () => {
	surfaceMcpStore.isFirstOpenModalVariant = true;
	surfaceMcpStore.hasDismissedFirstOpenModal = true;
	const { getByTestId } = renderComponent();
	expect(getByTestId('mcp-onboarding-reminder')).toHaveTextContent(
		'You can enable this later in Settings > MCP.',
	);
});
```

- [ ] **Step 4: Run the empty-state test**

Run:

```bash
pushd packages/frontend/editor-ui
pnpm test src/app/components/layouts/EmptyStateLayout.test.ts
popd
```

Expected:

- the existing empty-state assertions still pass
- the new card/reminder tests pass

- [ ] **Step 5: Commit the empty-state wiring**

Run:

```bash
git add \
	packages/frontend/editor-ui/src/features/workflows/composables/useWorkflowsEmptyState.ts \
	packages/frontend/editor-ui/src/app/components/layouts/EmptyStateLayout.vue \
	packages/frontend/editor-ui/src/app/components/layouts/EmptyStateLayout.test.ts \
	packages/frontend/@n8n/i18n/src/locales/en.json
git commit -m "feat: add surface mcp empty state tile" -m "Render the experiment tile in the simplified empty state and show a reminder after dismissing the first-open modal." -m "Co-authored-by: Codex <noreply@openai.com>"
```

### Task 4: Auto-Open the Modal Once for the First-Open Variant

**Files:**
- Modify: `packages/frontend/editor-ui/src/app/views/WorkflowsView.vue`
- Modify: `packages/frontend/editor-ui/src/app/views/WorkflowsView.test.ts`

- [ ] **Step 1: Add the one-time auto-open watcher**

Wire the first-open variant from the workflows overview rather than from the modal itself:

```ts
// packages/frontend/editor-ui/src/app/views/WorkflowsView.vue
import { MCP_ONBOARDING_MODAL_KEY } from '@/features/ai/mcpAccess/mcp.constants';
import { useSurfaceMcpToNewCloudUsersEligibility } from '@/experiments/surfaceMcpToNewCloudUsers/composables/useSurfaceMcpToNewCloudUsersEligibility';
import { useSurfaceMcpToNewCloudUsersStore } from '@/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store';

const surfaceMcpStore = useSurfaceMcpToNewCloudUsersStore();
const { isEligible: isSurfaceMcpEligible } = useSurfaceMcpToNewCloudUsersEligibility();

const maybeOpenSurfaceMcpModal = () => {
	if (!shouldUseSimplifiedLayout.value) return;
	if (!isSurfaceMcpEligible.value) return;
	if (!surfaceMcpStore.isFirstOpenModalVariant) return;
	if (surfaceMcpStore.hasSeenFirstEligibleOpen) return;

	surfaceMcpStore.markFirstEligibleOpenSeen();
	surfaceMcpStore.trackSurfaced('first_open_modal');
	surfaceMcpStore.trackOpened('first_open_modal');
	uiStore.openModalWithData({
		name: MCP_ONBOARDING_MODAL_KEY,
		data: { surface: 'first_open_modal' },
	});
};

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

- [ ] **Step 2: Extend the workflows view test**

Add these tests near the existing `Simplified Layout` suite:

```ts
// packages/frontend/editor-ui/src/app/views/WorkflowsView.test.ts
import { ref } from 'vue';
import { useSurfaceMcpToNewCloudUsersStore } from '@/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store';
import { useUIStore } from '@/app/stores/ui.store';

const mcpEligibility = ref(true);
vi.mock(
	'@/experiments/surfaceMcpToNewCloudUsers/composables/useSurfaceMcpToNewCloudUsersEligibility',
	() => ({
		useSurfaceMcpToNewCloudUsersEligibility: vi.fn(() => ({
			isEligible: mcpEligibility,
		})),
	}),
);

it('auto-opens the MCP onboarding modal once for the first-open variant', async () => {
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

	expect(surfaceMcpStore.markFirstEligibleOpenSeen).toHaveBeenCalled();
	expect(uiStore.openModalWithData).toHaveBeenCalledWith({
		name: 'mcpOnboardingModal',
		data: { surface: 'first_open_modal' },
	});
});

it('does not auto-open the modal for the tile variant', async () => {
	const readyToRunStore = mockedStore(useReadyToRunStore);
	const projectsStore = mockedStore(useProjectsStore);
	const surfaceMcpStore = mockedStore(useSurfaceMcpToNewCloudUsersStore);
	const uiStore = mockedStore(useUIStore);

	vi.spyOn(readyToRunStore, 'getSimplifiedLayoutVisibility').mockReturnValue(true);
	projectsStore.currentProject = { scopes: ['workflow:create'] } as Project;
	mcpEligibility.value = true;
	surfaceMcpStore.isTileVariant = true;
	surfaceMcpStore.isFirstOpenModalVariant = false;

	renderComponent({ pinia });
	await waitAllPromises();

	expect(uiStore.openModalWithData).not.toHaveBeenCalledWith({
		name: 'mcpOnboardingModal',
		data: { surface: 'first_open_modal' },
	});
});

it('does not auto-open the modal after the first eligible open has already been seen', async () => {
	const readyToRunStore = mockedStore(useReadyToRunStore);
	const projectsStore = mockedStore(useProjectsStore);
	const surfaceMcpStore = mockedStore(useSurfaceMcpToNewCloudUsersStore);
	const uiStore = mockedStore(useUIStore);

	vi.spyOn(readyToRunStore, 'getSimplifiedLayoutVisibility').mockReturnValue(true);
	projectsStore.currentProject = { scopes: ['workflow:create'] } as Project;
	mcpEligibility.value = true;
	surfaceMcpStore.isFirstOpenModalVariant = true;
	surfaceMcpStore.hasSeenFirstEligibleOpen = true;

	renderComponent({ pinia });
	await waitAllPromises();

	expect(uiStore.openModalWithData).not.toHaveBeenCalled();
});
```

- [ ] **Step 3: Run the workflows test and full editor-ui verification**

Run:

```bash
pushd packages/frontend/editor-ui
pnpm test src/app/views/WorkflowsView.test.ts
pnpm test src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.test.ts
pnpm test src/experiments/surfaceMcpToNewCloudUsers/composables/useSurfaceMcpToNewCloudUsersEligibility.test.ts
pnpm test src/features/ai/mcpAccess/modals/MCPOnboardingModal.test.ts
pnpm test src/app/components/layouts/EmptyStateLayout.test.ts
pnpm lint
pnpm typecheck
popd
```

Expected:

- all five targeted Vitest runs pass
- `pnpm lint` exits `0`
- `pnpm typecheck` exits `0`

- [ ] **Step 4: Manual QA the two experiment variants**

Check these cases in a Cloud trial account enrolled manually through PostHog overrides:

```text
1. Control: no tile, no auto-open modal, existing empty state unchanged.
2. variantTile: MCP tile appears in the simplified empty state and opens the onboarding modal.
3. variantTile after enable: modal shows server URL + token, tile copy switches to “MCP is enabled”.
4. variantFirstOpenModal: modal auto-opens exactly once on the eligible empty-state visit.
5. variantFirstOpenModal after dismiss: no auto-open on refresh, reminder copy appears in the empty state.
6. Non-admin or non-trial or non-empty instance: neither experiment surface appears.
```

- [ ] **Step 5: Commit the first-open wiring**

Run:

```bash
git add \
	packages/frontend/editor-ui/src/app/views/WorkflowsView.vue \
	packages/frontend/editor-ui/src/app/views/WorkflowsView.test.ts
git commit -m "feat: auto-open mcp onboarding for first-open variant" -m "Show the onboarding modal once for the first-open experiment variant and verify the full editor-ui surface." -m "Co-authored-by: Codex <noreply@openai.com>"
```

## Self-Review Checklist

- The plan covers all approved product decisions:
  - 3-arm experiment
  - shared onboarding modal
  - tile variant
  - first-open modal variant
  - admin-only local gating
  - Cloud trial gating
  - truly-empty gating
  - single-screen enable flow
  - Claude Code and Codex instructions
  - reminder after first-open dismissal
- No task depends on undocumented names or placeholder files.
- Store enrollment remains PostHog-only; product gating lives in the eligibility composable and host surfaces.
- Modal registration stays inside the MCP module instead of inventing a second modal system.
