# Cursor MCP Client Setup Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add Cursor as a user-global MCP client setup option in the existing MCP onboarding modal.

**Architecture:** Extend the existing client union types, i18n keys, and setup prompt selection. Keep the UI structure unchanged: the same radio-button picker swaps a client-specific prompt, and copy telemetry keeps using the selected client.

**Tech Stack:** Vue 3, TypeScript, Pinia, Vitest, `@n8n/i18n`, `@n8n/design-system`, pnpm.

---

## File Structure

- Modify `packages/frontend/editor-ui/src/features/ai/mcpAccess/modals/MCPOnboardingModal.test.ts`: add focused coverage for Cursor selection and copy telemetry.
- Modify `packages/frontend/editor-ui/src/features/ai/mcpAccess/modals/MCPOnboardingModal.vue`: add Cursor to the onboarding client union and radio options.
- Modify `packages/frontend/editor-ui/src/features/ai/mcpAccess/components/onboarding/MCPOnboardingClientSetup.vue`: replace two-client conditionals with client-to-i18n-key maps and support Cursor prompt generation.
- Modify `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.ts`: add `cursor` to telemetry client typing.
- Modify `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.test.ts`: assert Cursor copy telemetry payload is supported.
- Modify `packages/frontend/@n8n/i18n/src/locales/en.json`: add Cursor label and setup prompt; update client-list copy that still says only Claude Code or Codex.
- Modify `docs/superpowers/specs/2026-05-06-cursor-mcp-client-setup-design.md`: check off the implementation checklist after implementation.

---

### Task 1: Add Failing Cursor Modal Coverage

**Files:**
- Modify: `packages/frontend/editor-ui/src/features/ai/mcpAccess/modals/MCPOnboardingModal.test.ts`

- [x] **Step 1: Add Cursor selection test**

Add this test after the existing `switches between Claude Code and Codex setup instructions` test:

```ts
it('switches to Cursor setup instructions', async () => {
	const user = userEvent.setup();
	mockMcpStore.mcpAccessEnabled = true;
	mockMcpStore.currentUserMCPKey = { apiKey: 'n8n-test-token' };

	const { getByText, container } = renderComponent();

	await user.click(getByText('Cursor'));

	expect(mockExperimentStore.trackClientSelected).toHaveBeenCalledWith('cursor');
	expect(container.textContent).toContain('~/.cursor/mcp.json');
	expect(container.textContent).toContain('Bearer ${env:N8N_MCP_TOKEN}');
});
```

- [x] **Step 2: Update copy telemetry test for Cursor**

Change the existing `forwards prompt copy telemetry with the selected client` test to select Cursor and expect Cursor telemetry:

```ts
it('forwards prompt copy telemetry with the selected client', async () => {
	const user = userEvent.setup();
	mockMcpStore.mcpAccessEnabled = true;
	mockMcpStore.currentUserMCPKey = { apiKey: 'n8n-test-token' };

	const { getByText, getByTestId } = renderComponent();

	await user.click(getByText('Cursor'));
	await user.click(getByTestId('mcp-onboarding-copy-prompt-button'));

	expect(mockExperimentStore.trackCopiedParameter).toHaveBeenCalledWith(
		'first_open_modal',
		'cursor',
		'agent-prompt',
	);
});
```

- [x] **Step 3: Run the focused test and verify it fails**

Run:

```bash
pnpm test src/features/ai/mcpAccess/modals/MCPOnboardingModal.test.ts
```

Expected: FAIL because `Cursor` is not in the client picker yet.

---

### Task 2: Implement Cursor Client Support

**Files:**
- Modify: `packages/frontend/editor-ui/src/features/ai/mcpAccess/modals/MCPOnboardingModal.vue`
- Modify: `packages/frontend/editor-ui/src/features/ai/mcpAccess/components/onboarding/MCPOnboardingClientSetup.vue`
- Modify: `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.ts`
- Modify: `packages/frontend/@n8n/i18n/src/locales/en.json`

- [x] **Step 1: Add Cursor to modal client type and options**

In `MCPOnboardingModal.vue`, change:

```ts
type MCPOnboardingClient = 'claude_code' | 'codex';
```

to:

```ts
type MCPOnboardingClient = 'claude_code' | 'cursor' | 'codex';
```

Update `clientOptions` to include Cursor between Claude Code and Codex:

```ts
const clientOptions = computed(() => [
	{
		value: 'claude_code',
		label: i18n.baseText('settings.mcp.onboarding.client.claudeCode'),
	},
	{
		value: 'cursor',
		label: i18n.baseText('settings.mcp.onboarding.client.cursor'),
	},
	{
		value: 'codex',
		label: i18n.baseText('settings.mcp.onboarding.client.codex'),
	},
]);
```

- [x] **Step 2: Add Cursor to setup component mapping**

In `MCPOnboardingClientSetup.vue`, change the client type to:

```ts
type MCPOnboardingClient = 'claude_code' | 'cursor' | 'codex';
```

Add maps after `tokenValue`:

```ts
const promptKeys: Record<MCPOnboardingClient, string> = {
	claude_code: 'settings.mcp.onboarding.prompt.claudeCode',
	cursor: 'settings.mcp.onboarding.prompt.cursor',
	codex: 'settings.mcp.onboarding.prompt.codex',
};

const clientLabelKeys: Record<MCPOnboardingClient, string> = {
	claude_code: 'settings.mcp.onboarding.client.claudeCode',
	cursor: 'settings.mcp.onboarding.client.cursor',
	codex: 'settings.mcp.onboarding.client.codex',
};
```

Replace the existing `promptKey` computed with:

```ts
const promptKey = computed(() => promptKeys[props.client]);
```

Replace the existing `clientLabel` computed with:

```ts
const clientLabel = computed(() => i18n.baseText(clientLabelKeys[props.client]));
```

- [x] **Step 3: Add Cursor to telemetry type**

In `surfaceMcpToNewCloudUsers.store.ts`, change:

```ts
type SurfaceMcpOnboardingClient = 'claude_code' | 'codex';
```

to:

```ts
type SurfaceMcpOnboardingClient = 'claude_code' | 'cursor' | 'codex';
```

- [x] **Step 4: Add Cursor i18n keys and update relevant copy**

In `en.json`, add:

```json
"settings.mcp.onboarding.client.cursor": "Cursor",
"settings.mcp.onboarding.prompt.cursor": "Set up the n8n MCP server for Cursor globally.\n\n1. Add the environment variable to your shell config (for example, ~/.zshrc or ~/.bashrc), then reload your shell:\n\n   export N8N_MCP_TOKEN=\"{token}\"\n\n2. Add the following to ~/.cursor/mcp.json (create the file and directory if they do not exist; if the file already exists, merge the n8n server entry without removing existing config):\n\n   {\n     \"mcpServers\": {\n       \"n8n\": {\n         \"url\": \"{serverUrl}\",\n         \"headers\": {\n           \"Authorization\": \"Bearer ${env:N8N_MCP_TOKEN}\"\n         }\n       }\n     }\n   }\n\n3. Preserve ${env:N8N_MCP_TOKEN} literally in the JSON config. Do not replace it with the token value.\n\n4. When you finish, tell me to restart Cursor. Do not try to verify the server in this session."
```

Also update strings that describe the setup client list:

```json
"settings.mcp.onboarding.intro.title": "Try MCP with Claude Code, Cursor, or Codex",
"workflows.empty.mcp.tile.enabledDescription": "Reopen the setup steps for Claude Code, Cursor, or Codex."
```

- [x] **Step 5: Run the focused modal test and verify it passes**

Run:

```bash
pnpm test src/features/ai/mcpAccess/modals/MCPOnboardingModal.test.ts
```

Expected: PASS.

---

### Task 3: Add Store Type Coverage and Final Verification

**Files:**
- Modify: `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.test.ts`
- Modify: `docs/superpowers/specs/2026-05-06-cursor-mcp-client-setup-design.md`

- [x] **Step 1: Update store test to assert Cursor telemetry**

In `surfaceMcpToNewCloudUsers.store.test.ts`, change the copied parameter test to call Cursor:

```ts
store.trackCopiedParameter('first_open_modal', 'cursor', 'agent-prompt');

expect(mockTrack).toHaveBeenCalledWith('MCP onboarding copied parameter', {
	surface: 'first_open_modal',
	client: 'cursor',
	parameter: 'agent-prompt',
	variant: SURFACE_MCP_TO_NEW_CLOUD_USERS_EXPERIMENT.variantFirstOpenModal,
});
```

- [x] **Step 2: Check off completed spec checklist items**

In `docs/superpowers/specs/2026-05-06-cursor-mcp-client-setup-design.md`, change every implementation checklist item from `- [ ]` to `- [x]` after the implementation and verification pass.

- [x] **Step 3: Run focused store test**

Run:

```bash
pnpm test src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.test.ts
```

Expected: PASS.

- [x] **Step 4: Run focused modal test again**

Run:

```bash
pnpm test src/features/ai/mcpAccess/modals/MCPOnboardingModal.test.ts
```

Expected: PASS.

- [x] **Step 5: Run package typecheck**

Run:

```bash
pnpm typecheck
```

Expected: PASS.

---

## Notes

- Do not create a git commit unless explicitly requested.
- Do not change backend MCP APIs.
- Do not add a project-local Cursor option.
- Preserve `${env:N8N_MCP_TOKEN}` literally in Cursor config copy.
