# Setup Experiment Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a shared experiment setup skill with a Codex-native canonical workflow in `.agents/skills/setup-experiment/` and a thin Claude wrapper in `.claude/plugins/n8n/skills/setup-experiment/`.

**Architecture:** Keep the real workflow in `.agents/skills/setup-experiment/reference.md`, with `.agents/skills/setup-experiment/SKILL.md` as the Codex entrypoint and `.claude/plugins/n8n/skills/setup-experiment/SKILL.md` as the Claude wrapper. Avoid AGENTS or runtime code changes; this is a packaging and documentation task that should keep one source of truth for the experiment scaffolding rules.

**Tech Stack:** Markdown skill files, Codex repo skills, Claude plugin skills, existing n8n skill conventions.

---

## File Structure

- Create: `.agents/skills/setup-experiment/SKILL.md`
  Codex-native skill entrypoint with concise triggers and defaults.
- Create: `.agents/skills/setup-experiment/reference.md`
  Canonical workflow for scaffolding new editor-ui experiments.
- Create: `.claude/plugins/n8n/skills/setup-experiment/SKILL.md`
  Claude plugin wrapper that points to the shared reference.

No other files should change in the first implementation pass.

### Task 1: Create the Codex-native canonical skill

**Files:**
- Create: `.agents/skills/setup-experiment/SKILL.md`
- Create: `.agents/skills/setup-experiment/reference.md`
- Verify: `.agents/skills/setup-experiment/SKILL.md`
- Verify: `.agents/skills/setup-experiment/reference.md`

- [ ] **Step 1: Verify the Codex skill path does not exist yet**

Run:

```bash
find .agents/skills/setup-experiment -maxdepth 2 -type f | sort
```

Expected: the command exits non-zero with `find: .agents/skills/setup-experiment: No such file or directory`.

- [ ] **Step 2: Create the Codex `SKILL.md` entrypoint**

Write `.agents/skills/setup-experiment/SKILL.md` with this content:

```markdown
---
name: setup-experiment
description: Scaffolds new `packages/frontend/editor-ui` experiments using the n8n experiment standard. Use when creating a new editor-ui experiment, adding a PostHog experiment scaffold, or setting up experiment files for a new experiment.
---

# Setup experiment

Use this skill when creating a new `packages/frontend/editor-ui` experiment.

## Defaults

- Start in `packages/frontend/editor-ui/src/app/constants/experiments.ts`.
- Add the experiment name to `EXPERIMENTS_TO_TRACK`.
- Default to a store-backed scaffold with `STORES.EXPERIMENT_*`.
- Use `posthogStore.isVariantEnabled(EXPERIMENT.name, EXPERIMENT.variant)` for simple `control/variant` experiments.
- Use `currentVariant` for multi-variant experiments.
- Use `useStorage(KEY)` for experiment-owned persisted UI state.
- Keep participation tracking centralized.
- Leave host-surface wiring manual unless the user explicitly asks for it.

## More detail

Follow `reference.md` for the full workflow, defaults, and scaffold rules.
```

- [ ] **Step 3: Create the shared `reference.md` workflow**

Write `.agents/skills/setup-experiment/reference.md` with this content:

```markdown
# Setup experiment reference

Use this workflow when scaffolding a new experiment under `packages/frontend/editor-ui/src/experiments/`.

## Ask or infer

1. Experiment folder name, for example `setupPanelV2`.
2. Experiment constant name, for example `SETUP_PANEL_V2_EXPERIMENT`.
3. Variant shape:
   - simple `control/variant`, or
   - multi-variant.
4. Public API shape:
   - default to a Pinia store,
   - only use a composable when the experiment is genuinely tiny and stateless.
5. Whether persisted UI state is needed.
6. Whether optional `components/`, `data/`, `workflows/`, or host-surface wiring are in scope.

## Always do

1. Add the experiment constant to `packages/frontend/editor-ui/src/app/constants/experiments.ts`.
2. Add the experiment name to `EXPERIMENTS_TO_TRACK`.
3. Add a `STORES.EXPERIMENT_*` entry in `packages/frontend/@n8n/stores/src/constants.ts` for store-backed experiments.
4. Create `packages/frontend/editor-ui/src/experiments/<name>/`.
5. Default to `stores/<name>.store.ts` and `<name>.store.test.ts`.

## Enrollment standard

- Use `isEnabled` for PostHog enrollment only.
- For simple experiments, use `posthogStore.isVariantEnabled(EXPERIMENT.name, EXPERIMENT.variant)`.
- For multi-variant experiments, expose `currentVariant` and derive booleans from it.
- Keep display state separate with `shouldShow*`, `can*`, `hasDismissed*`, or `hasInteracted*`.

Do not mix enrollment with:

- `userIsTrialing`,
- cloud-only checks,
- selected-app checks,
- workflow-count checks,
- dismissal state.

If the user wants those conditions, treat them as local display logic, not enrollment logic.

## Persistence standard

Use `packages/frontend/editor-ui/src/app/composables/useStorage.ts` for experiment-owned persisted UI state.

Use it for:

- dismissed state,
- interaction state,
- last-seen state.

Do not use raw `localStorage` for new experiment-owned state unless the experiment must touch an existing non-experiment app key.

## Telemetry standard

- Keep `"User is part of experiment"` tracking centralized.
- Scaffold only experiment-specific `track*` helpers such as `trackViewed`, `trackClicked`, `trackDismissed`, or `trackCompleted`.

## Store scaffold

Default to this store shape and adapt names as needed:

```ts
import { useTelemetry } from '@/app/composables/useTelemetry';
import { useStorage } from '@/app/composables/useStorage';
import { SETUP_PANEL_V2_EXPERIMENT } from '@/app/constants';
import { usePostHog } from '@/app/stores/posthog.store';
import { STORES } from '@n8n/stores';
import { defineStore } from 'pinia';
import { computed } from 'vue';

const DISMISSED_KEY = 'N8N_SETUP_PANEL_V2_DISMISSED';

export const useSetupPanelV2Store = defineStore(STORES.EXPERIMENT_SETUP_PANEL_V2, () => {
	const posthogStore = usePostHog();
	const telemetry = useTelemetry();
	const dismissed = useStorage(DISMISSED_KEY);

	const isEnabled = computed(() =>
		posthogStore.isVariantEnabled(
			SETUP_PANEL_V2_EXPERIMENT.name,
			SETUP_PANEL_V2_EXPERIMENT.variant,
		),
	);
	const isDismissed = computed(() => dismissed.value === 'true');
	const shouldShowCallout = computed(() => isEnabled.value && !isDismissed.value);

	function dismiss() {
		dismissed.value = 'true';
	}

	function trackViewed() {
		telemetry.track('Setup panel v2 viewed');
	}

	return {
		isEnabled,
		isDismissed,
		shouldShowCallout,
		dismiss,
		trackViewed,
	};
});
```

## Test scaffold

Default to one public API test next to the store. Mirror the local style from:

- `packages/frontend/editor-ui/src/experiments/credentialsAppSelection/stores/credentialsAppSelection.store.test.ts`
- `packages/frontend/editor-ui/src/experiments/resourceCenter/stores/__tests__/resourceCenter.store.test.ts`

The default assertions should cover:

1. enrollment logic,
2. persisted state when present,
3. experiment-specific telemetry helpers when present.

## Do not do by default

- Do not wire routes, layouts, sidebars, or modals automatically.
- Do not add empty `components/`, `data/`, or `workflows/` folders unless requested.
- Do not add product gating on top of PostHog enrollment unless the user explicitly wants local display logic.
- Do not scaffold participation tracking inside the experiment.

## Output reminder

At the end of the scaffold, explicitly tell the user what is still manual, for example:

- host-surface wiring,
- i18n keys,
- route guards,
- modal registration,
- extra workflow payload files.
```

- [ ] **Step 4: Verify the Codex skill files and core rules**

Run:

```bash
find .agents/skills/setup-experiment -maxdepth 2 -type f | sort
rg -n "EXPERIMENTS_TO_TRACK|isVariantEnabled|useStorage|STORES\\.EXPERIMENT_" .agents/skills/setup-experiment/reference.md
sed -n '1,120p' .agents/skills/setup-experiment/SKILL.md
```

Expected:

- `find` lists exactly:
  - `.agents/skills/setup-experiment/SKILL.md`
  - `.agents/skills/setup-experiment/reference.md`
- `rg` prints matches for all four required patterns.
- `sed` shows the `name: setup-experiment` frontmatter and the `Defaults` section.

- [ ] **Step 5: Commit the Codex skill files**

Run:

```bash
git add .agents/skills/setup-experiment/SKILL.md .agents/skills/setup-experiment/reference.md
git commit -m "docs: add Codex setup experiment skill" -m "Add the Codex-native setup-experiment skill entrypoint and shared workflow reference." -m "Co-authored-by: Codex <noreply@openai.com>"
```

Expected: a commit is created with the two `.agents/skills/setup-experiment/` files staged and no unrelated files included.

### Task 2: Add the Claude wrapper that points to the shared reference

**Files:**
- Create: `.claude/plugins/n8n/skills/setup-experiment/SKILL.md`
- Verify: `.claude/plugins/n8n/skills/setup-experiment/SKILL.md`
- Verify: `.agents/skills/setup-experiment/reference.md`

- [ ] **Step 1: Verify the Claude wrapper path does not exist yet**

Run:

```bash
find .claude/plugins/n8n/skills/setup-experiment -maxdepth 2 -type f | sort
```

Expected: the command exits non-zero with `find: .claude/plugins/n8n/skills/setup-experiment: No such file or directory`.

- [ ] **Step 2: Create the Claude wrapper `SKILL.md`**

Write `.claude/plugins/n8n/skills/setup-experiment/SKILL.md` with this content:

```markdown
---
description: Scaffolds new `packages/frontend/editor-ui` experiments using the n8n experiment standard. Use when creating a new editor-ui experiment, adding a PostHog experiment scaffold, or setting up experiment files for a new experiment.
---

# Setup experiment

Use this skill when creating a new `packages/frontend/editor-ui` experiment.

Follow `.agents/skills/setup-experiment/reference.md`.

## Defaults

- Start in `packages/frontend/editor-ui/src/app/constants/experiments.ts`.
- Add the experiment name to `EXPERIMENTS_TO_TRACK`.
- Default to a store-backed scaffold with `STORES.EXPERIMENT_*`.
- Use `posthogStore.isVariantEnabled(EXPERIMENT.name, EXPERIMENT.variant)` for simple experiments.
- Use `currentVariant` for multi-variant experiments.
- Use `useStorage(KEY)` for experiment-owned persisted UI state.
- Keep participation tracking centralized.
```

- [ ] **Step 3: Verify the wrapper follows Claude plugin conventions**

Run:

```bash
test -f .claude/plugins/n8n/skills/setup-experiment/SKILL.md
rg -n "^name:" .claude/plugins/n8n/skills/setup-experiment/SKILL.md
rg -n "\\.agents/skills/setup-experiment/reference.md" .claude/plugins/n8n/skills/setup-experiment/SKILL.md
sed -n '1,120p' .claude/plugins/n8n/skills/setup-experiment/SKILL.md
```

Expected:

- `test -f` exits zero.
- `rg -n "^name:" .claude/plugins/n8n/skills/setup-experiment/SKILL.md` exits non-zero with no matches.
- `rg -n "\\.agents/skills/setup-experiment/reference.md" .claude/plugins/n8n/skills/setup-experiment/SKILL.md` prints one match.
- `sed` shows the description-only frontmatter and the shared reference path.

- [ ] **Step 4: Run a cross-entrypoint consistency check**

Run:

```bash
find .agents/skills/setup-experiment .claude/plugins/n8n/skills/setup-experiment -maxdepth 2 -type f | sort
rg -n "setup experiment|EXPERIMENTS_TO_TRACK|isVariantEnabled|useStorage|reference.md" .agents/skills/setup-experiment .claude/plugins/n8n/skills/setup-experiment
git diff --stat HEAD~1..HEAD || true
git diff --stat
```

Expected:

- `find` lists exactly three files:
  - `.agents/skills/setup-experiment/SKILL.md`
  - `.agents/skills/setup-experiment/reference.md`
  - `.claude/plugins/n8n/skills/setup-experiment/SKILL.md`
- `rg` shows the shared wording across the two entrypoints and confirms the wrapper points at the shared reference.
- `git diff --stat HEAD~1..HEAD` shows the two `.agents/skills/setup-experiment/` files from Task 1.
- `git diff --stat` shows only `.claude/plugins/n8n/skills/setup-experiment/SKILL.md` as the remaining uncommitted change before the final commit.

- [ ] **Step 5: Commit the Claude wrapper**

Run:

```bash
git add .claude/plugins/n8n/skills/setup-experiment/SKILL.md
git commit -m "docs: add Claude setup experiment wrapper" -m "Add the Claude plugin wrapper for the shared setup-experiment workflow without duplicating the canonical reference." -m "Co-authored-by: Codex <noreply@openai.com>"
```

Expected: a second commit is created with only the Claude wrapper file staged.
