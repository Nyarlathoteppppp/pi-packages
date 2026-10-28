---
issue: 1090
issue_title: "upgrade zod dependency - versions below 4.6.6 are affected by CVE-2023-54404"
---

# Retro: #1090 — upgrade zod dependency - versions below 4.6.6 are affected by CVE-2023-54404

## Stage: Planning (2026-10-28T03:28:52Z)

### Session summary

I planned the change for a third-party report asking for zod 4.6.6 or later.
The plan is cross-package: it raises `zod` to `^4.6.7` in `pi-permission-system` and `pi-permission-model-judge`, and bounds all six config arrays at 1,000 entries with `.max()`.
I also filed [#1092] for `pi-permission-model-judge` reading an untrusted project's config.

### Observations

- **Reading zod's source changed the direction.**
  The compiled zod 4.6.7 `$ZodArray` showed the upstream fix only pre-checks `.max()` / `.length()` / `.size()`.
  An unbounded array still records one issue per element, so the reporter's "may not be completely fixed" comment was right.
  On the operator's direction ("propose a bound for the array"), the plan bounds every array, not just the floor.
- **Exposure.**
  `pi-permission-system` parses only operator-owned configs (the project scope is trust-gated).
  `pi-permission-model-judge` parses an untrusted project's config, which is the one attacker-controlled zod input; that trust gap is [#1092].
- **Operator decisions.**
  - All six arrays, one shared cap of 1,000.
  - Breaking (`fix(<pkg>)!:`), chosen over non-breaking hardening.
  - Floor at the latest published version, `^4.6.7`, not `^4.6.6`.
  - File [#1092].
- **Measured with the published tarballs:**
  - 1,001 invalid entries gives 1,002 issues on 4.4.3 and 1 on 4.6.7 with `.max(1000)`.
  - 1,000,000 invalid entries on an unbounded array gives 1,000,000 issues in 672 ms.
  - The spike worktree passed every package's tests on 4.6.7, and both `gen:schema` outputs were unchanged.
- **The Tidy-First assessor caught a vacuous assertion.**
  At a union site (`declaredEffectsSchema`) the top-level issue count is 1 under every zod version.
  The plan's tests flatten `invalid_union` `errors` and assert exactly one `too_big` leaf and no element-indexed leaf (measured: 1,003 leaves on 4.4.3, 2 on 4.6.7).
- **Rejected options.**
  - `pnpm dedupe`: it churned 689 lockfile lines in the spike.
  - Capping `z.record` maps: zod 4.6.7 has no pre-walk cap for records (`capSize` covers only array, map, and set), and every record is read from trusted input only.
  - A shared `boundedArray` helper.
- Roadmap-fit: neither package has an open improvement phase, so [#1092] got no disposition.

## Stage: Implementation — TDD (2026-10-28T03:43:17Z)

### Session summary

I ran all four TDD steps, each as its own commit: two `fix(<pkg>)` zod 4.6.7 bumps (each crediting the reporter with a `Co-authored-by` trailer), then the two `fix(<pkg>)!:` array bounds.
Tests went from 6413 to 6428 in `pi-permission-system` and from 69 to 72 in `pi-permission-model-judge`.
The pre-completion reviewer returned PASS.

### Observations

- **Lockfile path.**
  Step 1's `pnpm install` left intermediate peer-variant entries, so Step 2's lockfile diff alone showed 467 changed lines.
  The net diff from the plan commit was exactly the spike's 110 lines of zod-only swaps, with zero `zod@4.4.3` entries.
  Judge a multi-step lockfile change by its net diff, not the per-commit stat.
- **Union-path deviation.**
  zod reports an `invalid_union` arm's issues with paths relative to the union, not absolute (`[]`, not `["t", "effects"]`).
  The `leafIssues` test helper now prefixes each union's own path while it recurses.
  The plan's assertion design held; only the helper's path handling changed.
- **Mutations:** every one killed exactly the predicted tests, plus the schema-parity pin.
  - Dropping `authorizerChain`'s `.max` killed 1 row; dropping `piInfrastructureReadPaths`' killed its row and the loader test.
  - Dropping `declaredEffectsSchema`'s `.max` killed all 3 effects rows.
  - Cap 999 killed all 7 acceptance rows.
  - Pinning zod 4.4.3 killed all 7 rejection rows and the loader test, while the acceptance rows stayed green.
  - In model-judge, both mutations killed the predicted tests.
- **Doc deviation:** the plan predicted `architecture.md` unchanged, but its `config-schema.ts` module-tree entry got a constraint sentence on the array bound; the reviewer judged it sound.
- I first inserted the `pi-permission-system` `commandEffects` cap sentence between two table rows; I moved it below the table before committing.
- Pre-completion reviewer: PASS.
  It re-derived six `z.array` sites, all bounded, and found one zod resolution (4.6.7) in the lockfile.

[#1092]: https://github.com/gotgenes/pi-packages/issues/1092
