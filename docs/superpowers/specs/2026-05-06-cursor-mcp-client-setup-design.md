# Cursor MCP Client Setup Design

## Summary

Add Cursor as a third client option in the existing MCP onboarding setup modal.
Cursor follows the same user-global setup model as Claude Code and Codex, so the
n8n MCP server is available across Cursor projects after setup.

This spec extends
`docs/superpowers/specs/2026-05-05-mcp-onboarding-toggle-and-agent-prompt-design.md`.
Where the older spec says the PoC supports only Claude Code and Codex, this
addendum supersedes that scope for Cursor only.

## Goals

- Add Cursor to the onboarding client picker.
- Generate a Cursor-specific setup prompt that uses `~/.cursor/mcp.json`.
- Keep the current single setup-prompt interaction and copy telemetry model.
- Keep the implementation additive and small.

## Non-goals

- Add other MCP clients in this iteration.
- Add a project-local Cursor setup option.
- Redesign the setup modal layout.
- Change MCP backend APIs or token generation.

## Product Decisions

### Client picker

The existing client picker becomes:

```text
Claude Code | Cursor | Codex
```

Selecting Cursor swaps the prompt body in place, just like the existing Claude
Code and Codex options.

### Cursor setup scope

Cursor setup targets the user-global Cursor MCP config at:

```text
~/.cursor/mcp.json
```

Project-local `.cursor/mcp.json` setup is not included in this iteration.

### Cursor prompt content

The Cursor prompt tells the agent to:

1. Add `N8N_MCP_TOKEN` to the user's shell config, then reload the shell.
2. Create `~/.cursor/mcp.json` if it does not exist.
3. Merge an `n8n` MCP server entry without removing existing Cursor config.
4. Tell the user to restart Cursor when finished.

The server entry uses Cursor's remote MCP server format and environment-variable
interpolation:

```json
{
	"mcpServers": {
		"n8n": {
			"url": "{serverUrl}",
			"headers": {
				"Authorization": "Bearer ${env:N8N_MCP_TOKEN}"
			}
		}
	}
}
```

The prompt must preserve `${env:N8N_MCP_TOKEN}` literally in the JSON config.
The token itself is stored in the shell environment variable, not directly in
Cursor's MCP config.

## Technical Design

- Extend the onboarding client type from `'claude_code' | 'codex'` to include
  `'cursor'` in the modal, setup component, and experiment telemetry store.
- Add `settings.mcp.onboarding.client.cursor` to the i18n file.
- Add `settings.mcp.onboarding.prompt.cursor` to the i18n file.
- Replace two-client conditionals in `MCPOnboardingClientSetup.vue` with a small
  client-to-key mapping so adding Cursor does not create nested conditionals.
- Add Cursor to `clientOptions` in `MCPOnboardingModal.vue`.
- Update tests to assert Cursor can be selected, renders the Cursor config
  snippet, and forwards copy telemetry with `cursor` as the selected client.
- Update user-facing references that say only "Claude Code or Codex" when they
  describe the client picker or setup flow.
- Escape the literal JSON braces in the i18n prompt and pass
  `${env:N8N_MCP_TOKEN}` as an interpolation value so Vue i18n still resolves
  `{token}` and `{serverUrl}`.

## Copy Notes

- Keep labels sentence case. Client names remain proper nouns: `Claude Code`,
  `Cursor`, and `Codex`.
- Keep prompt copy imperative and agent-directed, matching the existing Claude
  Code and Codex prompts.
- Avoid Latin abbreviations in new copy.

## Implementation TODO

- [x] Add Cursor to client types and telemetry types.
- [x] Add Cursor client label and setup prompt i18n keys.
- [x] Render Cursor in the setup modal client picker.
- [x] Generate and copy the Cursor setup prompt.
- [x] Update tests for Cursor selection and copy telemetry.
- [x] Verify the focused MCP onboarding tests pass.
