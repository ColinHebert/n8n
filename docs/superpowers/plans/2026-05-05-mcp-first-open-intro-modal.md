# MCP First-open Intro Modal Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a lightweight opt-in intro modal for the `variant-first-open-modal` experiment arm, then open the existing MCP setup modal when the user clicks `Try MCP`.

**Architecture:** Keep the tile flow and the existing setup modal unchanged. Add one experiment-scoped intro modal, register it through the existing MCP module descriptor, and change `WorkflowsView.vue` so the first-open auto-open path targets the intro modal instead of the setup modal. Reuse the existing experiment store dismissal state and the existing `MCPOnboardingModal.vue` setup flow.

**Tech Stack:** Vue 3, Pinia, `@n8n/design-system` (`Modal`, `N8nButton`, `N8nText`), `@n8n/i18n`, `@n8n/utils/event-bus`, vitest, `@testing-library/vue`.

**Spec:** `docs/superpowers/specs/2026-05-05-mcp-onboarding-toggle-and-agent-prompt-design.md`

---

## File map

**Create:**

- `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/constants.ts` — intro modal key shared by the component, tests, and `WorkflowsView.vue`.
- `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/components/SurfaceMcpFirstOpenIntroModal.vue` — experiment-scoped intro modal that gates the first-open flow.
- `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/components/SurfaceMcpFirstOpenIntroModal.test.ts` — component test for `Try MCP`, `Skip for now`, and generic close behavior.

**Modify:**

- `packages/frontend/@n8n/i18n/src/locales/en.json` — add intro modal copy keys.
- `packages/frontend/editor-ui/src/features/ai/mcpAccess/module.descriptor.ts` — register the new intro modal alongside the existing MCP modals.
- `packages/frontend/editor-ui/src/app/views/WorkflowsView.vue` — auto-open the intro modal for the first-open variant.
- `packages/frontend/editor-ui/src/app/views/WorkflowsView.test.ts` — update the first-open variant tests to target the intro modal.

---

### Task 1: Add intro modal key and i18n copy

**Files:**
- Create: `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/constants.ts`
- Modify: `packages/frontend/@n8n/i18n/src/locales/en.json`

- [ ] **Step 1: Run a failing assertion for the new intro-copy keys**

Run from the repository root:

```bash
node - <<'EOF'
const fs = require('fs');
const json = JSON.parse(fs.readFileSync('packages/frontend/@n8n/i18n/src/locales/en.json', 'utf8'));
for (const key of [
  'settings.mcp.onboarding.intro.title',
  'settings.mcp.onboarding.intro.description',
  'settings.mcp.onboarding.intro.tryIt',
  'settings.mcp.onboarding.intro.skip',
]) {
  if (!(key in json)) throw new Error(`Missing ${key}`);
}
EOF
```

Expected: FAIL on the first missing intro key.

- [ ] **Step 2: Create the experiment constants file**

Create `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/constants.ts` with:

```ts
export const SURFACE_MCP_FIRST_OPEN_INTRO_MODAL_KEY = 'surfaceMcpFirstOpenIntroModal';
```

- [ ] **Step 3: Add the intro modal copy to `en.json`**

In the `settings.mcp.onboarding.*` block, add these lines immediately before `settings.mcp.onboarding.prompt.title`:

```json
"settings.mcp.onboarding.intro.title": "Try MCP with Claude Code or Codex",
"settings.mcp.onboarding.intro.description": "Use MCP to connect Claude Code or Codex to this n8n instance. You can set it up now, or turn it on later in Settings > MCP.",
"settings.mcp.onboarding.intro.tryIt": "Try MCP",
"settings.mcp.onboarding.intro.skip": "Skip for now",
```

- [ ] **Step 4: Verify the new keys and JSON both pass**

Run from the repository root:

```bash
node - <<'EOF'
const fs = require('fs');
const json = JSON.parse(fs.readFileSync('packages/frontend/@n8n/i18n/src/locales/en.json', 'utf8'));
for (const [key, value] of Object.entries({
  'settings.mcp.onboarding.intro.title': 'Try MCP with Claude Code or Codex',
  'settings.mcp.onboarding.intro.description': 'Use MCP to connect Claude Code or Codex to this n8n instance. You can set it up now, or turn it on later in Settings > MCP.',
  'settings.mcp.onboarding.intro.tryIt': 'Try MCP',
  'settings.mcp.onboarding.intro.skip': 'Skip for now',
})) {
  if (json[key] !== value) throw new Error(`Unexpected value for ${key}`);
}
EOF
pnpm --dir packages/frontend/@n8n/i18n typecheck
```

Expected: PASS. The node check exits silently, and the i18n package typecheck passes.

- [ ] **Step 5: Commit**

```bash
git add packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/constants.ts \
        packages/frontend/@n8n/i18n/src/locales/en.json
git commit -m "feat(editor): add MCP first-open intro modal copy"
```

---

### Task 2: Build the first-open intro modal component

**Files:**
- Create: `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/components/SurfaceMcpFirstOpenIntroModal.vue`
- Create: `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/components/SurfaceMcpFirstOpenIntroModal.test.ts`

- [ ] **Step 1: Write the failing component test file**

Create `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/components/SurfaceMcpFirstOpenIntroModal.test.ts` with:

```ts
import { createComponentRenderer } from '@/__tests__/render';
import { mockedStore } from '@/__tests__/utils';
import { useUIStore } from '@/app/stores/ui.store';
import { MCP_ONBOARDING_MODAL_KEY } from '@/features/ai/mcpAccess/mcp.constants';
import { useSurfaceMcpToNewCloudUsersStore } from '@/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store';
import { SURFACE_MCP_FIRST_OPEN_INTRO_MODAL_KEY } from '@/experiments/surfaceMcpToNewCloudUsers/constants';
import { createTestingPinia } from '@pinia/testing';
import userEvent from '@testing-library/user-event';
import { defineComponent } from 'vue';
import SurfaceMcpFirstOpenIntroModal from './SurfaceMcpFirstOpenIntroModal.vue';

const ModalStub = defineComponent({
  props: ['name', 'title', 'eventBus'],
  template: `
    <div :data-test-id="name">
      <h1>{{ title }}</h1>
      <slot name="content" />
      <slot name="footer" />
      <button data-test-id="surface-mcp-intro-generic-close" @click="eventBus.emit('closed')" />
    </div>
  `,
});

const renderComponent = createComponentRenderer(SurfaceMcpFirstOpenIntroModal, {
  props: {
    modalName: SURFACE_MCP_FIRST_OPEN_INTRO_MODAL_KEY,
  },
  global: {
    stubs: {
      Modal: ModalStub,
    },
  },
});

describe('SurfaceMcpFirstOpenIntroModal', () => {
  let pinia: ReturnType<typeof createTestingPinia>;

  beforeEach(() => {
    pinia = createTestingPinia({ stubActions: false });
    const uiStore = mockedStore(useUIStore);
    const surfaceMcpStore = mockedStore(useSurfaceMcpToNewCloudUsersStore);

    uiStore.openModalWithData = vi.fn();
    uiStore.closeModal = vi.fn();
    surfaceMcpStore.dismissFirstOpenModal = vi.fn();
    surfaceMcpStore.trackDismissed = vi.fn();
  });

  it('renders the intro copy and both actions', () => {
    const { getByText, getByTestId } = renderComponent({ pinia });

    expect(getByText('Try MCP with Claude Code or Codex')).toBeInTheDocument();
    expect(
      getByText(
        'Use MCP to connect Claude Code or Codex to this n8n instance. You can set it up now, or turn it on later in Settings > MCP.',
      ),
    ).toBeInTheDocument();
    expect(getByTestId('surface-mcp-intro-skip-button')).toBeInTheDocument();
    expect(getByTestId('surface-mcp-intro-try-button')).toBeInTheDocument();
  });

  it('closes the intro modal and opens the setup modal when the user clicks Try MCP', async () => {
    const user = userEvent.setup();
    const uiStore = mockedStore(useUIStore);
    const surfaceMcpStore = mockedStore(useSurfaceMcpToNewCloudUsersStore);
    const { getByTestId } = renderComponent({ pinia });

    await user.click(getByTestId('surface-mcp-intro-try-button'));

    expect(uiStore.closeModal).toHaveBeenCalledWith(SURFACE_MCP_FIRST_OPEN_INTRO_MODAL_KEY);
    expect(uiStore.openModalWithData).toHaveBeenCalledWith({
      name: MCP_ONBOARDING_MODAL_KEY,
      data: { surface: 'first_open_modal' },
    });
    expect(surfaceMcpStore.dismissFirstOpenModal).not.toHaveBeenCalled();
    expect(surfaceMcpStore.trackDismissed).not.toHaveBeenCalled();
  });

  it('dismisses the experiment when the user clicks Skip for now', async () => {
    const user = userEvent.setup();
    const uiStore = mockedStore(useUIStore);
    const surfaceMcpStore = mockedStore(useSurfaceMcpToNewCloudUsersStore);
    const { getByTestId } = renderComponent({ pinia });

    await user.click(getByTestId('surface-mcp-intro-skip-button'));

    expect(surfaceMcpStore.dismissFirstOpenModal).toHaveBeenCalled();
    expect(surfaceMcpStore.trackDismissed).toHaveBeenCalledWith('first_open_modal');
    expect(uiStore.closeModal).toHaveBeenCalledWith(SURFACE_MCP_FIRST_OPEN_INTRO_MODAL_KEY);
    expect(uiStore.openModalWithData).not.toHaveBeenCalled();
  });

  it('treats a generic modal close as Skip for now', async () => {
    const user = userEvent.setup();
    const uiStore = mockedStore(useUIStore);
    const surfaceMcpStore = mockedStore(useSurfaceMcpToNewCloudUsersStore);
    const { getByTestId } = renderComponent({ pinia });

    await user.click(getByTestId('surface-mcp-intro-generic-close'));

    expect(surfaceMcpStore.dismissFirstOpenModal).toHaveBeenCalled();
    expect(surfaceMcpStore.trackDismissed).toHaveBeenCalledWith('first_open_modal');
    expect(uiStore.closeModal).not.toHaveBeenCalled();
    expect(uiStore.openModalWithData).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run the component tests to verify they fail**

Run from `packages/frontend/editor-ui`:

```bash
pnpm test src/experiments/surfaceMcpToNewCloudUsers/components/SurfaceMcpFirstOpenIntroModal.test.ts
```

Expected: FAIL because the component file does not exist yet.

- [ ] **Step 3: Create the intro modal component**

Create `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/components/SurfaceMcpFirstOpenIntroModal.vue` with:

```vue
<script setup lang="ts">
import Modal from '@/app/components/Modal.vue';
import { useUIStore } from '@/app/stores/ui.store';
import { MCP_ONBOARDING_MODAL_KEY } from '@/features/ai/mcpAccess/mcp.constants';
import { SURFACE_MCP_FIRST_OPEN_INTRO_MODAL_KEY } from '@/experiments/surfaceMcpToNewCloudUsers/constants';
import { useSurfaceMcpToNewCloudUsersStore } from '@/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store';
import { N8nButton, N8nText } from '@n8n/design-system';
import { useI18n } from '@n8n/i18n';
import { createEventBus } from '@n8n/utils/event-bus';
import { onBeforeUnmount, onMounted, ref } from 'vue';

const props = defineProps<{
  modalName?: string;
}>();

const i18n = useI18n();
const uiStore = useUIStore();
const surfaceMcpStore = useSurfaceMcpToNewCloudUsersStore();
const modalBus = createEventBus();
const exitMode = ref<'none' | 'skip' | 'try'>('none');

const modalName = props.modalName ?? SURFACE_MCP_FIRST_OPEN_INTRO_MODAL_KEY;

function dismissExperiment() {
  surfaceMcpStore.dismissFirstOpenModal();
  surfaceMcpStore.trackDismissed('first_open_modal');
}

function handleSkip() {
  if (exitMode.value !== 'none') {
    return;
  }

  exitMode.value = 'skip';
  dismissExperiment();
  uiStore.closeModal(modalName);
}

function handleTryIt() {
  if (exitMode.value !== 'none') {
    return;
  }

  exitMode.value = 'try';
  uiStore.closeModal(modalName);
  uiStore.openModalWithData({
    name: MCP_ONBOARDING_MODAL_KEY,
    data: { surface: 'first_open_modal' },
  });
}

function handleClosed() {
  if (exitMode.value !== 'none') {
    return;
  }

  exitMode.value = 'skip';
  dismissExperiment();
}

onMounted(() => {
  modalBus.on('closed', handleClosed);
});

onBeforeUnmount(() => {
  modalBus.off('closed', handleClosed);
});
</script>

<template>
  <Modal
    :name="modalName"
    :title="i18n.baseText('settings.mcp.onboarding.intro.title')"
    width="560px"
    :event-bus="modalBus"
    :close-on-click-modal="true"
    :close-on-press-escape="true"
  >
    <template #content>
      <div :class="$style.content">
        <N8nText tag="p" size="medium" color="text-base">
          {{ i18n.baseText('settings.mcp.onboarding.intro.description') }}
        </N8nText>
      </div>
    </template>

    <template #footer>
      <div :class="$style.footer">
        <N8nButton
          variant="subtle"
          data-test-id="surface-mcp-intro-skip-button"
          @click="handleSkip"
        >
          {{ i18n.baseText('settings.mcp.onboarding.intro.skip') }}
        </N8nButton>
        <N8nButton
          variant="solid"
          data-test-id="surface-mcp-intro-try-button"
          @click="handleTryIt"
        >
          {{ i18n.baseText('settings.mcp.onboarding.intro.tryIt') }}
        </N8nButton>
      </div>
    </template>
  </Modal>
</template>

<style lang="scss" module>
.content {
  display: flex;
  flex-direction: column;
  gap: var(--spacing--xs);
  max-width: 60ch;
}

.footer {
  display: flex;
  justify-content: flex-end;
  gap: var(--spacing--2xs);
}
</style>
```

- [ ] **Step 4: Run the component tests to verify they pass**

```bash
pnpm test src/experiments/surfaceMcpToNewCloudUsers/components/SurfaceMcpFirstOpenIntroModal.test.ts
```

Expected: PASS for all four tests.

- [ ] **Step 5: Commit**

```bash
git add packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/constants.ts \
        packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/components/SurfaceMcpFirstOpenIntroModal.vue \
        packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/components/SurfaceMcpFirstOpenIntroModal.test.ts
git commit -m "feat(editor): add MCP first-open intro modal"
```

---

### Task 3: Register the intro modal and reroute first-open auto-open

**Files:**
- Modify: `packages/frontend/editor-ui/src/features/ai/mcpAccess/module.descriptor.ts`
- Modify: `packages/frontend/editor-ui/src/app/views/WorkflowsView.vue`
- Modify: `packages/frontend/editor-ui/src/app/views/WorkflowsView.test.ts`

- [ ] **Step 1: Update the failing `WorkflowsView` tests**

In `packages/frontend/editor-ui/src/app/views/WorkflowsView.test.ts`, add this import near the other top-level imports:

```ts
import { SURFACE_MCP_FIRST_OPEN_INTRO_MODAL_KEY } from '@/experiments/surfaceMcpToNewCloudUsers/constants';
```

Then replace the first-open experiment tests at the end of the file with:

```ts
it('auto-opens the MCP intro modal once for the first-open variant', async () => {
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
  expect(surfaceMcpStore.trackSurfaced).toHaveBeenCalledWith('first_open_modal');
  expect(surfaceMcpStore.trackOpened).toHaveBeenCalledWith('first_open_modal');
  expect(uiStore.openModal).toHaveBeenCalledWith(SURFACE_MCP_FIRST_OPEN_INTRO_MODAL_KEY);
});

it('does not auto-open the intro modal for the tile variant', async () => {
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

  expect(uiStore.openModal).not.toHaveBeenCalled();
});

it('does not auto-open the intro modal after the first eligible open has already been seen', async () => {
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

  expect(uiStore.openModal).not.toHaveBeenCalled();
});
```

- [ ] **Step 2: Run the `WorkflowsView` tests to verify they fail**

Run from `packages/frontend/editor-ui`:

```bash
pnpm test src/app/views/WorkflowsView.test.ts
```

Expected: FAIL because `WorkflowsView.vue` still opens `mcpOnboardingModal` directly through `openModalWithData(...)`.

- [ ] **Step 3: Register the intro modal in the MCP module descriptor**

In `packages/frontend/editor-ui/src/features/ai/mcpAccess/module.descriptor.ts`, add this import:

```ts
import { SURFACE_MCP_FIRST_OPEN_INTRO_MODAL_KEY } from '@/experiments/surfaceMcpToNewCloudUsers/constants';
```

Then insert this modal definition before the existing `MCP_ONBOARDING_MODAL_KEY` block:

```ts
{
  key: SURFACE_MCP_FIRST_OPEN_INTRO_MODAL_KEY,
  component:
    async () =>
      await import('@/experiments/surfaceMcpToNewCloudUsers/components/SurfaceMcpFirstOpenIntroModal.vue'),
  initialState: { open: false },
},
```

- [ ] **Step 4: Change `WorkflowsView.vue` to open the intro modal**

In `packages/frontend/editor-ui/src/app/views/WorkflowsView.vue`, add this import near the other experiment imports:

```ts
import { SURFACE_MCP_FIRST_OPEN_INTRO_MODAL_KEY } from '@/experiments/surfaceMcpToNewCloudUsers/constants';
```

Then replace the final lines of `maybeOpenSurfaceMcpModal()` with:

```ts
surfaceMcpStore.markFirstEligibleOpenSeen();
surfaceMcpStore.trackSurfaced('first_open_modal');
surfaceMcpStore.trackOpened('first_open_modal');
uiStore.openModal(SURFACE_MCP_FIRST_OPEN_INTRO_MODAL_KEY);
```

- [ ] **Step 5: Run the `WorkflowsView` tests again**

```bash
pnpm test src/app/views/WorkflowsView.test.ts
```

Expected: PASS for the updated first-open tests and the rest of the file.

- [ ] **Step 6: Re-run the intro modal tests to verify no regression**

```bash
pnpm test src/experiments/surfaceMcpToNewCloudUsers/components/SurfaceMcpFirstOpenIntroModal.test.ts
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add packages/frontend/editor-ui/src/features/ai/mcpAccess/module.descriptor.ts \
        packages/frontend/editor-ui/src/app/views/WorkflowsView.vue \
        packages/frontend/editor-ui/src/app/views/WorkflowsView.test.ts
git commit -m "feat(editor): gate first-open MCP onboarding with intro modal"
```

---

### Task 4: Verify the first-open intro flow end-to-end

**Files:** none unless verification exposes a small regression.

- [ ] **Step 1: Typecheck the editor-ui package**

Run from `packages/frontend/editor-ui`:

```bash
pnpm typecheck
```

Expected: PASS.

- [ ] **Step 2: Lint the editor-ui package**

```bash
pnpm lint
```

Expected: PASS with zero errors. Warnings are acceptable if they are pre-existing and unrelated.

- [ ] **Step 3: Re-run the onboarding modal test suite**

```bash
pnpm test src/features/ai/mcpAccess/modals/MCPOnboardingModal.test.ts
```

Expected: PASS. The setup modal behavior should remain unchanged.

- [ ] **Step 4: Re-run the full experiment store tests**

```bash
pnpm test src/experiments/surfaceMcpToNewCloudUsers
```

Expected: PASS.

- [ ] **Step 5: Manually smoke-test the exact browser flow that motivated this change**

In the browser console on the empty workflows page, run:

```js
window.featureFlags.override(
  '080_surface_mcp_to_new_cloud_users',
  'variant-first-open-modal',
);

localStorage.removeItem('N8N_SURFACE_MCP_TO_NEW_CLOUD_USERS_FIRST_OPEN_SEEN');
localStorage.removeItem('N8N_SURFACE_MCP_TO_NEW_CLOUD_USERS_FIRST_OPEN_DISMISSED');

location.reload();
```

Verify these outcomes in order:

1. The first-open intro modal appears, not the setup modal.
2. Clicking `Try MCP` closes the intro modal and opens the existing setup MCP modal.
3. Repeat the override/reset snippet, reload again, click `Skip for now`, and verify the intro modal closes without opening the setup modal.
4. Verify the empty-state reminder text appears after skip.
5. Reload again **without** clearing those two localStorage keys and verify the intro modal does not auto-open anymore.

- [ ] **Step 6: Commit any small verification fixups (only if needed)**

If steps 1–5 expose a small implementation issue that you fix, commit it:

```bash
git add -A
git commit -m "fix(editor): polish first-open MCP intro modal flow"
```

If no code changed during verification, skip this step.

---

## Done condition

- The first-open variant auto-opens the new intro modal instead of opening the setup modal directly.
- Clicking `Try MCP` closes the intro modal and opens `MCPOnboardingModal` with `surface: 'first_open_modal'`.
- Clicking `Skip for now`, or closing the intro modal through a generic close path, dismisses the experiment and prevents future auto-open.
- The tile variant still opens the existing setup modal directly.
- `pnpm typecheck` and `pnpm lint` succeed for `packages/frontend/editor-ui`.
- `pnpm test src/app/views/WorkflowsView.test.ts`, `pnpm test src/experiments/surfaceMcpToNewCloudUsers/components/SurfaceMcpFirstOpenIntroModal.test.ts`, `pnpm test src/features/ai/mcpAccess/modals/MCPOnboardingModal.test.ts`, and `pnpm test src/experiments/surfaceMcpToNewCloudUsers` all pass.
