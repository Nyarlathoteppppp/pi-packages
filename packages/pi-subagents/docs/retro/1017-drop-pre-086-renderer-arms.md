---
issue: 1017
issue_title: "pi-subagents: drop the pre-0.86 prompt-renderer arms once the Pi floor is 1.0.0"
---

# Retro: #1017 — pi-subagents: drop the pre-0.86 prompt-renderer arms once the Pi floor is 1.0.0

## Stage: Planning (2026-10-25T02:24:55Z)

### Session summary

Planned the removal of the ≤0.85 footer arm and the ≤0.85 project-context lead-in offset from `src/session/prompts.ts`, the migration of `prompts.test.ts` off its footer-shaped `parentPrompt` fixture, and the Phase 23 ride-along (a shared `makeAgentConfig` test helper replacing four local factories and the prompt-test literals).
The operator chose to fold in a `fix:` for the surviving unanchored fallback arm, which leaves a dangling `<skills>` line on a section-shaped prompt with no `<cwd>` section.
The plan has 11 steps: test helper, factory migration, fixture rename, test moves, arm removal, fix, comment cleanup, describe regrouping, and two docs steps.

### Observations

- Spiking the src removal turned 15 of 81 `prompts.test.ts` tests red, all on the footer fixture; another 18 footer-fixture tests stayed green only because the unanchored arm happened to cut their catalogue.
- The unanchored arm is still reachable on Pi 1.0: a `before_agent_start` handler that returns `systemPrompt` becomes Pi's `forceSystemPrompt` (installed `dist/core/extensions/runner.js`).
  No `@gotgenes/*` extension does this; `pi-nocd` and `pi-permission-system` edit `systemPromptOptions`, and `pi-anthropic-auth` shapes the prompt at the provider boundary.
- A disposable probe measured that arm's defect on a section-shaped prompt: a dangling `<skills>` line, and a relocated child keeps the parent's `<project_context>`.
  The operator chose to fix it here rather than file a follow-up.
- `docs/configuration.md` lines 24 and 35 still describe the `Current working directory:` footer, which has been stale since 0.86, so the docs step widens past the issue's one sentence; `.pi/skills/package-pi-subagents/SKILL.md` lines 40–44 describe the footer anchor too.
- The Tidy-First assessor recommended landing the shared helper and the factory migrations first, and pre-renaming the fixtures (`parentPrompt` → `footerParentPrompt`, `sectionParentPrompt` → `parentPrompt`).
  It flagged that several moved tests turn vacuous unless they are reasserted on `<cwd>`/`<skills>` tokens.
  It also caught that "leaves a quoted catalogue alone" must move rather than be deleted.
- `invocation-config.test.ts`'s `makeOpinionatedConfig` matches the roadmap's factory grep, so the plan makes it a module-level `OPINIONATED` preset to bring the row to 0.
- ADRs 0006, 0010, and 0011 keep their footer-era wording, since decision records describe the shape they were decided against.

#### Deferred tidyings

- `test/helpers/subagent-session-io.ts`: `DEFAULT_AGENT_CONFIG` is a second `AgentConfig` baseline that could build on `makeAgentConfig`.
- `test/display.test.ts`, `test/tools/agent-tool.test.ts`, `test/tools/spawn-config.test.ts`, `test/helpers/make-deps.ts`: these still build bare `AgentConfig` literals.
- `src/session/prompts.ts`: `PromptShape` could become a boolean once only `"section" | "unanchored"` remain.
