# MCP Onboarding Toggle and Agent Prompt Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Refactor `MCPOnboardingModal.vue` so that enabling MCP access uses a bidirectional toggle and the manual two-step setup + connection-details panel are replaced by a single agent-prompt panel that the user pastes into Claude Code or Codex.

**Architecture:** Reuse the existing `McpAccessToggle` component for the toggle; rewrite `MCPOnboardingClientSetup.vue` to render one prompt panel instead of two manual-step cards; leave `mcp.store.ts` and `McpAccessToggle.vue` untouched; narrow the experiment store's telemetry parameter union.

**Tech Stack:** Vue 3, Pinia, `@n8n/design-system` (`McpAccessToggle`, `N8nButton`, `N8nMarkdown`, `N8nNotice`, `N8nText`, `N8nRadioButtons`), `@n8n/i18n`, vitest, `@testing-library/vue`.

**Spec:** `docs/superpowers/specs/2026-05-05-mcp-onboarding-toggle-and-agent-prompt-design.md`

---

## File map

**Create:**

- `packages/frontend/editor-ui/src/features/ai/mcpAccess/components/onboarding/MCPOnboardingClientSetup.test.ts` — vitest tests for the new prompt-rendering component.

**Modify:**

- `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.ts` — narrow `SurfaceMcpOnboardingParameter` to `'agent-prompt'`.
- `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.test.ts` — update copy-parameter test.
- `packages/frontend/@n8n/i18n/src/locales/en.json` — drop 4 keys, modify 2, add 5.
- `packages/frontend/editor-ui/src/features/ai/mcpAccess/components/onboarding/MCPOnboardingClientSetup.vue` — rewrite for prompt rendering.
- `packages/frontend/editor-ui/src/features/ai/mcpAccess/modals/MCPOnboardingModal.vue` — toggle, drop connection panel, own token fetch.
- `packages/frontend/editor-ui/src/features/ai/mcpAccess/modals/MCPOnboardingModal.test.ts` — update modal tests.
- `docs/superpowers/specs/2026-05-01-surface-mcp-to-new-cloud-users-design.md` — add a short note pointing to the revised onboarding-modal spec.

---

## Task 1: Narrow the experiment-store telemetry parameter union

**Files:**

- Modify: `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.ts:14`
- Test: `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.test.ts:89-100`

- [ ] **Step 1: Update the failing test**

Replace the existing copy-parameter test (lines 89–100) so it asserts the new `'agent-prompt'` value:

```ts
it('tracks copied parameter payload with the current variant', () => {
    mockGetVariant.mockReturnValue(SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT.variantFirstOpenModal);

    store.trackCopiedParameter('first_open_modal', 'codex', 'agent-prompt');

    expect(mockTrack).toHaveBeenCalledWith('MCP onboarding copied parameter', {
        surface: 'first_open_modal',
        client: 'codex',
        parameter: 'agent-prompt',
        variant: SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT.variantFirstOpenModal,
    });
});
```

- [ ] **Step 2: Run typecheck to verify it fails**

Run from inside `packages/frontend/editor-ui`:

```bash
pnpm typecheck
```

Expected: FAIL with a TypeScript error in `surfaceMcpToNewCloudUsers.store.test.ts` because `'agent-prompt'` is not assignable to `'server-url' | 'access-token' | 'setup-config'`.

- [ ] **Step 3: Narrow the type alias in the store**

In `surfaceMcpToNewCloudUsers.store.ts:14`, replace:

```ts
type SurfaceMcpOnboardingParameter = 'server-url' | 'access-token' | 'setup-config';
```

with:

```ts
type SurfaceMcpOnboardingParameter = 'agent-prompt';
```

- [ ] **Step 4: Run typecheck to verify it passes**

```bash
pnpm typecheck
```

Expected: FAIL, but only on the existing `MCPOnboardingModal.vue` call sites that still pass `'server-url'`, `'access-token'`, and `'setup-config'`. This is expected until Task 4 rewires the modal to `'agent-prompt'`.

- [ ] **Step 5: Run the store test to verify behavior still passes**

```bash
pnpm test src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.test.ts
```

Expected: PASS for all five tests in the file.

- [ ] **Step 6: Commit**

```bash
git add packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.ts \
        packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.test.ts
git commit -m "refactor(experiment): narrow MCP onboarding parameter to agent-prompt"
```

---

## Task 2: Update i18n keys for the prompt-driven onboarding modal

**Files:**

- Modify: `packages/frontend/@n8n/i18n/src/locales/en.json:2825-2832`

This task has no standalone test; the i18n changes are exercised by the component and modal tests in Tasks 3 and 4. Run typecheck after the change.

- [ ] **Step 1: Drop obsolete keys**

In `en.json`, remove these four lines (currently 2825 and 2829–2832):

```json
"settings.mcp.onboarding.enable": "Enable MCP access",
"settings.mcp.onboarding.step.env": "1. Set an environment variable",
"settings.mcp.onboarding.step.config": "2. Add this MCP config",
"settings.mcp.onboarding.claudeCode.path": "Create or update `.mcp.json` in your project root.",
"settings.mcp.onboarding.codex.path": "Create or update `.codex/config.toml`.",
```

- [ ] **Step 2: Modify the `pending` key**

Replace the existing `settings.mcp.onboarding.pending` value:

```json
"settings.mcp.onboarding.pending": "Toggle MCP access on to reveal your setup prompt.",
```

- [ ] **Step 3: Add the six new keys**

Add these lines in the `settings.mcp.onboarding.*` block (place them right after `settings.mcp.onboarding.client.codex`):

```json
"settings.mcp.onboarding.prompt.title": "Setup prompt",
"settings.mcp.onboarding.redacted.notice": "This access token is hidden for security. Rotate the access token in Settings > MCP to copy a new setup prompt.",
"settings.mcp.onboarding.prompt.description": "Paste this into {client} to set up the n8n MCP server. When the agent finishes, restart {client}.",
"settings.mcp.onboarding.prompt.claudeCode": "Set up the n8n MCP server for Claude Code globally (available in all projects).\n\n1. Add the environment variable to your shell config (e.g. ~/.zshrc or ~/.bashrc), then reload your shell:\n\n   export N8N_MCP_TOKEN=\"{token}\"\n\n2. Register the MCP server at user scope by running:\n\n   claude mcp add --scope user --transport http n8n \\\n     \"{serverUrl}\" \\\n     --header 'Authorization: Bearer ${N8N_MCP_TOKEN}'\n\n3. Preserve `${N8N_MCP_TOKEN}` literally in the stored header value. Do not expand it before running the command.\n\n4. When you finish, tell me to restart Claude Code. Do not try to verify the server in this session.",
"settings.mcp.onboarding.prompt.codex": "Set up the n8n MCP server for Codex globally.\n\n1. Add the environment variable to your shell config (e.g. ~/.zshrc or ~/.bashrc), then reload your shell:\n\n   export N8N_MCP_TOKEN=\"{token}\"\n\n2. Add the following to ~/.codex/config.toml (create the file and directory if they do not exist; if the file already exists, append the section without removing existing config):\n\n   [mcp_servers.n8n]\n   url = \"{serverUrl}\"\n   bearer_token_env_var = \"N8N_MCP_TOKEN\"\n\n3. When you finish, tell me to restart Codex. Do not try to verify the server in this session.",
```

Use vue-i18n literal-message syntax for the Claude prompt's shell placeholder when you apply this block in `en.json`: encode `${N8N_MCP_TOKEN}` as `${'{N8N_MCP_TOKEN}'}` so the rendered output still contains the literal `${N8N_MCP_TOKEN}`.

- [ ] **Step 4: Verify the JSON parses**

```bash
node -e "JSON.parse(require('fs').readFileSync('packages/frontend/@n8n/i18n/src/locales/en.json','utf8'))"
```

Expected: no output (silent success). If there is a SyntaxError, fix the offending line before proceeding.

- [ ] **Step 5: Verify typecheck still works**

Run from `packages/frontend/@n8n/i18n`:

```bash
pnpm typecheck
```

Expected: PASS. (The locale file is JSON; this checks the package builds.)

- [ ] **Step 6: Commit**

```bash
git add packages/frontend/@n8n/i18n/src/locales/en.json
git commit -m "feat(i18n): replace MCP onboarding step keys with agent-prompt keys"
```

---

## Task 3: Rewrite `MCPOnboardingClientSetup.vue` to render the agent prompt

**Files:**

- Create: `packages/frontend/editor-ui/src/features/ai/mcpAccess/components/onboarding/MCPOnboardingClientSetup.test.ts`
- Modify: `packages/frontend/editor-ui/src/features/ai/mcpAccess/components/onboarding/MCPOnboardingClientSetup.vue`

The new component receives `client`, `serverUrl`, `accessToken`, `isTokenReady` as props (same as today), renders one panel containing the i18n-templated prompt, and emits `copy: 'agent-prompt'` from a single copy button.

- [ ] **Step 1: Write the failing component tests**

Create the test file with this content:

```ts
import { createComponentRenderer } from '@/__tests__/render';
import userEvent from '@testing-library/user-event';
import MCPOnboardingClientSetup from './MCPOnboardingClientSetup.vue';

const mockClipboardCopy = vi.fn();

vi.mock('@/app/composables/useClipboard', () => ({
    useClipboard: () => ({
        copy: mockClipboardCopy,
        copied: { value: false },
        isSupported: { value: true },
    }),
}));

const renderComponent = createComponentRenderer(MCPOnboardingClientSetup, {
    props: {
        client: 'claude_code',
        serverUrl: 'https://example.n8n.cloud/mcp-server/http',
        accessToken: 'n8n-real-token',
        isTokenReady: true,
    },
});

describe('MCPOnboardingClientSetup', () => {
    beforeEach(() => {
        mockClipboardCopy.mockReset();
    });

    it('renders the Claude Code prompt with the resolved token, server URL, and literal placeholder', () => {
        const { container } = renderComponent();
        const text = container.textContent ?? '';

        expect(text).toContain('claude mcp add --scope user --transport http n8n');
        expect(text).toContain('https://example.n8n.cloud/mcp-server/http');
        expect(text).toContain('export N8N_MCP_TOKEN="n8n-real-token"');
        expect(text).toContain("'Authorization: Bearer ${N8N_MCP_TOKEN}'");
        expect(text).not.toContain('claude mcp list');
    });

    it('renders the Codex prompt with the TOML section and home-dir path', () => {
        const { container } = renderComponent({ props: { client: 'codex' } });
        const text = container.textContent ?? '';

        expect(text).toContain('[mcp_servers.n8n]');
        expect(text).toContain('~/.codex/config.toml');
        expect(text).toContain('bearer_token_env_var = "N8N_MCP_TOKEN"');
        expect(text).toContain('https://example.n8n.cloud/mcp-server/http');
    });

    it('renders the placeholder and disables copy when the token is not ready', () => {
        const { container, getByTestId } = renderComponent({
            props: { isTokenReady: false, accessToken: '' },
        });
        const text = container.textContent ?? '';

        expect(text).toContain('<your-access-token>');
        expect(getByTestId('mcp-onboarding-copy-prompt-button')).toBeDisabled();
    });

    it('copies the prompt body and emits copy event on click', async () => {
        const user = userEvent.setup();
        const { getByTestId, emitted } = renderComponent();

        await user.click(getByTestId('mcp-onboarding-copy-prompt-button'));

        expect(mockClipboardCopy).toHaveBeenCalledTimes(1);
        const copiedText = mockClipboardCopy.mock.calls[0][0] as string;
        expect(copiedText).toContain('claude mcp add --scope user');
        expect(copiedText).toContain('n8n-real-token');

        expect(emitted('copy')).toEqual([['agent-prompt']]);
    });
});
```

- [ ] **Step 2: Run the tests to verify they fail**

```bash
pnpm test src/features/ai/mcpAccess/components/onboarding/MCPOnboardingClientSetup.test.ts
```

Expected: FAIL — the component still renders the old two-step layout, no `mcp-onboarding-copy-prompt-button` test id, no `claude mcp add` text.

- [ ] **Step 3: Rewrite the component**

Replace the entire contents of `MCPOnboardingClientSetup.vue` with:

```vue
<script setup lang="ts">
import { useClipboard } from '@/app/composables/useClipboard';
import { N8nButton, N8nMarkdown, N8nText } from '@n8n/design-system';
import { useI18n } from '@n8n/i18n';
import { computed } from 'vue';

type MCPOnboardingClient = 'claude_code' | 'codex';

const props = defineProps<{
    client: MCPOnboardingClient;
    serverUrl: string;
    accessToken: string;
    isTokenReady: boolean;
}>();

const emit = defineEmits<{
    copy: [parameter: 'agent-prompt'];
}>();

const i18n = useI18n();
const { copy } = useClipboard();

const tokenValue = computed(() =>
    props.isTokenReady ? props.accessToken : '<your-access-token>',
);

const promptKey = computed(() =>
    props.client === 'claude_code'
        ? 'settings.mcp.onboarding.prompt.claudeCode'
        : 'settings.mcp.onboarding.prompt.codex',
);

const promptBody = computed(() =>
    i18n.baseText(promptKey.value, {
        interpolate: {
            token: tokenValue.value,
            serverUrl: props.serverUrl,
        },
    }),
);

const promptMarkdown = computed(() => `\`\`\`\n${promptBody.value}\n\`\`\``);

const clientLabel = computed(() =>
    i18n.baseText(
        props.client === 'claude_code'
            ? 'settings.mcp.onboarding.client.claudeCode'
            : 'settings.mcp.onboarding.client.codex',
    ),
);

const description = computed(() =>
    i18n.baseText('settings.mcp.onboarding.prompt.description', {
        interpolate: { client: clientLabel.value },
    }),
);

async function handleCopy() {
    await copy(promptBody.value);
    emit('copy', 'agent-prompt');
}
</script>

<template>
    <div :class="$style.container" data-test-id="mcp-onboarding-client-setup">
        <N8nText tag="p" size="small" color="text-base" :bold="true">
            {{ i18n.baseText('settings.mcp.onboarding.prompt.title') }}
        </N8nText>
        <N8nText tag="p" size="small" color="text-light">
            {{ description }}
        </N8nText>
        <N8nMarkdown :content="promptMarkdown" />
        <N8nButton
            variant="subtle"
            size="small"
            :disabled="!isTokenReady"
            data-test-id="mcp-onboarding-copy-prompt-button"
            @click="handleCopy"
        >
            {{ i18n.baseText('generic.copy') }}
        </N8nButton>
    </div>
</template>

<style lang="scss" module>
.container {
    display: flex;
    flex-direction: column;
    gap: var(--spacing--3xs);
    padding: var(--spacing--xs);
    border: var(--border);
    border-radius: var(--radius);

    :global(.n8n-markdown) {
        width: 100%;
    }

    :global(code) {
        display: block;
        overflow-x: auto;
        font-size: var(--font-size--2xs);
        white-space: pre-wrap;
    }
}
</style>
```

- [ ] **Step 4: Run the tests to verify they pass**

```bash
pnpm test src/features/ai/mcpAccess/components/onboarding/MCPOnboardingClientSetup.test.ts
```

Expected: PASS for all four tests.

- [ ] **Step 5: If `${N8N_MCP_TOKEN}` does not appear literally in the rendered output**

If the test asserting `'Authorization: Bearer ${N8N_MCP_TOKEN}'` fails because vue-i18n stripped or warned on the unresolved `{N8N_MCP_TOKEN}` placeholder (the i18n value contains `{N8N_MCP_TOKEN}` and vue-i18n's named formatter may try to interpolate it), STOP and switch to manual string substitution:

In `MCPOnboardingClientSetup.vue`, replace the `promptBody` computed with:

```ts
const promptBody = computed(() => {
    const template = i18n.baseText(promptKey.value);
    return template
        .replaceAll('{token}', tokenValue.value)
        .replaceAll('{serverUrl}', props.serverUrl);
});
```

Then update the i18n values for `prompt.claudeCode` and `prompt.codex` to escape the env-var placeholder: change `${N8N_MCP_TOKEN}` to `$\u007BN8N_MCP_TOKEN\u007D` is **not** needed — vue-i18n is bypassed by the manual substitution above, so the original i18n values are kept verbatim.

Re-run the tests to verify they pass.

- [ ] **Step 6: Commit**

```bash
git add packages/frontend/editor-ui/src/features/ai/mcpAccess/components/onboarding/MCPOnboardingClientSetup.vue \
        packages/frontend/editor-ui/src/features/ai/mcpAccess/components/onboarding/MCPOnboardingClientSetup.test.ts
git commit -m "refactor(editor): render agent prompt in MCP onboarding setup component"
```

---

## Task 4: Update `MCPOnboardingModal.vue` to use the toggle and own token fetch

**Files:**

- Modify: `packages/frontend/editor-ui/src/features/ai/mcpAccess/modals/MCPOnboardingModal.vue`
- Modify: `packages/frontend/editor-ui/src/features/ai/mcpAccess/modals/MCPOnboardingModal.test.ts`

The modal swaps the `N8nButton` for `McpAccessToggle`, owns `getOrCreateApiKey()` (previously called from the removed `MCPAccessTokenPopoverTab`), forwards copy events to `trackCopiedParameter(..., 'agent-prompt')`, and shows a warning notice when the resolved token is redacted.

- [ ] **Step 1: Update the existing tests in `MCPOnboardingModal.test.ts`**

The existing `vi.mock` setup for clipboard, toast, root store, and the experiment store (lines 7–41) all stay as-is. Update the render stubs so `ElSwitch` behaves predictably in jsdom, then replace the `MockMcpStore` type and the test bodies.

Replace the current `renderComponent` block (lines 69–80) with:

```ts
const renderComponent = createComponentRenderer(MCPOnboardingModal, {
    props: {
        data: {
            surface: 'first_open_modal',
        },
    },
    global: {
        stubs: {
            Modal: ModalStub,
            ElSwitch: {
                props: ['modelValue', 'disabled', 'loading'],
                template: `
                    <button
                        type="button"
                        role="switch"
                        :data-test-id="$attrs['data-test-id']"
                        :aria-checked="String(!!modelValue)"
                        :disabled="disabled || loading"
                        @click="$emit('update:model-value', !modelValue)"
                    />
                `,
            },
        },
    },
});
```

Replace the mock-store typing (lines 43–57) with the new behaviour. Replace these two type/setup blocks:

```ts
type MockApiKey = { apiKey: string } | null;

type MockMcpStore = {
    mcpAccessEnabled: boolean;
    currentUserMCPKey: MockApiKey;
    setMcpAccessEnabled: ReturnType<typeof vi.fn>;
    getOrCreateApiKey: ReturnType<typeof vi.fn>;
    resetCurrentUserMCPKey: ReturnType<typeof vi.fn>;
};
```

with:

```ts
type MockApiKey = { apiKey: string } | null;

type MockMcpStore = {
    mcpAccessEnabled: boolean;
    mcpManagedByEnv: boolean;
    currentUserMCPKey: MockApiKey;
    setMcpAccessEnabled: ReturnType<typeof vi.fn>;
    getOrCreateApiKey: ReturnType<typeof vi.fn>;
    resetCurrentUserMCPKey: ReturnType<typeof vi.fn>;
};
```

In the `beforeEach` block (around line 88), include `mcpManagedByEnv` in the reactive store:

```ts
mockMcpStore = reactive({
    mcpAccessEnabled: false,
    mcpManagedByEnv: false,
    currentUserMCPKey: null,
    setMcpAccessEnabled: vi.fn().mockImplementation(async (next: boolean) => {
        mockMcpStore.mcpAccessEnabled = next;
        return next;
    }),
    getOrCreateApiKey: vi.fn().mockImplementation(async () => {
        mockMcpStore.currentUserMCPKey = { apiKey: 'n8n-test-token' };
        return mockMcpStore.currentUserMCPKey;
    }),
    resetCurrentUserMCPKey: vi.fn(),
}) as MockMcpStore;
```

Replace the three existing tests (lines 103–142) with these six tests:

```ts
it('enables MCP via the toggle and renders the prompt inline', async () => {
    const user = userEvent.setup();
    const { getByTestId, container } = renderComponent();

    await user.click(getByTestId('mcp-access-toggle'));

    expect(mockExperimentStore.trackEnableClicked).toHaveBeenCalledWith('first_open_modal');
    expect(mockMcpStore.setMcpAccessEnabled).toHaveBeenCalledWith(true);
    expect(mockExperimentStore.trackEnabled).toHaveBeenCalledWith('first_open_modal');
    expect(mockMcpStore.getOrCreateApiKey).toHaveBeenCalled();

    await waitFor(() => {
        expect(container.textContent ?? '').toContain('n8n-test-token');
    });
});

it('disables MCP via the toggle when already enabled', async () => {
    mockMcpStore.mcpAccessEnabled = true;
    mockMcpStore.currentUserMCPKey = { apiKey: 'n8n-test-token' };

    const user = userEvent.setup();
    const { getByTestId, queryByTestId } = renderComponent();

    expect(getByTestId('mcp-onboarding-client-setup')).toBeInTheDocument();

    await user.click(getByTestId('mcp-access-toggle'));

    expect(mockMcpStore.setMcpAccessEnabled).toHaveBeenCalledWith(false);
    expect(mockExperimentStore.trackEnableClicked).not.toHaveBeenCalled();
    expect(mockExperimentStore.trackEnabled).not.toHaveBeenCalled();

    await waitFor(() => {
        expect(queryByTestId('mcp-onboarding-client-setup')).not.toBeInTheDocument();
    });
    expect(getByTestId('mcp-onboarding-pending-notice')).toBeInTheDocument();
});

it('renders a disabled toggle when MCP is managed by environment', () => {
    mockMcpStore.mcpManagedByEnv = true;

    const { getByTestId } = renderComponent();

    expect(getByTestId('mcp-access-toggle')).toBeDisabled();
});

it('renders the prompt immediately when MCP is already enabled on mount', async () => {
    mockMcpStore.mcpAccessEnabled = true;
    mockMcpStore.currentUserMCPKey = null;
    mockMcpStore.getOrCreateApiKey = vi.fn().mockImplementation(async () => {
        mockMcpStore.currentUserMCPKey = { apiKey: 'existing-token' };
        return mockMcpStore.currentUserMCPKey;
    });

    const { container } = renderComponent();

    await waitFor(() => {
        expect(mockMcpStore.getOrCreateApiKey).toHaveBeenCalled();
        expect(container.textContent ?? '').toContain('existing-token');
    });
});

it('switches between Claude Code and Codex prompts', async () => {
    const user = userEvent.setup();
    mockMcpStore.mcpAccessEnabled = true;
    mockMcpStore.currentUserMCPKey = { apiKey: 'n8n-test-token' };

    const { getByText, container } = renderComponent();

    await user.click(getByText('Codex'));

    expect(mockExperimentStore.trackClientSelected).toHaveBeenCalledWith('codex');
    expect(container.textContent ?? '').toContain('[mcp_servers.n8n]');
});

it('forwards prompt copy to trackCopiedParameter with agent-prompt', async () => {
    const user = userEvent.setup();
    mockMcpStore.mcpAccessEnabled = true;
    mockMcpStore.currentUserMCPKey = { apiKey: 'n8n-test-token' };

    const { getByTestId } = renderComponent();

    await user.click(getByTestId('mcp-onboarding-copy-prompt-button'));

    expect(mockExperimentStore.trackCopiedParameter).toHaveBeenCalledWith(
        'first_open_modal',
        'claude_code',
        'agent-prompt',
    );
});
```

- [ ] **Step 2: Run the modal tests to verify they fail**

```bash
pnpm test src/features/ai/mcpAccess/modals/MCPOnboardingModal.test.ts
```

Expected: FAIL — current modal still renders `Enable MCP access` button, still renders `MCPAccessTokenPopoverTab`, no `mcp-access-toggle` test id, no automatic `getOrCreateApiKey()` call from the modal itself.

- [ ] **Step 3: Rewrite `MCPOnboardingModal.vue`**

Replace the entire `<script setup>` block (lines 1–122) with:

```vue
<script setup lang="ts">
import Modal from '@/app/components/Modal.vue';
import { useToast } from '@/app/composables/useToast';
import { useSurfaceMcpToNewCloudUsersStore } from '@/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store';
import McpAccessToggle from '@/features/ai/mcpAccess/components/header/McpAccessToggle.vue';
import MCPOnboardingClientSetup from '@/features/ai/mcpAccess/components/onboarding/MCPOnboardingClientSetup.vue';
import { MCP_ENDPOINT, MCP_ONBOARDING_MODAL_KEY } from '@/features/ai/mcpAccess/mcp.constants';
import { useMCPStore } from '@/features/ai/mcpAccess/mcp.store';
import { N8nNotice, N8nRadioButtons, N8nText } from '@n8n/design-system';
import { useI18n } from '@n8n/i18n';
import { useRootStore } from '@n8n/stores/useRootStore';
import { createEventBus } from '@n8n/utils/event-bus';
import { computed, onBeforeUnmount, onMounted, ref } from 'vue';

type MCPOnboardingClient = 'claude_code' | 'codex';
type MCPOnboardingSurface = 'tile' | 'first_open_modal';

const props = defineProps<{
    data?: {
        surface?: MCPOnboardingSurface;
    };
}>();

const i18n = useI18n();
const toast = useToast();
const rootStore = useRootStore();
const mcpStore = useMCPStore();
const experimentStore = useSurfaceMcpToNewCloudUsersStore();
const modalBus = createEventBus();

const activeClient = ref<MCPOnboardingClient>('claude_code');
const isToggling = ref(false);
const enabledDuringThisOpen = ref(false);

const surface = computed<MCPOnboardingSurface>(() => props.data?.surface ?? 'tile');

const clientOptions = computed(() => [
    {
        value: 'claude_code',
        label: i18n.baseText('settings.mcp.onboarding.client.claudeCode'),
    },
    {
        value: 'codex',
        label: i18n.baseText('settings.mcp.onboarding.client.codex'),
    },
]);

const serverUrl = computed(() => `${rootStore.urlBaseEditor}${MCP_ENDPOINT}`);

const isKeyRedacted = computed(
    () => mcpStore.currentUserMCPKey?.apiKey?.includes('******') ?? false,
);

const hasResolvedAccessToken = computed(
    () => Boolean(mcpStore.currentUserMCPKey?.apiKey) && !isKeyRedacted.value,
);

const accessToken = computed(() => {
    const token = mcpStore.currentUserMCPKey?.apiKey ?? '';
    return isKeyRedacted.value ? '' : token;
});

async function fetchApiKeySafely() {
    try {
        await mcpStore.getOrCreateApiKey();
    } catch (error) {
        toast.showError(error, i18n.baseText('settings.mcp.error.fetching.apiKey'));
    }
}

async function handleToggle() {
    const nextValue = !mcpStore.mcpAccessEnabled;

    try {
        isToggling.value = true;

        if (nextValue) {
            experimentStore.trackEnableClicked(surface.value);
        }

        const updated = await mcpStore.setMcpAccessEnabled(nextValue);

        if (!updated && nextValue) {
            return;
        }

        if (nextValue) {
            enabledDuringThisOpen.value = true;
            experimentStore.trackEnabled(surface.value);
            await fetchApiKeySafely();
        }
    } catch (error) {
        toast.showError(error, i18n.baseText('settings.mcp.toggle.error'));
    } finally {
        isToggling.value = false;
    }
}

function handleModalClosed() {
    if (
        surface.value === 'first_open_modal' &&
        !enabledDuringThisOpen.value &&
        !mcpStore.mcpAccessEnabled
    ) {
        experimentStore.dismissFirstOpenModal();
        experimentStore.trackDismissed(surface.value);
    }

    mcpStore.resetCurrentUserMCPKey();
}

function handleClientChange(value: string) {
    activeClient.value = value as MCPOnboardingClient;
    experimentStore.trackClientSelected(activeClient.value);
}

function handlePromptCopy() {
    experimentStore.trackCopiedParameter(surface.value, activeClient.value, 'agent-prompt');
}

onMounted(async () => {
    modalBus.on('closed', handleModalClosed);

    if (mcpStore.mcpAccessEnabled && !mcpStore.currentUserMCPKey) {
        await fetchApiKeySafely();
    }
});

onBeforeUnmount(() => {
    modalBus.off('closed', handleModalClosed);
    mcpStore.resetCurrentUserMCPKey();
});
</script>
```

- [ ] **Step 4: Replace the `<template>` block**

Replace the existing `<template>` (lines 124–185) with:

```vue
<template>
    <Modal
        :name="MCP_ONBOARDING_MODAL_KEY"
        :title="i18n.baseText('settings.mcp.onboarding.title')"
        width="720px"
        :event-bus="modalBus"
        :close-on-click-modal="true"
    >
        <template #content>
            <div :class="$style.content" data-test-id="mcp-onboarding-modal-content">
                <div :class="$style.summary">
                    <N8nText tag="p" size="large" color="text-base">
                        {{ i18n.baseText('settings.mcp.onboarding.description') }}
                    </N8nText>
                </div>

                <div :class="$style.controls">
                    <N8nRadioButtons
                        data-test-id="mcp-onboarding-client-switcher"
                        :model-value="activeClient"
                        :options="clientOptions"
                        @update:model-value="handleClientChange"
                    />

                    <McpAccessToggle
                        :model-value="mcpStore.mcpAccessEnabled"
                        :disabled="mcpStore.mcpManagedByEnv"
                        :loading="isToggling"
                        :managed-by-env="mcpStore.mcpManagedByEnv"
                        @disable-mcp-access="handleToggle"
                    />
                </div>

                <N8nNotice
                    v-if="!mcpStore.mcpAccessEnabled"
                    theme="info"
                    data-test-id="mcp-onboarding-pending-notice"
                >
                    {{ i18n.baseText('settings.mcp.onboarding.pending') }}
                </N8nNotice>

                <MCPOnboardingClientSetup
                    v-if="mcpStore.mcpAccessEnabled"
                    :client="activeClient"
                    :server-url="serverUrl"
                    :access-token="accessToken"
                    :is-token-ready="hasResolvedAccessToken"
                    @copy="handlePromptCopy"
                />
            </div>
        </template>
    </Modal>
</template>
```

- [ ] **Step 5: Drop the `connectionDetails` style rule**

The `<style module>` block at the bottom of the file (lines 187–214) currently includes a `.connectionDetails` rule that is no longer used. Replace the entire `<style module>` block with:

```vue
<style lang="scss" module>
.content {
    display: flex;
    flex-direction: column;
    gap: var(--spacing--sm);
}

.summary {
    max-width: 68ch;
}

.controls {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: var(--spacing--sm);

    @media (max-width: 768px) {
        flex-direction: column;
        align-items: stretch;
    }
}
</style>
```

- [ ] **Step 6: Run the modal tests to verify they pass**

```bash
pnpm test src/features/ai/mcpAccess/modals/MCPOnboardingModal.test.ts
```

Expected: PASS for all six tests.

- [ ] **Step 7: Re-run the prompt component tests to verify no regression**

```bash
pnpm test src/features/ai/mcpAccess/components/onboarding/MCPOnboardingClientSetup.test.ts
```

Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add packages/frontend/editor-ui/src/features/ai/mcpAccess/modals/MCPOnboardingModal.vue \
        packages/frontend/editor-ui/src/features/ai/mcpAccess/modals/MCPOnboardingModal.test.ts
git commit -m "refactor(editor): replace MCP onboarding button with toggle and prompt"
```

---

## Task 5: Update the original experiment design doc

**Files:**

- Modify: `docs/superpowers/specs/2026-05-01-surface-mcp-to-new-cloud-users-design.md:130-170`

- [ ] **Step 1: Add a note pointing to the revised onboarding spec**

Insert this block immediately after `### Shared setup experience`:

```md
> Note (2026-05-05): The onboarding-modal design changed after this
> draft. The current source of truth for the modal is
> `docs/superpowers/specs/2026-05-05-mcp-onboarding-toggle-and-agent-prompt-design.md`.
> It replaces the one-shot enable button, connection details panel, and
> two-card manual setup with a bidirectional toggle and a single agent
> setup prompt.
```

- [ ] **Step 2: Verify the doc still formats cleanly**

Run from the repository root:

```bash
pnpm exec prettier --check docs/superpowers/specs/2026-05-01-surface-mcp-to-new-cloud-users-design.md
```

Expected: PASS with Prettier reporting the file is correctly formatted.

- [ ] **Step 3: Commit**

```bash
git add docs/superpowers/specs/2026-05-01-surface-mcp-to-new-cloud-users-design.md
git commit -m "docs(spec): point MCP experiment doc to revised onboarding design"
```

---

## Task 6: Repository-wide lint and typecheck

**Files:** none (verification only).

- [ ] **Step 1: Typecheck the editor-ui package**

Run from `packages/frontend/editor-ui`:

```bash
pnpm typecheck
```

Expected: PASS. If any error references a removed i18n key, the offending consumer was missed in earlier tasks — `grep -r 'settings.mcp.onboarding.enable' packages/frontend/editor-ui/src` to find leftovers and remove them. Repeat for `step.env`, `step.config`, `claudeCode.path`, `codex.path`.

- [ ] **Step 2: Lint the editor-ui package**

Run from `packages/frontend/editor-ui`:

```bash
pnpm lint
```

Expected: PASS. Fix anything reported (most likely unused imports left over after the modal refactor).

- [ ] **Step 3: Run the full MCP-access feature test directory**

Run from `packages/frontend/editor-ui`:

```bash
pnpm test src/features/ai/mcpAccess
```

Expected: PASS. This catches any unintended regression in `SettingsMCPView.test.ts`, `MCPConnectWorkflowsModal.test.ts`, `mcp.store.test.ts`, `WorkflowLocation.test.ts`, `MCPWorkflowsSelect.test.ts`, and the connect-popover tests, because those still use the same MCP store.

- [ ] **Step 4: Run the experiment store tests**

```bash
pnpm test src/experiments/surfaceMcpToNewCloudUsers
```

Expected: PASS.

- [ ] **Step 5: Commit any small fixups (only if needed)**

If steps 1, 2, or 3 surfaced changes, commit them:

```bash
git add -A
git commit -m "chore(editor): clean up MCP onboarding refactor leftovers"
```

If nothing changed, skip this step.

- [ ] **Step 6: Manual smoke check for `claude mcp add` literal preservation**

The spec calls out that the single-quoted `claude mcp add` command must store `${N8N_MCP_TOKEN}` literally in the user-scope config so Claude Code resolves it at connect time, not at registration time. This is a runtime property of the Claude Code CLI; no automated test in this plan covers it. Verify manually before merge:

1. Spin up the editor-ui dev server, open the onboarding modal, toggle MCP on, and copy the Claude Code prompt.
2. Paste the prompt into a Claude Code session against this n8n instance.
3. After Claude Code runs `claude mcp add ...`, inspect the user-scope config (typically `~/.claude.json`; path may vary by Claude Code version) and confirm the stored header value contains the literal text `${N8N_MCP_TOKEN}`, not the expanded token.
4. Restart Claude Code and run a workflow tool from the n8n MCP server to confirm the env-var indirection actually resolves at connect time.

If step 3 shows the expanded token instead of the literal placeholder, **stop and update both this plan and `docs/superpowers/specs/2026-05-05-mcp-onboarding-toggle-and-agent-prompt-design.md` before shipping.** Do not merge the current Claude Code flow as-is.

---

## Done condition

- All seven test files in the affected paths pass.
- `pnpm typecheck` and `pnpm lint` succeed for `packages/frontend/editor-ui`.
- The onboarding modal renders the toggle, renders the agent prompt panel only when MCP is enabled, and copies a prompt body containing `claude mcp add --scope user` (Claude Code) or `[mcp_servers.n8n]` (Codex) with the resolved token interpolated and the literal `${N8N_MCP_TOKEN}` placeholder preserved in the header.
- When `getOrCreateApiKey()` returns a redacted token, the modal shows a recovery notice pointing users to `Settings > MCP` and keeps prompt copy disabled.
- No code path references the dropped i18n keys (`settings.mcp.onboarding.enable`, `step.env`, `step.config`, `claudeCode.path`, `codex.path`).
- `docs/superpowers/specs/2026-05-01-surface-mcp-to-new-cloud-users-design.md` contains a short note pointing to the revised onboarding-modal spec.
- Each task ends in a single commit with a Conventional Commits prefix matching the existing repo style (`feat`, `refactor`, `chore`).
