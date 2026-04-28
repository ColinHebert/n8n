# Setup Experiment Skill Design

## Summary

Create a shared skill for scaffolding new `packages/frontend/editor-ui` experiments.

The skill should:

- scaffold new experiments using a stricter standard than some legacy experiments follow today,
- be available natively to Codex through `.agents/skills/`,
- be available natively to Claude through `.claude/plugins/n8n/skills/`,
- keep one canonical workflow so the Codex and Claude entrypoints do not drift.

## Goals

- Standardize how new editor-ui experiments are created.
- Encode the current best experiment pattern instead of copying older inconsistencies.
- Make the workflow available to both Codex and Claude in this repository.
- Keep the implementation small, explicit, and easy to maintain.

## Non-goals

- Automatically wire experiments into app surfaces such as routes, layouts, sidebars, or modals.
- Refactor existing experiments to the new standard.
- Introduce a runtime experiment abstraction such as `createExperimentStore(...)`.
- Generate full feature implementations beyond the experiment shell.

## Current Repo Facts

- Experiment definitions live in `packages/frontend/editor-ui/src/app/constants/experiments.ts`.
- Experiment participation telemetry is centralized through `EXPERIMENTS_TO_TRACK` and `usePostHog()`.
- Experiments are wired manually into host surfaces.
- Most substantial experiments expose a Pinia store as their public API.
- The repository already has a Claude-native skill mechanism under `.claude/plugins/n8n/`.
- The repository does not yet have a shared Codex experiment setup skill under `.agents/skills/`.

## Target Standard For New Experiments

New experiments created by the skill should follow this contract.

### Experiment definition

Every experiment must:

1. Add a constant to `packages/frontend/editor-ui/src/app/constants/experiments.ts`.
2. Add the experiment name to `EXPERIMENTS_TO_TRACK`.
3. Use `createExperiment(...)` unless a legacy-compatible shape is explicitly required.

### Public API shape

Default to a Pinia store.

Use a composable instead of a store only when the experiment is genuinely small, stateless, and does not benefit from a stable shared store API.

For store-backed experiments:

1. Add a new `STORES.EXPERIMENT_*` entry in `packages/frontend/@n8n/stores/src/constants.ts`.
2. Create `packages/frontend/editor-ui/src/experiments/<name>/stores/<name>.store.ts`.
3. Treat that store as the public API for the experiment.

### Enrollment and local state

Separate enrollment from local UI state.

- `isEnabled`
  - PostHog enrollment only.
  - For simple experiments, derive from `posthogStore.isVariantEnabled(...)`.
  - For multi-variant experiments, derive from `currentVariant`.
- `currentVariant`
  - Only expose this for multi-variant experiments.
- `shouldShow*`, `can*`, `hasDismissed*`, `hasInteracted*`
  - Local UI state, display conditions, or workflow conditions.

New experiments should not mix dismissal state, plan checks, or product checks into `isEnabled`.

### Gating

The standard for new experiments is:

- gate enrollment through PostHog only,
- prefer person properties and group properties inside PostHog targeting,
- avoid duplicating product gating in the frontend unless it is truly required for correctness.

This means the skill should not scaffold additional frontend gating such as:

- `userIsTrialing`,
- cloud-only checks,
- selected-app checks,
- workflow-count checks,

unless the user explicitly requests those conditions as local display logic rather than enrollment logic.

### Persistence

Persist experiment-owned UI state with `packages/frontend/editor-ui/src/app/composables/useStorage.ts`.

Use `useStorage(...)` for:

- dismissed state,
- interaction state,
- last-seen state,
- other experiment-local persisted UI state.

Do not use raw `localStorage` for new experiment-owned state by default.

Raw `localStorage` is acceptable only when touching an existing non-experiment app key that already lives outside the experiment boundary.

### Telemetry

Keep participation tracking centralized.

The skill must not scaffold `"User is part of experiment"` tracking inside the experiment store or composable.

The experiment may scaffold only experiment-specific events such as:

- `trackViewed`,
- `trackDismissed`,
- `trackClicked`,
- `trackCompleted`.

### Host-surface wiring

The skill does not wire experiments into the app automatically.

The generated output must explicitly tell the user that follow-up wiring is manual and may include:

- importing the store or composable into a host surface,
- adding route guards,
- registering modal entrypoints,
- adding i18n keys,
- adding workflow payload files,
- adding components or data modules.

## Packaging Strategy

Use a dual-entrypoint setup with one canonical workflow.

### Canonical workflow

Create the Codex-native skill as the canonical source:

```text
.agents/skills/setup-experiment/
├── SKILL.md
└── reference.md
```

- `SKILL.md` contains the Codex-native metadata and the short activation instructions.
- `reference.md` contains the full workflow and scaffolding rules.

### Claude wrapper

Create a thin Claude wrapper:

```text
.claude/plugins/n8n/skills/setup-experiment/
└── SKILL.md
```

This wrapper should:

- stay short,
- point to `.agents/skills/setup-experiment/reference.md`,
- avoid duplicating the workflow body.

### Why this packaging

- Codex gets a native repo skill under `.agents/skills/`.
- Claude gets native plugin discovery under `.claude/plugins/n8n/skills/`.
- The workflow itself is maintained in one place.

## Skill File Requirements

### Codex skill

`.agents/skills/setup-experiment/SKILL.md` should include:

- `name: setup-experiment`
- `description: ...`
- short trigger guidance,
- a short default workflow,
- a pointer to `reference.md` for the full rules.

### Claude wrapper skill

`.claude/plugins/n8n/skills/setup-experiment/SKILL.md` should:

- follow the repository plugin convention,
- omit the `name` field from frontmatter,
- include a discovery-friendly `description`,
- instruct Claude to follow `.agents/skills/setup-experiment/reference.md`.

### Shared reference

`.agents/skills/setup-experiment/reference.md` is the single source of truth for:

- triggers,
- defaults,
- branching rules,
- file layout,
- store contract,
- persistence rules,
- test expectations,
- follow-up reminders.

## Skill Workflow

When activated, the skill should do the following.

1. Ask for or infer the experiment name.
2. Ask whether the experiment is:
   - simple `control/variant`, or
   - multi-variant.
3. Default to a store-based scaffold.
4. Add the experiment constant in `experiments.ts`.
5. Add the experiment name to `EXPERIMENTS_TO_TRACK`.
6. Add a `STORES.EXPERIMENT_*` entry for store-backed experiments.
7. Scaffold the experiment directory under `packages/frontend/editor-ui/src/experiments/<name>/`.
8. Create the public store or composable.
9. Create the public API test.
10. Remind the user that host-surface wiring is not included.

## Generated Output Scope

The skill should scaffold the smallest useful shell.

### Default generated files

For a store-backed experiment:

```text
packages/frontend/editor-ui/src/experiments/<name>/
└── stores/
    ├── <name>.store.ts
    └── <name>.store.test.ts
```

It should also update:

```text
packages/frontend/editor-ui/src/app/constants/experiments.ts
packages/frontend/@n8n/stores/src/constants.ts
```

### Optional generated files

Only generate these when the user asks for them:

- `components/`
- `data/`
- `workflows/`
- route or host-surface integration
- i18n entries

### Store template expectations

The default store template should include:

- `const posthogStore = usePostHog();`
- `const telemetry = useTelemetry();` only if the experiment needs events immediately
- `const isEnabled = computed(() => posthogStore.isVariantEnabled(...));`
- `const currentVariant = computed(() => posthogStore.getVariant(...));` only for multi-variant experiments
- `useStorage(...)` only when persisted local state is needed
- `track*` methods only for experiment-specific events

## Testing Standard

Aim for decent coverage, not full coverage.

The skill should default to:

- one unit test for the public store or composable,
- assertions for enablement behavior,
- assertions for persisted local state when present,
- assertions for any experiment-specific telemetry helpers when present.

The skill should not default to:

- full component coverage,
- end-to-end tests,
- tests for static workflow payload files,

unless the user asks for them or the scaffold includes behavior that requires them.

## Documentation And Follow-up

The skill output should end with a short follow-up checklist for the user or agent.

Example follow-up items:

- wire the experiment into the target surface,
- add i18n keys if UI text was introduced,
- add route guards or modal registration if needed,
- add components or workflow payload files if the experiment goes beyond the base scaffold.

## Alternatives Considered

### Separate full skills for Codex and Claude

Rejected because the workflow would drift over time.

### Canonical workflow in the Claude plugin with Codex using AGENTS pointers

Rejected because Codex has a native repo skill location and should use it directly.

### Runtime abstraction for experiment creation

Rejected because this is a scaffolding problem, not a runtime architecture problem.

## Decision

Build the experiment setup skill with:

- a Codex-native canonical skill in `.agents/skills/setup-experiment/`,
- a thin Claude wrapper in `.claude/plugins/n8n/skills/setup-experiment/`,
- one shared workflow in `.agents/skills/setup-experiment/reference.md`,
- a stricter experiment scaffold that standardizes:
  - `isEnabled`,
  - `STORES.EXPERIMENT_*`,
  - `useStorage(...)`,
  - PostHog-only enrollment,
  - centralized participation tracking.

## Implementation Notes

When implementation starts, use the repository’s existing skill-authoring conventions for Claude plugin skills and keep the wrapper files short.

No `AGENTS.md` change is required for Codex skill activation because Codex discovers repo skills from `.agents/skills/`.

Do not create placeholder directories or boilerplate files that are not immediately useful.
