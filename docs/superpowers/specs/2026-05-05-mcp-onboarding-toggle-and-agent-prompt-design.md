# MCP Onboarding Modal: Toggle and Agent Prompt Design

## Summary

Refactor the surface-MCP onboarding modal so that:

1. Enabling MCP access uses a bidirectional toggle instead of a one-shot
   button.
2. The two-step manual setup (env var + config file) and the connection
   details panel (URL / token / Configuration JSON) are replaced with a
   single agent setup prompt that the user pastes into Claude Code or
   Codex. The agent performs the shell or config changes, then tells the
   user to restart the client to finish setup.

The change keeps the modal a single-screen experience and continues to
target the experiment's audience: Cloud trial admins running Claude Code
or Codex.

## Goals

- Reduce the onboarding flow from "toggle + read connection details +
  set env var + edit config file" to "toggle + paste prompt + restart
  the client".
- Use a toggle to make the enabled / disabled state explicit and
  reversible inside the onboarding surface.
- Register the MCP server **globally** (user scope) so it works in every
  project the user opens with their agent.
- Reuse existing MCP store logic and the existing `McpAccessToggle`
  component.
- Keep the change additive within the surface-MCP experiment files so it
  is easy to remove or graduate.

## Non-goals

- Redesign the existing `Settings > MCP` page.
- Add new MCP clients beyond Claude Code and Codex.
- Add a per-project setup option in this iteration.
- Solve the pre-existing token-redaction issue for re-enable flows
  (called out in Open issues).
- Change MCP store APIs.

## Current repo facts

- `MCPOnboardingModal.vue` lives at
  `packages/frontend/editor-ui/src/features/ai/mcpAccess/modals/MCPOnboardingModal.vue`.
  It currently renders:
  - a primary `Enable MCP access` `N8nButton`,
  - a connection details panel via `MCPAccessTokenPopoverTab`
    (server URL, access token, Configuration JSON, "you won't see it
    again" notice),
  - a setup component `MCPOnboardingClientSetup` that renders two
    cards: "Set an environment variable" and "Add this MCP config".
- `MCPOnboardingClientSetup.vue` lives at
  `packages/frontend/editor-ui/src/features/ai/mcpAccess/components/onboarding/MCPOnboardingClientSetup.vue`.
  It computes:
  - an `export N8N_MCP_TOKEN=...` snippet,
  - a Claude Code `.mcp.json` snippet,
  - a Codex `.codex/config.toml` snippet (project-local).
- `McpAccessToggle.vue` already exists at
  `packages/frontend/editor-ui/src/features/ai/mcpAccess/components/header/McpAccessToggle.vue`
  and supports `modelValue`, `disabled`, `loading`, `managedByEnv`. It is
  currently used in `SettingsMCPView.vue`.
- The MCP store
  (`packages/frontend/editor-ui/src/features/ai/mcpAccess/mcp.store.ts`)
  exposes `setMcpAccessEnabled(enabled: boolean)`, which already supports
  both directions, plus `mcpAccessEnabled` and `mcpManagedByEnv`
  computeds, and `getOrCreateApiKey()` / `generateNewApiKey()`.
- The experiment store at
  `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/stores/surfaceMcpToNewCloudUsers.store.ts`
  exposes telemetry helpers including
  `trackCopiedParameter(surface, client, parameter)` with
  `parameter` typed as `'server-url' | 'access-token' | 'setup-config'`.

## Product decisions

### Toggle replaces the enable button

Replace the `Enable MCP access` `N8nButton` with the existing
`McpAccessToggle` component. The toggle is bidirectional. Toggling on
enables MCP access. Toggling off disables it.

To get the correct env-managed behavior, the modal must pass both
`disabled` and `managedByEnv` to `McpAccessToggle`. When
`mcpManagedByEnv` is true, the toggle is disabled and the tooltip
explains that MCP access is managed by environment.

### Single agent prompt replaces manual steps and the connection details panel

The connection details panel (server URL, access token, Configuration
JSON, "you won't see it again" notice) is removed.

The two-step manual setup is removed.

In their place, a single panel renders a setup prompt that the user
copies and pastes into their agent (Claude Code or Codex). The agent
performs the file or CLI changes, then tells the user to restart the
client. Verification happens after restart, not in the same pasted
session.

The prompt embeds the server URL and access token inline. The user does
not need to copy raw connection values separately.

### Global setup, not project-local

The prompts target user-global setup so the MCP server is available in
all projects the agent opens, not only the current working directory.

This differs from the previous project-local `.mcp.json` flow.

### Env-var indirection is preserved

The token is referenced from the agent config via a literal
`${N8N_MCP_TOKEN}` placeholder for Claude Code and
`bearer_token_env_var` for Codex. The user is instructed to set the env
var in their shell config (e.g. `~/.zshrc`, `~/.bashrc`). The token does
not end up written into the config file as a literal string.

### Claude Code uses the `claude mcp add` CLI

For Claude Code, the prompt instructs the agent to run:

```
claude mcp add --scope user --transport http n8n "<server-url>" \
  --header 'Authorization: Bearer ${N8N_MCP_TOKEN}'
```

This avoids depending on Claude Code's user-scope config file path,
which can vary across versions, and lets the CLI handle merging into
existing configuration. The single quotes are required so the shell
passes `${N8N_MCP_TOKEN}` through literally instead of expanding it
before Claude Code stores the config.

### Codex edits `~/.codex/config.toml`

Codex does not have an equivalent `add` CLI. The prompt instructs the
agent to add (or merge) the following section to `~/.codex/config.toml`,
creating the file and directory if they do not exist:

```
[mcp_servers.n8n]
url = "<server-url>"
bearer_token_env_var = "N8N_MCP_TOKEN"
```

### No rotate affordance in the modal

The modal does not expose token rotation. Users who need to rotate go
to `Settings > MCP`. This decision is intentional for PoC scope.

The implication is documented under Open issues (token redaction on
re-enable).

### No telemetry for toggle-off

Disabling MCP from this modal does not emit a dedicated telemetry event
in this iteration. `trackEnableClicked` and `trackEnabled` continue to
fire only on toggle-on transitions. This keeps the experiment readout
focused on the metric of interest (enable rate by surface).

## Final modal layout

Top to bottom:

```
Set up MCP for your instance
Connect Claude Code or Codex...

[ Claude Code | Codex ]              [ ⬤ Toggle ]

  --- Toggle OFF ---
  Notice: "Toggle MCP access on to reveal your setup prompt."

  --- Toggle ON ---
  Setup prompt
  Paste this into <client> to set up the n8n MCP server.
  When the agent finishes, restart <client>.

  <prompt code block>

                                              [ Copy ]
```

The `Claude Code | Codex` radio remains. Switching the radio swaps the
prompt body in place. The toggle remains visible in both states.

## Prompt content

### Claude Code prompt

```
Set up the n8n MCP server for Claude Code globally
(available in all projects).

1. Add the environment variable to your shell config
   (e.g. ~/.zshrc or ~/.bashrc), then reload your shell:

   export N8N_MCP_TOKEN="<token>"

2. Register the MCP server at user scope by running:

   claude mcp add --scope user --transport http n8n \
     "<server-url>" \
     --header 'Authorization: Bearer ${N8N_MCP_TOKEN}'

3. Preserve `${N8N_MCP_TOKEN}` literally in the stored header value.
   Do not expand it before running the command.

4. When you finish, tell me to restart Claude Code. Do not try to
   verify the server in this session.
```

### Codex prompt

```
Set up the n8n MCP server for Codex globally.

1. Add the environment variable to your shell config
   (e.g. ~/.zshrc or ~/.bashrc), then reload your shell:

   export N8N_MCP_TOKEN="<token>"

2. Add the following to ~/.codex/config.toml (create the
   file and directory if they do not exist; if the file
   already exists, append the section without removing
   existing config):

   [mcp_servers.n8n]
   url = "<server-url>"
   bearer_token_env_var = "N8N_MCP_TOKEN"

3. When you finish, tell me to restart Codex. Do not try to verify the
   server in this session.
```

The `<server-url>` and `<token>` placeholders are replaced inline before
copy. While the token is redacted (see Open issues), the prompt shows
`<your-access-token>` instead of a real value and the copy button is
disabled.

## Component boundaries

### `MCPOnboardingModal.vue`

Responsibility:

- modal frame and lifecycle,
- toggle wiring,
- client selection,
- access-key fetching (`getOrCreateApiKey()` on mount and after a
  successful toggle-on),
- delegation to the prompt component.

Changes:

- Remove the `MCPAccessTokenPopoverTab` import and rendering.
- Remove the `N8nButton`. Replace with `McpAccessToggle` bound to
  `mcpStore.mcpAccessEnabled`, `isToggling` mapped to `loading`, and
  `mcpStore.mcpManagedByEnv` mapped to both `disabled` and
  `managedByEnv`.
- Replace the existing `enableMcpAccess` handler with a unified
  toggle handler that derives `nextValue` from the current
  `mcpStore.mcpAccessEnabled` value and calls
  `mcpStore.setMcpAccessEnabled(nextValue)`.
  - On toggle-on success, fire `trackEnableClicked` (before the call)
    and `trackEnabled` (after success), preserving today's semantics.
  - On toggle-off, no telemetry, no error toast for an in-progress
    toggle (use the same error handling pattern as today for failures).
- Rename the local in-flight ref from `isEnabling` to `isToggling`
  since requests now go in both directions.
- Move the `getOrCreateApiKey()` call up from the removed
  `MCPAccessTokenPopoverTab` into the modal itself. The modal must
  call it:
  - on mount when `mcpAccessEnabled` is already `true`,
  - on a successful toggle-on transition, before the prompt panel
    renders.
- Keep the local computeds `accessToken`, `isKeyRedacted`, and
  `hasResolvedAccessToken`. The modal continues to own token fetch and
  readiness state, and passes the resolved values into the prompt
  component.
- Keep `enabledDuringThisOpen`, `surface`, and the
  `handleModalClosed` dismissal-tracking behavior unchanged for the
  first-open-modal variant.
- Forward copy events from the prompt component to
  `experimentStore.trackCopiedParameter(surface, client, 'agent-prompt')`.

### `MCPOnboardingClientSetup.vue`

Responsibility (new):

- compose the per-client setup prompt,
- render the prompt as a code block,
- handle the copy action.

Changes:

- Drop the env / config snippet computeds.
- Add `claudeCodePrompt` / `codexPrompt` builders that interpolate
  `serverUrl` and either the resolved token or
  `<your-access-token>` placeholder when `isTokenReady` is false.
- Render a single panel (title + description + code block + copy
  button), not two cards.
- Keep the `client`, `serverUrl`, `accessToken`, `isTokenReady`
  props.
- Keep token fetch, redaction detection, and readiness ownership in the
  modal. This component only formats the final prompt text.
- Change the `copy` event payload from
  `[parameter: 'setup-config']` to `[parameter: 'agent-prompt']`.

### `McpAccessToggle.vue`

No changes to the component itself. It is reused as-is, and the modal
adapts to its existing emit contract (`disableMcpAccess` with no
boolean payload).

### MCP store

No changes. The modal continues to call `setMcpAccessEnabled`,
`getOrCreateApiKey`, and `resetCurrentUserMCPKey`.

### Experiment store

Update the `SurfaceMcpOnboardingParameter` type alias:

- before: `'server-url' | 'access-token' | 'setup-config'`
- after: `'agent-prompt'`

`trackCopiedParameter` continues to accept the same arguments. Only the
`parameter` union narrows. Existing call sites in the modal go from
emitting three values to one.

## Data flow

### Toggle on

1. User toggles MCP on.
2. Modal sets `isToggling = true`, which puts the toggle into its
   loading state for the duration of the request.
3. Modal calls `experimentStore.trackEnableClicked(surface)`.
4. Modal calls `mcpStore.setMcpAccessEnabled(true)`.
5. On success:
   - `enabledDuringThisOpen = true`,
   - `experimentStore.trackEnabled(surface)`.
6. Modal then calls `mcpStore.getOrCreateApiKey()` to populate
   `currentUserMCPKey`. This responsibility moves from the removed
   `MCPAccessTokenPopoverTab.onMounted` into the modal.
7. The prompt panel renders the prompt with the resolved token (or the
   placeholder if redacted).

### Toggle off

1. User toggles MCP off.
2. Modal sets `isToggling = true`, which puts the toggle into its
   loading state for the duration of the request.
3. Modal calls `mcpStore.setMcpAccessEnabled(false)`.
4. On success, the prompt panel hides and the pending notice is shown
   again.
5. No telemetry is emitted.

### Modal opens with MCP already enabled

1. The toggle renders in the on position.
2. Modal calls `getOrCreateApiKey()` on mount to populate the key.
3. The prompt panel renders immediately.

### Copy

1. User clicks Copy in the prompt panel.
2. Prompt component copies the prompt body to the clipboard.
3. Prompt component emits `copy: 'agent-prompt'`.
4. Modal forwards to
   `experimentStore.trackCopiedParameter(surface, client, 'agent-prompt')`.

### Client switch

Same as today: radio change triggers
`experimentStore.trackClientSelected(client)`. The prompt component
recomputes the prompt body on the new `client` prop.

## Error handling

### Enable / disable request fails

- Show an inline toast via `useToast`, same pattern as today.
- Keep the modal open, do not flip the toggle state.
- Allow retry.

### Token fetch fails

- Show an inline toast via `useToast`.
- The prompt panel renders with the `<your-access-token>` placeholder
  and a disabled copy button so the user does not paste a broken
  prompt.
- Allow retry by closing and reopening the modal, or by toggling
  off / on.

### Duplicate clicks on the toggle

- The toggle is locked to its current state while a request is in
  flight (`loading` prop on `McpAccessToggle`).

### Modal reopened after enable

- Same as data-flow case "Modal opens with MCP already enabled".

## i18n changes

### Drop

- `settings.mcp.onboarding.step.env`
- `settings.mcp.onboarding.step.config`
- `settings.mcp.onboarding.claudeCode.path`
- `settings.mcp.onboarding.codex.path`

### Modify

- `settings.mcp.onboarding.pending`: change to a toggle-based prompt,
  e.g. "Toggle MCP access on to reveal your setup prompt."
- `settings.mcp.onboarding.enable`: no longer used as a button label.
  Remove the key. The toggle uses
  `settings.mcp.header.toggle.enabled` /
  `settings.mcp.header.toggle.disabled` for its on/off label.

### Add

- `settings.mcp.onboarding.prompt.title`: e.g. "Setup prompt".
- `settings.mcp.onboarding.prompt.description`: e.g. "Paste this into
  {client} to set up the n8n MCP server. When the agent finishes,
  restart {client}."
- `settings.mcp.onboarding.prompt.claudeCode`: full Claude Code prompt
  body with `{serverUrl}` and `{token}` placeholders.
- `settings.mcp.onboarding.prompt.codex`: full Codex prompt body with
  `{serverUrl}` and `{token}` placeholders.

The reusable `generic.copy` key is reused for the copy button label.

## Telemetry

- `trackCopiedParameter` payload: `parameter` becomes `'agent-prompt'`
  for the only remaining copyable item. The PostHog readout for this
  experiment must be told about the rename.
- `trackEnableClicked` and `trackEnabled` continue to fire only on
  toggle-on transitions.
- `trackDismissed` continues to fire only when the first-open-modal is
  closed without enabling, identical to today.
- `trackClientSelected` is unchanged.
- No new event for toggle-off.

## Testing

### `MCPOnboardingModal.test.ts`

Update existing tests:

- Replace `getByRole('button', { name: 'Enable MCP access' })` with the
  toggle (`data-test-id="mcp-access-toggle"`).
- Update assertion for "loads connection details immediately when MCP
  is already enabled" → "renders the prompt immediately when MCP is
  already enabled". Verify the prompt code block contains the resolved
  token and server URL.
- Drop assertions on the connection details panel.

Add new tests:

- Toggling off calls `setMcpAccessEnabled(false)` and hides the prompt
  panel.
- Toggle is disabled when `mcpManagedByEnv` is true.
- Copy emits `trackCopiedParameter` with `'agent-prompt'`.

### `MCPOnboardingClientSetup` tests

Add or update component-level tests:

- Renders Claude Code prompt when `client === 'claude_code'`. Verify
  the prompt contains the `claude mcp add --scope user` command, the
  resolved token / URL in the export step, and the literal
  `${N8N_MCP_TOKEN}` placeholder in the header.
- Renders Codex prompt when `client === 'codex'`. Verify it contains
  the `[mcp_servers.n8n]` section and `~/.codex/config.toml` path.
- When `isTokenReady` is false, the prompt contains the
  `<your-access-token>` placeholder and the copy button is disabled.
- Copy click calls clipboard with the prompt and emits
  `copy: 'agent-prompt'`.

### Experiment store tests

Update `surfaceMcpToNewCloudUsers.store.test.ts` to reflect the
narrower `parameter` union: tests for `'server-url'` and
`'access-token'` are removed; `'agent-prompt'` is asserted instead of
`'setup-config'`.

### Out of scope for this iteration

- Full MCP settings page regression coverage.
- Non-admin permission branching.
- Network error retry UX beyond toast-based feedback.

## Open issues

### Token redaction on re-enable

`mcpStore.getOrCreateApiKey()` returns a redacted value (`****`) when
the user already has an API key. With the new bidirectional toggle, a
user who toggles off, then on, will see the prompt rendered with
`<your-access-token>` placeholder text and a disabled copy button. The
agent cannot configure access from such a prompt without rotation.

This is pre-existing behavior in the current modal: the
`accessToken` computed already substitutes `<your-access-token>` when
the key is redacted, and there is no rotate affordance in this modal
today.

This iteration does not solve it. If we want to solve it later, the
options are:

- add a small rotate-token affordance to the prompt panel that calls
  `mcpStore.generateNewApiKey()`,
- detect the redacted state and show a banner pointing the user to
  `Settings > MCP` to rotate,
- always rotate when MCP is enabled from this modal.

### Claude Code CLI availability

The Claude Code prompt depends on the `claude` CLI being on the
user's `PATH`. If the user runs Claude Code through an embedded IDE
mode without the CLI, the prompt's step 2 fails. The agent will report
the failure back to the user. Treating this as acceptable for the PoC.

### Telemetry rename

Renaming the `parameter` value from `setup-config` to `agent-prompt`
breaks continuity with any existing dashboards consuming the previous
value. Coordinate with the experiment owner before shipping.

## Implementation notes

- Keep all changes within
  `packages/frontend/editor-ui/src/features/ai/mcpAccess/...` and the
  experiment store under
  `packages/frontend/editor-ui/src/experiments/surfaceMcpToNewCloudUsers/...`
  so the experiment is still trivial to remove.
- Do not introduce a new modal component; mutate the existing
  `MCPOnboardingClientSetup.vue` and `MCPOnboardingModal.vue`.
- All user-facing strings continue to go through `@n8n/i18n`.
- Verify during implementation that the single-quoted Claude Code
  command stores `${N8N_MCP_TOKEN}` literally in user-scope config. If
  Claude Code normalizes or expands it eagerly, stop and revise the
  design before shipping.
- Render the prompt with `N8nMarkdown` as a fenced code block, matching
  the rendering already used in `MCPOnboardingClientSetup.vue` today.
  This keeps styling, copy-button placement, and font tokens consistent
  with the existing setup cards. If `N8nMarkdown` produces unwanted
  whitespace or color overrides for the prompt's plain numbered text
  outside the fenced block, fall back to a styled `<pre>` for the
  surrounding text only — keep the code block on `N8nMarkdown`.
- After implementation, update the experiment design doc
  (`docs/superpowers/specs/2026-05-01-surface-mcp-to-new-cloud-users-design.md`)
  with a short note pointing at this revision so future readers see
  the divergence from the original "manual steps" plan.
